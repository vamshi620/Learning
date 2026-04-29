# Error Handling in Kafka .NET
## File 08: Retry, Dead Letter Queue & Resilience Patterns

---

## What You'll Learn

- Error types in Kafka (retriable vs fatal)
- Retry strategies (immediate, delayed, exponential backoff)
- Dead Letter Queue (DLQ) pattern
- Poison message detection and handling
- Idempotent consumers
- Circuit breaker pattern with Kafka

---

## 1. Error Types in Kafka

```
Consumer errors fall into two categories:

┌─────────────────────────────────────────────────────────────────┐
│  RETRIABLE ERRORS                    FATAL / SKIP ERRORS        │
│                                                                   │
│  - Network timeout                   - JSON parse failure        │
│  - Broker temporarily unavailable   - Schema mismatch           │
│  - Transient DB failure             - Business logic validation  │
│  - HTTP 503 from downstream API     - Data contract violation   │
│                                                                   │
│  → Retry with backoff               → Skip or send to DLQ       │
└─────────────────────────────────────────────────────────────────┘
```

### Kafka Library Error Categories

```csharp
try
{
    var result = consumer.Consume(token);
}
catch (ConsumeException ex)
{
    if (ex.Error.IsFatal)
    {
        // Fatal: cannot recover, must restart the consumer
        logger.LogCritical("Fatal Kafka error: {Reason}", ex.Error.Reason);
        throw;
    }

    if (ex.Error.IsLocalError)
    {
        // Local error: network, timeout, buffer overflow
        logger.LogWarning("Local error (retriable): {Code} - {Reason}",
            ex.Error.Code, ex.Error.Reason);
    }

    if (ex.Error.IsBrokerError)
    {
        // Broker error: topic doesn't exist, auth failure, etc.
        logger.LogError("Broker error: {Code} - {Reason}",
            ex.Error.Code, ex.Error.Reason);
    }
}
```

---

## 2. Retry Strategies

### 2.1 Simple Retry (In-Memory)

```csharp
public class RetryConsumerService : BackgroundService
{
    private readonly int _maxRetries = 3;
    private readonly TimeSpan _retryDelay = TimeSpan.FromSeconds(2);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildConsumer();
        consumer.Subscribe("orders");

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(stoppingToken);
            if (result?.Message == null) continue;

            await ProcessWithRetryAsync(result, consumer, stoppingToken);
        }
    }

    private async Task ProcessWithRetryAsync(
        ConsumeResult<string, string> result,
        IConsumer<string, string> consumer,
        CancellationToken ct)
    {
        var attempt = 0;

        while (attempt <= _maxRetries)
        {
            try
            {
                await ProcessMessageAsync(result.Message.Value, ct);

                // Success — commit and move on
                consumer.Commit(result);
                _logger.LogInformation("Message processed successfully on attempt {Attempt}", attempt + 1);
                return;
            }
            catch (Exception ex) when (IsRetriable(ex) && attempt < _maxRetries)
            {
                attempt++;
                var delay = TimeSpan.FromMilliseconds(_retryDelay.TotalMilliseconds * Math.Pow(2, attempt - 1));

                _logger.LogWarning("Attempt {Attempt}/{Max} failed: {Error}. Retrying in {Delay}ms",
                    attempt, _maxRetries, ex.Message, delay.TotalMilliseconds);

                await Task.Delay(delay, ct); // Exponential backoff
            }
            catch (Exception ex)
            {
                // Exhausted retries OR non-retriable error → send to DLQ
                _logger.LogError(ex, "Message failed after {Max} attempts. Sending to DLQ", _maxRetries);
                await SendToDeadLetterQueueAsync(result, ex);
                consumer.Commit(result); // Skip this message
                return;
            }
        }
    }

    private bool IsRetriable(Exception ex) => ex switch
    {
        HttpRequestException => true,         // Network issues
        TimeoutException => true,             // Downstream timeout
        JsonException => false,               // Bad message — don't retry
        ArgumentException => false,           // Bad data — don't retry
        _ => true                             // Default: retry unknown errors
    };
}
```

### 2.2 Retry with Polly

```powershell
dotnet add package Microsoft.Extensions.Http.Resilience
dotnet add package Polly
```

```csharp
using Polly;
using Polly.Retry;

// Define a reusable retry policy
var retryPolicy = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        Delay = TimeSpan.FromSeconds(1),         // 1s, 2s, 4s
        MaxDelay = TimeSpan.FromSeconds(30),
        UseJitter = true,                         // Add randomness to prevent thundering herd
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutException>(),
        OnRetry = args =>
        {
            _logger.LogWarning("Retry {Attempt} after {Delay}ms: {Exception}",
                args.AttemptNumber, args.RetryDelay.TotalMilliseconds,
                args.Outcome.Exception?.Message);
            return default;
        }
    })
    .AddTimeout(TimeSpan.FromSeconds(10))         // Per-attempt timeout
    .Build();

// Use in consumer
await retryPolicy.ExecuteAsync(async ct =>
{
    await ProcessMessageAsync(message, ct);
}, stoppingToken);
```

