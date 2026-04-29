# Advanced Producers in .NET
## File 04: Producer Patterns, Configuration & Best Practices

---

## What You'll Learn

- Producer configuration tuning for throughput vs reliability
- Partitioning strategies and custom partitioners
- Transactional producers (exactly-once semantics)
- Producing with headers and metadata
- Batch producing patterns
- Producer metrics and monitoring hooks

---

## 1. Producer Configuration Deep Dive

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",

    // ─── Reliability Settings ──────────────────────────────
    
    // Acks: How many brokers must confirm before success
    // Acks.None (0)  = Fire and forget — fastest, may lose messages
    // Acks.Leader (1) = Leader only — fast, loses if leader dies
    // Acks.All (-1)  = All ISR replicas — slowest, most durable (USE THIS)
    Acks = Acks.All,

    // Idempotent: ensures exactly-one delivery on retry (requires Acks.All)
    EnableIdempotence = true,

    // Retry settings
    MessageSendMaxRetries = 5,       // Max retry attempts
    RetryBackoffMs = 200,            // Initial backoff between retries
    // Backoff is exponential: 200ms, 400ms, 800ms...

    // ─── Throughput Settings ───────────────────────────────

    // LingerMs: Wait this long for more messages before sending a batch
    // 0 = send immediately (lowest latency)
    // 5-100 = better throughput (slight latency increase)
    LingerMs = 5,

    // BatchSize: Max bytes per batch (per partition)
    // Default: 16384 (16KB). Increase to 65536 (64KB) for high throughput
    BatchSize = 65536,

    // Max bytes the producer will use for buffering
    // If full, ProduceAsync blocks until space is available
    QueueBufferingMaxMessages = 100000,
    QueueBufferingMaxKbytes = 1048576, // 1 GB

    // Max time (ms) to wait for buffer space
    QueueBlockingEnabled = true,

    // ─── Compression ───────────────────────────────────────
    // None     = No compression (fastest, most bytes)
    // Gzip     = Good ratio, slower CPU
    // Snappy   = Good balance (Google's fast compression)
    // Lz4      = Fastest, decent ratio
    // Zstd     = Best ratio + good speed (Kafka 2.1+)
    CompressionType = CompressionType.Snappy,

    // ─── Message Settings ──────────────────────────────────
    // Max message size (must match broker's message.max.bytes)
    MessageMaxBytes = 10485760, // 10 MB

    // Timeout for produce requests
    RequestTimeoutMs = 30000,

    // Total delivery timeout (including all retries)
    DeliveryTimeoutMs = 120000, // 2 minutes

    // ─── Connection Settings ───────────────────────────────
    SocketKeepaliveEnable = true,
    MetadataMaxAgeMs = 300000, // Refresh metadata every 5 minutes
};
```

### Configuration Profiles

```csharp
public static class KafkaProducerProfiles
{
    // HIGH RELIABILITY — payment systems, critical data
    public static ProducerConfig HighReliability(string bootstrapServers) => new()
    {
        BootstrapServers = bootstrapServers,
        Acks = Acks.All,
        EnableIdempotence = true,
        MessageSendMaxRetries = 10,
        RetryBackoffMs = 500,
        LingerMs = 0,   // Send immediately, don't batch
        CompressionType = CompressionType.None,
        DeliveryTimeoutMs = 300000, // 5 minutes
    };

    // HIGH THROUGHPUT — analytics, logging, metrics
    public static ProducerConfig HighThroughput(string bootstrapServers) => new()
    {
        BootstrapServers = bootstrapServers,
        Acks = Acks.Leader,
        LingerMs = 100,
        BatchSize = 131072,   // 128 KB
        CompressionType = CompressionType.Lz4,
        QueueBufferingMaxMessages = 500000,
    };

    // BALANCED — general purpose microservices
    public static ProducerConfig Balanced(string bootstrapServers) => new()
    {
        BootstrapServers = bootstrapServers,
        Acks = Acks.All,
        EnableIdempotence = true,
        LingerMs = 5,
        BatchSize = 65536,
        CompressionType = CompressionType.Snappy,
        MessageSendMaxRetries = 3,
    };
}
```

---

## 2. Partitioning Strategies

### 2.1 Default Partitioning Behavior

```csharp
// With key: hash(key) % numPartitions → same key ALWAYS same partition
var message = new Message<string, string>
{
    Key = "customer-123",  // All events for customer-123 → same partition (ordered!)
    Value = json
};