---

## 3. Dead Letter Queue (DLQ) Pattern

Messages that repeatedly fail get moved to a DLQ topic for human inspection or later reprocessing.

```
Normal Flow:
[orders] → Consumer → ✅ Processed

Error Flow (with DLQ):
[orders] → Consumer → ❌ Failed × 3 → [orders.dlq] → Alert / Manual Review
```

### 3.1 DLQ Message Wrapper

```csharp
// Models/DeadLetterMessage.cs
public class DeadLetterMessage
{
    // Original message data
    public string OriginalTopic { get; set; } = string.Empty;
    public int OriginalPartition { get; set; }
    public long OriginalOffset { get; set; }
    public string OriginalKey { get; set; } = string.Empty;
    public string OriginalValue { get; set; } = string.Empty;
    public Dictionary<string, string> OriginalHeaders { get; set; } = new();

    // Error information
    public string ExceptionType { get; set; } = string.Empty;
    public string ExceptionMessage { get; set; } = string.Empty;
    public string StackTrace { get; set; } = string.Empty;
    public int RetryCount { get; set; }

    // Tracking
    public string ConsumerGroupId { get; set; } = string.Empty;
    public DateTime FailedAt { get; set; } = DateTime.UtcNow;
    public string FailedOnHost { get; set; } = Environment.MachineName;
}
```

### 3.2 DLQ Producer Service

```csharp
// Services/DeadLetterQueueService.cs
public class DeadLetterQueueService : IDeadLetterQueueService, IDisposable
{
    private readonly IProducer<string, string> _producer;
    private readonly ILogger<DeadLetterQueueService> _logger;

    public DeadLetterQueueService(
        ProducerConfig config,
        ILogger<DeadLetterQueueService> logger)
    {
        _logger = logger;
        _producer = new ProducerBuilder<string, string>(config).Build();
    }

    public async Task SendAsync<TKey, TValue>(
        ConsumeResult<TKey, TValue> originalMessage,
        Exception exception,
        int retryCount,
        string groupId)
    {
        // DLQ topic name convention: original topic + ".dlq"
        var dlqTopic = $"{originalMessage.Topic}.dlq";

        var dlqMessage = new DeadLetterMessage
        {
            OriginalTopic = originalMessage.Topic,
            OriginalPartition = originalMessage.Partition.Value,
            OriginalOffset = originalMessage.Offset.Value,
            OriginalKey = originalMessage.Message.Key?.ToString() ?? "null",
            OriginalValue = originalMessage.Message.Value?.ToString() ?? "null",
            OriginalHeaders = ExtractHeaders(originalMessage.Message.Headers),
            ExceptionType = exception.GetType().FullName ?? "Unknown",
            ExceptionMessage = exception.Message,
            StackTrace = exception.StackTrace ?? string.Empty,
            RetryCount = retryCount,
            ConsumerGroupId = groupId,
        };

        try
        {
            await _producer.ProduceAsync(dlqTopic, new Message<string, string>
            {
                Key = dlqMessage.OriginalKey,
                Value = JsonSerializer.Serialize(dlqMessage),
                Headers = new Headers
                {
                    { "dlq-reason", Encoding.UTF8.GetBytes(exception.Message[..Math.Min(exception.Message.Length, 200)]) },
                    { "dlq-source-topic", Encoding.UTF8.GetBytes(originalMessage.Topic) },
                    { "dlq-timestamp", Encoding.UTF8.GetBytes(DateTime.UtcNow.ToString("O")) },
                }
            });

            _logger.LogWarning(
                "Message from {Topic} P[{Partition}] O[{Offset}] sent to DLQ after {RetryCount} retries",
                originalMessage.Topic,
                originalMessage.Partition.Value,
                originalMessage.Offset.Value,
                retryCount);
        }
        catch (Exception ex)
        {
            // DLQ failure is critical — log and alert!
            _logger.LogCritical(ex, "CRITICAL: Failed to send message to DLQ {Topic}", dlqTopic);
            // In production: send alert to PagerDuty / email / Teams
        }
    }

    private static Dictionary<string, string> ExtractHeaders(Headers headers)
    {
        return headers?.ToDictionary(
            h => h.Key,
            h => Encoding.UTF8.GetString(h.GetValueBytes())
        ) ?? new();
    }

    public void Dispose()
    {
        _producer.Flush(TimeSpan.FromSeconds(5));
        _producer.Dispose();
    }
}
```

### 3.3 Consumer with Full Retry + DLQ

```csharp
public class ResilientOrderConsumer : BackgroundService
{
    private const int MaxRetries = 3;
    private const string Topic = "orders";
    private const string GroupId = "order-service";

    private readonly IDeadLetterQueueService _dlq;
    private readonly ILogger<ResilientOrderConsumer> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildConsumer();
        consumer.Subscribe(Topic);

        while (!stoppingToken.IsCancellationRequested)
        {
            ConsumeResult<string, string>? result = null;

            try
            {
                result = consumer.Consume(stoppingToken);
                if (result?.Message == null) continue;

                await ProcessWithFullResilienceAsync(result, consumer, stoppingToken);
            }
            catch (OperationCanceledException) { break; }
            catch (ConsumeException ex) when (!ex.Error.IsFatal)
            {
                _logger.LogWarning("Non-fatal consume error: {Reason}", ex.Error.Reason);
                await Task.Delay(1000, stoppingToken);
            }
        }

        consumer.Close();
    }

    private async Task ProcessWithFullResilienceAsync(
        ConsumeResult<string, string> result,
        IConsumer<string, string> consumer,
        CancellationToken ct)
    {
        var retryCount = 0;
        var baseDelay = TimeSpan.FromSeconds(1);

        while (true)
        {
            try
            {
                // Deserialize
                var order = JsonSerializer.Deserialize<OrderEvent>(result.Message.Value)
                    ?? throw new InvalidDataException("Null order after deserialization");

                // Validate
                if (string.IsNullOrEmpty(order.OrderId))
                    throw new ArgumentException("OrderId is required");

                // Process
                await DoBusinessLogicAsync(order, ct);

                // Success
                consumer.Commit(result);
                _logger.LogInformation("✅ Order {OrderId} processed", order.OrderId);
                return;
            }
            catch (JsonException ex)
            {
                // Poison message — never retry JSON errors
                _logger.LogError(ex, "Poison message detected — sending to DLQ immediately");
                await _dlq.SendAsync(result, ex, retryCount, GroupId);
                consumer.Commit(result);
                return;
            }
            catch (ArgumentException ex)
            {
                // Validation failure — don't retry
                _logger.LogError(ex, "Validation failed — sending to DLQ");
                await _dlq.SendAsync(result, ex, retryCount, GroupId);
                consumer.Commit(result);
                return;
            }
            catch (Exception ex) when (retryCount < MaxRetries)
            {
                retryCount++;
                var delay = TimeSpan.FromMilliseconds(
                    baseDelay.TotalMilliseconds * Math.Pow(2, retryCount - 1)); // Exponential
                delay += TimeSpan.FromMilliseconds(Random.Shared.Next(0, 500)); // Jitter

                _logger.LogWarning(ex,
                    "⚠️ Attempt {Attempt}/{Max} failed for {OrderId}. Retrying in {Delay:N0}ms",
                    retryCount, MaxRetries, result.Message.Key, delay.TotalMilliseconds);

                await Task.Delay(delay, ct);
            }
            catch (Exception ex)
            {
                // Exhausted retries
                _logger.LogError(ex, "❌ Failed after {Max} retries — sending to DLQ", MaxRetries);
                await _dlq.SendAsync(result, ex, retryCount, GroupId);
                consumer.Commit(result);
                return;
            }
        }
    }

    private async Task DoBusinessLogicAsync(OrderEvent order, CancellationToken ct)
    {
        // ... actual processing ...
        await Task.Delay(100, ct);
    }
}
```

### 3.4 DLQ Reprocessing Consumer

```csharp
// A separate consumer to replay failed messages from DLQ
public class DlqReprocessorService : BackgroundService
{
    // Read from the DLQ
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = new ConsumerBuilder<string, string>(config).Build();
        consumer.Subscribe("orders.dlq");

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(stoppingToken);
            if (result?.Message == null) continue;

            var dlqMessage = JsonSerializer.Deserialize<DeadLetterMessage>(result.Message.Value);
            if (dlqMessage == null) continue;

            // Option 1: Log and alert for manual intervention
            _logger.LogCritical("DLQ message needs attention: {Key} from {Topic}",
                dlqMessage.OriginalKey, dlqMessage.OriginalTopic);

            // Option 2: Re-publish to original topic for reprocessing
            await _producer.ProduceAsync(dlqMessage.OriginalTopic, new Message<string, string>
            {
                Key = dlqMessage.OriginalKey,
                Value = dlqMessage.OriginalValue,
                Headers = new Headers
                {
                    { "reprocessed", Encoding.UTF8.GetBytes("true") },
                    { "reprocessed-at", Encoding.UTF8.GetBytes(DateTime.UtcNow.ToString("O")) }
                }
            });

            consumer.Commit(result);
        }
    }
}
```