// Without key: round-robin across partitions (max throughput, no ordering)
var message = new Message<string, string>
{
    Key = null,
    Value = json
};
```

### 2.2 Custom Partitioner

```csharp
// Custom partitioner: route by geographic region
public class RegionPartitioner : IPartitioner
{
    public int Partition(string topic, Type keyType, Type valueType,
        byte[] key, byte[] val, int partitionCount, ISerializingProducer<byte[], byte[]> rp)
    {
        if (key == null) return 0;

        var keyString = System.Text.Encoding.UTF8.GetString(key);

        // Region-based routing
        return keyString switch
        {
            var k when k.StartsWith("US-") => 0,
            var k when k.StartsWith("EU-") => 1,
            var k when k.StartsWith("APAC-") => 2,
            _ => Math.Abs(keyString.GetHashCode()) % partitionCount
        };
    }

    public void Dispose() { }
}

// Register the custom partitioner
var producer = new ProducerBuilder<string, string>(config)
    .SetPartitioner("orders", new RegionPartitioner())
    .Build();
```

### 2.3 Sticky Partitioner (Built-in, High Throughput)

The default behavior since Kafka 2.4 uses "sticky partitioning" — messages without keys
stick to a single partition until the batch is full, then switch. This improves batching efficiency.

```csharp
// Sticky partitioning is enabled by default for null-key messages
// No configuration needed — Confluent.Kafka handles this automatically
```

---

## 3. Producing with Headers

Headers carry metadata without polluting the message body:

```csharp
var message = new Message<string, string>
{
    Key = order.OrderId,
    Value = JsonSerializer.Serialize(order),
    Headers = new Headers
    {
        // Correlation ID for distributed tracing
        { "correlation-id", Encoding.UTF8.GetBytes(Guid.NewGuid().ToString()) },
        // Causation ID: which command/event caused this event
        { "causation-id", Encoding.UTF8.GetBytes(previousEventId) },
        // Schema version for backward compatibility
        { "schema-version", Encoding.UTF8.GetBytes("2.1") },
        // Source service for observability
        { "source-service", Encoding.UTF8.GetBytes("order-service") },
        // Environment
        { "environment", Encoding.UTF8.GetBytes("production") },
        // Tenant ID for multi-tenant apps
        { "tenant-id", Encoding.UTF8.GetBytes(tenantId) },
    }
};

// Reading headers in consumer:
if (result.Message.Headers.TryGetLastBytes("correlation-id", out var correlationBytes))
{
    var correlationId = Encoding.UTF8.GetString(correlationBytes);
    logger.LogInformation("Processing with correlation ID: {CorrelationId}", correlationId);
}
```

---

## 4. Transactional Producer (Exactly-Once Semantics)

Transactions ensure a group of messages are either ALL delivered or NONE:

```csharp
// Step 1: Configure with transactional ID (unique per producer instance)
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    TransactionalId = "order-service-producer-1", // MUST be unique per producer
    Acks = Acks.All,
    EnableIdempotence = true, // Required for transactions
};

using var producer = new ProducerBuilder<string, string>(config).Build();

// Step 2: Initialize transactions ONCE at startup
producer.InitTransactions(TimeSpan.FromSeconds(30));