---

## 4. Idempotent Consumers

Since we use at-least-once delivery, consumers must be **idempotent** — safe to process the same message multiple times.

```csharp
// Pattern 1: Idempotency via database unique constraint
public async Task ProcessOrderAsync(OrderEvent order)
{
    try
    {
        // Upsert — INSERT or IGNORE if already exists
        await _db.Orders.Upsert(new Order { Id = order.OrderId, ... })
            .On(o => o.Id)    // Conflict on OrderId
            .NoUpdate()       // Don't update if exists (idempotent!)
            .RunAsync();
    }
    catch (UniqueConstraintException)
    {
        _logger.LogInformation("Order {OrderId} already processed — skipping", order.OrderId);
    }
}

// Pattern 2: Idempotency key check
public async Task ProcessOrderAsync(OrderEvent order, string messageOffset)
{
    var idempotencyKey = $"order-{order.OrderId}-{messageOffset}";

    if (await _cache.ExistsAsync(idempotencyKey))
    {
        _logger.LogInformation("Duplicate message detected: {Key}", idempotencyKey);
        return; // Already processed
    }

    // Process the order...
    await DoProcessingAsync(order);

    // Mark as processed (with TTL matching your max retry window)
    await _cache.SetAsync(idempotencyKey, "1", TimeSpan.FromDays(7));
}

// Pattern 3: Event sourcing (inherently idempotent)
// Store events with their offset as part of the event ID
// Duplicate events simply fail to insert (same event ID already exists)
public async Task ProcessAsync(OrderEvent order, TopicPartitionOffset tpo)
{
    var eventId = $"{tpo.Topic}:{tpo.Partition}:{tpo.Offset}";
    // This event ID is globally unique — safe to use as idempotency key
    await _eventStore.AppendAsync(eventId, order);
}
```

---

## 5. Circuit Breaker Pattern

If a downstream service is failing, stop hammering it:

```csharp
using Polly.CircuitBreaker;

var circuitBreaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        // Open circuit after 5 failures in 30 seconds
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 5,

        // Stay open for 30 seconds before trying again
        BreakDuration = TimeSpan.FromSeconds(30),

        OnOpened = args =>
        {
            _logger.LogCritical("Circuit OPENED — downstream service unavailable!");
            // Send alert to your team
            return default;
        },
        OnClosed = args =>
        {
            _logger.LogInformation("Circuit CLOSED — downstream service recovered");
            return default;
        },
        OnHalfOpened = args =>
        {
            _logger.LogInformation("Circuit HALF-OPEN — testing downstream service");
            return default;
        }
    })
    .Build();

// In consumer loop:
try
{
    await circuitBreaker.ExecuteAsync(async ct =>
    {
        await _paymentService.ProcessPaymentAsync(order, ct);
    }, stoppingToken);

    consumer.Commit(result);
}
catch (BrokenCircuitException)
{
    // Circuit is open — pause consumption to avoid piling up failed messages
    _logger.LogWarning("Circuit open — pausing consumption for 10 seconds");
    await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
    // Don't commit — message will be reprocessed when circuit closes
}
```

---

## 6. Error Handling Checklist

```
✅ Distinguish retriable vs non-retriable errors
✅ Never leave a consumer in an infinite retry loop without a limit
✅ Always send truly failed messages to a DLQ
✅ Ensure your consumer is idempotent (safe to process twice)
✅ Monitor DLQ depth — non-zero DLQ count = alert your team
✅ Have a DLQ reprocessor for human-reviewed messages
✅ Use circuit breakers for downstream service calls
✅ Log error context: topic, partition, offset, message key
✅ Set up alerts for consumer lag spikes (consumer stuck on error)
✅ Test your error paths with chaos engineering (inject failures)
```

---

## Summary

You now know:
- ✅ The difference between retriable and non-retriable errors
- ✅ Retry with exponential backoff and jitter
- ✅ Polly integration for resilience policies
- ✅ Dead Letter Queue (DLQ) pattern — full implementation
- ✅ DLQ reprocessor for manual intervention
- ✅ Idempotent consumer patterns (DB, cache, event sourcing)
- ✅ Circuit breaker to protect downstream services

**Next:** [09-KAFKA-WITH-AZURE.md](./09-KAFKA-WITH-AZURE.md) — Azure Event Hubs for Kafka & deployment