// Step 3: Wrap related messages in a transaction
try
{
    producer.BeginTransaction();

    // Send multiple messages atomically
    await producer.ProduceAsync("orders", new Message<string, string>
    {
        Key = order.OrderId,
        Value = JsonSerializer.Serialize(new { orderId = order.OrderId, status = "placed" })
    });

    await producer.ProduceAsync("inventory", new Message<string, string>
    {
        Key = order.OrderId,
        Value = JsonSerializer.Serialize(new { orderId = order.OrderId, action = "reserve" })
    });

    await producer.ProduceAsync("audit-logs", new Message<string, string>
    {
        Key = order.OrderId,
        Value = JsonSerializer.Serialize(new { orderId = order.OrderId, action = "order_placed" })
    });

    // Commit: all 3 messages visible to consumers atomically
    producer.CommitTransaction();
    Console.WriteLine("Transaction committed — all messages delivered");
}
catch (KafkaException ex)
{
    Console.WriteLine($"Transaction failed: {ex.Error.Reason}");
    producer.AbortTransaction(); // All messages rolled back
    throw;
}
```

> ⚠️ **Common Mistake:** Don't use transactional producers for every use case — they add latency. Use only when you need atomic multi-topic writes.

---

## 5. Typed Generic Producer with DI

```csharp
// IKafkaProducer<T> — typed abstraction
public interface IKafkaProducer<TKey, TValue>
{
    Task<DeliveryResult<TKey, TValue>> ProduceAsync(
        string topic, TKey key, TValue value,
        Dictionary<string, string>? headers = null,
        CancellationToken cancellationToken = default);
}

// Implementation
public class KafkaProducer<TKey, TValue> : IKafkaProducer<TKey, TValue>, IDisposable
{
    private readonly IProducer<TKey, TValue> _producer;
    private readonly ILogger<KafkaProducer<TKey, TValue>> _logger;

    public KafkaProducer(ProducerConfig config, ILogger<KafkaProducer<TKey, TValue>> logger)
    {
        _logger = logger;
        _producer = new ProducerBuilder<TKey, TValue>(config)
            .SetValueSerializer(new KafkaJsonSerializer<TValue>())
            .SetErrorHandler((_, e) => _logger.LogError("Kafka error: {Reason}", e.Reason))
            .Build();
    }

    public async Task<DeliveryResult<TKey, TValue>> ProduceAsync(
        string topic, TKey key, TValue value,
        Dictionary<string, string>? headers = null,
        CancellationToken cancellationToken = default)
    {
        var message = new Message<TKey, TValue>
        {
            Key = key,
            Value = value,
            Headers = BuildHeaders(headers)
        };

        try
        {
            var result = await _producer.ProduceAsync(topic, message, cancellationToken);
            _logger.LogDebug("Produced to {TopicPartitionOffset}", result.TopicPartitionOffset);
            return result;
        }
        catch (ProduceException<TKey, TValue> ex)
        {
            _logger.LogError("Failed to produce to {Topic}: {Reason}", topic, ex.Error.Reason);
            throw;
        }
    }

    private static Headers BuildHeaders(Dictionary<string, string>? headers)
    {
        var kafkaHeaders = new Headers();
        if (headers == null) return kafkaHeaders;

        foreach (var (key, value) in headers)
            kafkaHeaders.Add(key, Encoding.UTF8.GetBytes(value));

        return kafkaHeaders;
    }

    public void Dispose()
    {
        _producer.Flush(TimeSpan.FromSeconds(10));
        _producer.Dispose();
    }
}

// Custom JSON serializer for Kafka
// NOTE: Named KafkaJsonSerializer to avoid collision with System.Text.Json.JsonSerializer
public class KafkaJsonSerializer<T> : ISerializer<T>
{
    public byte[] Serialize(T data, SerializationContext context)
        => System.Text.Json.JsonSerializer.SerializeToUtf8Bytes(data);
}
```

### Register in DI

```csharp
// Program.cs or Startup.cs
services.AddSingleton(new ProducerConfig
{
    BootstrapServers = configuration["Kafka:BootstrapServers"],
    Acks = Acks.All,
    EnableIdempotence = true,
    LingerMs = 5,
});

services.AddSingleton(typeof(IKafkaProducer<,>), typeof(KafkaProducer<,>));

// Usage in a service
public class OrderService
{
    private readonly IKafkaProducer<string, OrderEvent> _producer;

    public OrderService(IKafkaProducer<string, OrderEvent> producer)
    {
        _producer = producer;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        // ... business logic ...

        var orderEvent = new OrderEvent { OrderId = order.Id, Status = "placed" };
        await _producer.ProduceAsync("orders", order.Id, orderEvent);
    }
}
```

---

## 6. Outbox Pattern (Reliable Produce)

The biggest challenge: your database write and Kafka produce must succeed together.
If the DB write succeeds but Kafka fails, you have an inconsistent state.

**Solution: The Outbox Pattern**

```
Normal (risky):
1. Save order to DB           ✅
2. Publish event to Kafka      ❌ ← Kafka down? Event lost!

Outbox Pattern (safe):
1. Save order to DB            }
2. Save event to Outbox table  } ← SAME database transaction (atomic!)
3. Background relay reads Outbox → publishes to Kafka
4. Marks Outbox row as "sent"

Result: If DB write succeeds, the event WILL eventually reach Kafka.
```

The key idea: instead of producing to Kafka directly, you write the message into an `OutboxMessages` table **inside the same DB transaction** as your business entity. A background relay service then picks up pending messages and publishes them to Kafka.

> 📌 **Full implementation:** See [10-ADVANCED-PATTERNS.md](./10-ADVANCED-PATTERNS.md) (Section 5) for the complete Outbox implementation with EF Core, retry counts, and status tracking.

> 💡 **Pro Tip:** Libraries like **MassTransit** (covered in [File 11](./11-MASSTRANSIT-KAFKA.md)) implement the outbox pattern for you with `AddEntityFrameworkOutbox`. Use them in production rather than building your own.

---

## 7. Producer Health Monitoring

```csharp
// Track producer statistics
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    StatisticsIntervalMs = 5000, // Report stats every 5 seconds
};

var producer = new ProducerBuilder<string, string>(config)
    .SetStatisticsHandler((_, json) =>
    {
        // json is a rich JSON document with metrics
        // Parse with System.Text.Json for specific metrics
        using var doc = JsonDocument.Parse(json);
        var txMsgs = doc.RootElement.GetProperty("txmsgs").GetInt64();
        var txBytes = doc.RootElement.GetProperty("txbytes").GetInt64();
        Console.WriteLine($"[STATS] Produced: {txMsgs} messages, {txBytes} bytes");
    })
    .Build();
```

---

## 8. Common Producer Mistakes

### Mistake 1: Not Flushing Before Application Exit

```csharp
// ❌ WRONG — messages in buffer may be lost
producer.Dispose();

// ✅ CORRECT — flush first, then dispose
producer.Flush(TimeSpan.FromSeconds(10));
producer.Dispose();
// OR use 'using' — IDisposable calls Flush internally in Confluent.Kafka
using var producer = new ProducerBuilder<string, string>(config).Build();
```

### Mistake 2: Creating a New Producer Per Message

```csharp
// ❌ WRONG — very expensive, loses all batching benefits
public async Task SendAsync(OrderEvent order)
{
    using var producer = new ProducerBuilder<string, string>(config).Build();
    await producer.ProduceAsync("orders", ...);
}

// ✅ CORRECT — create once, reuse throughout application lifetime
private readonly IProducer<string, string> _producer; // Singleton!
```

### Mistake 3: Ignoring Delivery Errors

```csharp
// ❌ WRONG — silently ignores errors
producer.Produce("orders", message, _ => { });

// ✅ CORRECT — always handle errors
producer.Produce("orders", message, deliveryReport =>
{
    if (deliveryReport.Error.IsError)
        logger.LogError("Delivery failed: {Reason}", deliveryReport.Error.Reason);
});
```

### Mistake 4: Using TransactionalId Without Understanding

```csharp
// ❌ WRONG — two producers with same transactional ID running simultaneously
// The second one will fence (kill) the first!
// Each producer instance needs a UNIQUE transactional ID

// ✅ CORRECT — include instance identifier
TransactionalId = $"order-service-{Environment.MachineName}-{Process.GetCurrentProcess().Id}"
```

---

## Summary

You now know:
- ✅ All important `ProducerConfig` settings and when to use them
- ✅ Custom partitioning strategies
- ✅ How to use headers for metadata
- ✅ Transactional producers for atomic multi-topic writes
- ✅ Generic typed producer with DI
- ✅ The Outbox Pattern for reliable message publishing
- ✅ Common mistakes to avoid

**Next:** [05-CONSUMERS-DEEP-DIVE.md](./05-CONSUMERS-DEEP-DIVE.md) — Advanced consumer patterns
