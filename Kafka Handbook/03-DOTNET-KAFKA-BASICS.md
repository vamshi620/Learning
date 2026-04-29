# .NET Kafka Basics
## File 03: Your First .NET Producer and Consumer

---

## What You'll Build

A .NET 8 solution with:
- `KafkaProducer` — Console app that sends order events
- `KafkaConsumer` — Console app that reads and processes order events

**Time Required:** 45-60 minutes

---

## 1. Create the Solution

```powershell
# Create solution folder
mkdir C:\Projects\KafkaDemo
cd C:\Projects\KafkaDemo

# Create solution
dotnet new sln -n KafkaDemo

# Create projects
dotnet new console -n KafkaDemo.Producer -o src/Producer
dotnet new console -n KafkaDemo.Consumer -o src/Consumer
dotnet new classlib -n KafkaDemo.Shared -o src/Shared

# Add projects to solution
dotnet sln add src/Producer/KafkaDemo.Producer.csproj
dotnet sln add src/Consumer/KafkaDemo.Consumer.csproj
dotnet sln add src/Shared/KafkaDemo.Shared.csproj

# Add references
dotnet add src/Producer/KafkaDemo.Producer.csproj reference src/Shared/KafkaDemo.Shared.csproj
dotnet add src/Consumer/KafkaDemo.Consumer.csproj reference src/Shared/KafkaDemo.Shared.csproj
```

### Add NuGet Packages

```powershell
# Core Kafka package
dotnet add src/Producer/KafkaDemo.Producer.csproj package Confluent.Kafka
dotnet add src/Consumer/KafkaDemo.Consumer.csproj package Confluent.Kafka

# Logging
dotnet add src/Producer/KafkaDemo.Producer.csproj package Microsoft.Extensions.Logging.Console
dotnet add src/Consumer/KafkaDemo.Consumer.csproj package Microsoft.Extensions.Logging.Console
```

---

## 2. Shared Models

```csharp
// src/Shared/Models/OrderEvent.cs
namespace KafkaDemo.Shared.Models;

public class OrderEvent
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal Amount { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
    public string EventType { get; set; } = string.Empty;
}

public class OrderItem
{
    public string ProductId { get; set; } = string.Empty;
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

```csharp
// src/Shared/Constants/KafkaTopics.cs
namespace KafkaDemo.Shared.Constants;

public static class KafkaTopics
{
    public const string Orders = "orders";
    public const string Payments = "payments";
    public const string Inventory = "inventory";
    public const string Notifications = "notifications";
    public const string DeadLetterQueue = "dead-letter-queue";
}
```

---

## 3. Basic Producer

```csharp
// src/Producer/Program.cs
using System.Text.Json;
using Confluent.Kafka;
using KafkaDemo.Shared.Constants;
using KafkaDemo.Shared.Models;

Console.WriteLine("=== Kafka Producer Demo ===");

// 1. Configure the producer
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",

    // Acknowledgment: wait for ALL ISR replicas to confirm
    Acks = Acks.All,

    // Retry settings
    MessageSendMaxRetries = 3,
    RetryBackoffMs = 1000,

    // Idempotent producer: prevents duplicate messages on retry
    EnableIdempotence = true,

    // Batching: wait up to 5ms to accumulate messages (higher throughput)
    LingerMs = 5,

    // Batch size in bytes (16KB default, increase for higher throughput)
    BatchSize = 16384,

    // Compression (snappy is good balance of speed/ratio)
    CompressionType = CompressionType.Snappy,
};

// 2. Build the producer
// ProducerBuilder<TKey, TValue> — key is string, value is string (JSON)
using var producer = new ProducerBuilder<string, string>(config)
    .SetErrorHandler((_, error) =>
    {
        Console.WriteLine($"[ERROR] {error.Code}: {error.Reason}");
    })
    .SetLogHandler((_, log) =>
    {
        if (log.Level <= SyslogLevel.Warning)
            Console.WriteLine($"[LOG] {log.Level}: {log.Message}");
    })
    .Build();

// 3. Send messages
var orders = GenerateOrders(10);

foreach (var order in orders)
{
    var message = new Message<string, string>
    {
        // Key: used to determine partition. Same key = same partition = ordered
        Key = order.OrderId,

        // Value: serialized to JSON
        Value = JsonSerializer.Serialize(order),

        // Optional headers
        Headers = new Headers
        {
            { "event-type", System.Text.Encoding.UTF8.GetBytes(order.EventType) },
            { "source", System.Text.Encoding.UTF8.GetBytes("order-service") },
            { "version", System.Text.Encoding.UTF8.GetBytes("1.0") }
        }
    };

    try
    {
        // Async send — awaiting the delivery report
        var deliveryResult = await producer.ProduceAsync(KafkaTopics.Orders, message);

        Console.WriteLine($"✅ Sent order {order.OrderId} to " +
                          $"Partition [{deliveryResult.Partition}] " +
                          $"Offset [{deliveryResult.Offset}]");
    }
    catch (ProduceException<string, string> ex)
    {
        Console.WriteLine($"❌ Failed to send {order.OrderId}: {ex.Error.Reason}");

        // Handle fatal vs retriable errors differently
        if (ex.Error.IsFatal)
        {
            throw; // Fatal error — stop the application
        }
        // Non-fatal: log and continue
    }

    await Task.Delay(500); // Simulate 0.5s between orders
}

// 4. IMPORTANT: Flush before exiting
// Ensures all buffered messages are sent
producer.Flush(TimeSpan.FromSeconds(10));
Console.WriteLine("\n✅ All messages sent. Producer flushed.");

// Helper method to generate test data
static List<OrderEvent> GenerateOrders(int count)
{
    var random = new Random();
    var products = new[]
    {
        ("P001", "Laptop", 999.99m),
        ("P002", "Mouse", 29.99m),
        ("P003", "Keyboard", 79.99m),
        ("P004", "Monitor", 399.99m),
        ("P005", "Headphones", 149.99m)
    };

    return Enumerable.Range(1, count).Select(i =>
    {
        var product = products[random.Next(products.Length)];
        var quantity = random.Next(1, 5);

        return new OrderEvent
        {
            OrderId = $"ORD-{i:D5}",
            CustomerId = $"CUST-{random.Next(1, 100):D3}",
            Status = "placed",
            EventType = "order.placed",
            Amount = product.Item3 * quantity,
            Items = new List<OrderItem>
            {
                new OrderItem
                {
                    ProductId = product.Item1,
                    ProductName = product.Item2,
                    Quantity = quantity,
                    UnitPrice = product.Item3
                }
            }
        };
    }).ToList();
}
```

### Run the Producer

```powershell
# Make sure Kafka is running first!
docker compose ps  # Should show kafka as "Up (healthy)"

# Run the producer
cd src/Producer
dotnet run
```

**Expected output:**
```
=== Kafka Producer Demo ===
✅ Sent order ORD-00001 to Partition [1] Offset [0]
✅ Sent order ORD-00002 to Partition [2] Offset [0]
✅ Sent order ORD-00003 to Partition [0] Offset [0]
✅ Sent order ORD-00004 to Partition [1] Offset [1]
...
✅ All messages sent. Producer flushed.
```

---

## 4. Basic Consumer

```csharp
// src/Consumer/Program.cs
using System.Text.Json;
using Confluent.Kafka;
using KafkaDemo.Shared.Constants;
using KafkaDemo.Shared.Models;

Console.WriteLine("=== Kafka Consumer Demo ===");
Console.WriteLine("Listening for orders... (Press Ctrl+C to stop)");

// 1. Configure the consumer
var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",

    // REQUIRED: Consumer Group ID
    // All instances with same group ID share the work (parallel consumption)
    GroupId = "order-processing-service",

    // What to do when no committed offset exists:
    // Earliest = read from beginning of topic
    // Latest = read only new messages (default)
    AutoOffsetReset = AutoOffsetReset.Earliest,

    // IMPORTANT: Manual offset commit (recommended for reliability)
    EnableAutoCommit = false,

    // Session timeout: if consumer doesn't send heartbeat within this time,
    // it's considered dead and Kafka triggers rebalance
    SessionTimeoutMs = 30000,

    // Heartbeat interval (should be 1/3 of session timeout)
    HeartbeatIntervalMs = 10000,

    // Maximum records per poll
    MaxPollIntervalMs = 300000, // 5 minutes max to process a batch

    // Fetch settings
    FetchMinBytes = 1,
    FetchWaitMaxMs = 500,
};

// 2. Cancellation token for graceful shutdown
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true; // Don't terminate immediately
    cts.Cancel();    // Signal our loop to stop
    Console.WriteLine("\n⏹️  Shutting down consumer...");
};

// 3. Build and run the consumer
using var consumer = new ConsumerBuilder<string, string>(config)
    .SetErrorHandler((_, error) =>
    {
        Console.WriteLine($"[ERROR] {error.Code}: {error.Reason}");
    })
    .SetPartitionsAssignedHandler((c, partitions) =>
    {
        // Called when Kafka assigns partitions to this consumer (after rebalance)
        Console.WriteLine($"[REBALANCE] Assigned partitions: {string.Join(", ", partitions)}");
    })
    .SetPartitionsRevokedHandler((c, partitions) =>
    {
        // Called when partitions are being taken away (before rebalance)
        Console.WriteLine($"[REBALANCE] Revoking partitions: {string.Join(", ", partitions)}");
        // IMPORTANT: Commit any pending offsets before partitions are revoked!
        c.Commit();
    })
    .Build();

// 4. Subscribe to topic(s)
consumer.Subscribe(KafkaTopics.Orders);
// Can subscribe to multiple: consumer.Subscribe(new[] { "orders", "payments" });

int processedCount = 0;

try
{
    while (!cts.Token.IsCancellationRequested)
    {
        try
        {
            // Poll for a message (blocks up to 1 second)
            var consumeResult = consumer.Consume(cts.Token);

            if (consumeResult?.Message == null) continue;

            // 5. Process the message
            await ProcessOrderAsync(consumeResult, consumer);
            processedCount++;
        }
        catch (ConsumeException ex) when (!ex.Error.IsFatal)
        {
            Console.WriteLine($"⚠️  ConsumeException (non-fatal): {ex.Error.Reason}");
            // Continue the loop for non-fatal errors
        }
    }
}
catch (OperationCanceledException)
{
    // Expected on Ctrl+C — graceful shutdown
}
finally
{
    // Close consumer — triggers final commit and leaves the group cleanly
    consumer.Close();
    Console.WriteLine($"✅ Consumer closed. Total processed: {processedCount}");
}

// Processing function
async Task ProcessOrderAsync(
    ConsumeResult<string, string> consumeResult,
    IConsumer<string, string> kafkaConsumer)
{
    var meta = $"P[{consumeResult.Partition}] O[{consumeResult.Offset}]";

    try
    {
        // Deserialize the message value
        var order = JsonSerializer.Deserialize<OrderEvent>(consumeResult.Message.Value);

        if (order == null)
        {
            Console.WriteLine($"[{meta}] ⚠️  Null order — skipping");
            kafkaConsumer.Commit(consumeResult);
            return;
        }

        // Read optional headers
        var eventType = consumeResult.Message.Headers.TryGetLastBytes("event-type", out var headerBytes)
            ? System.Text.Encoding.UTF8.GetString(headerBytes)
            : "unknown";

        Console.WriteLine($"[{meta}] 📦 Processing order: {order.OrderId}" +
                          $" | Customer: {order.CustomerId}" +
                          $" | Amount: {order.Amount:C}" +
                          $" | Event: {eventType}");

        // Simulate processing work (DB write, API call, etc.)
        await Task.Delay(200);

        // --- YOUR BUSINESS LOGIC GOES HERE ---
        // Example: await orderRepository.SaveAsync(order);
        // Example: await inventoryService.ReserveAsync(order);

        Console.WriteLine($"[{meta}] ✅ Order {order.OrderId} processed");

        // 6. COMMIT the offset AFTER successful processing
        // This tells Kafka "I've handled everything up to this offset"
        kafkaConsumer.Commit(consumeResult);
    }
    catch (JsonException ex)
    {
        Console.WriteLine($"[{meta}] ❌ JSON deserialization failed: {ex.Message}");
        // Bad message — commit it to skip (or send to DLQ — see [File 08](./08-ERROR-HANDLING.md))
        kafkaConsumer.Commit(consumeResult);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[{meta}] ❌ Processing failed: {ex.Message}");
        // DON'T commit — on restart, this message will be reprocessed
        // This is "at-least-once" delivery
        throw; // Or implement retry logic (see [File 08](./08-ERROR-HANDLING.md))
    }
}
```

### Run the Consumer

```powershell
# In a NEW terminal window, from the solution root:
cd src/Consumer
dotnet run
```

**Expected output:**
```
=== Kafka Consumer Demo ===
Listening for orders... (Press Ctrl+C to stop)
[REBALANCE] Assigned partitions: [orders] [0], [1], [2]
[P[1] O[0]] 📦 Processing order: ORD-00001 | Customer: CUST-042 | Amount: $999.99 | Event: order.placed
[P[1] O[0]] ✅ Order ORD-00001 processed
[P[2] O[0]] 📦 Processing order: ORD-00002 | Customer: CUST-017 | Amount: $149.99 | Event: order.placed
...
```

---

## 4.5. Configuration via appsettings.json

So far, we hardcoded `localhost:9092` and the `GroupId`. In a real .NET application, you should use `appsettings.json` and the `IConfiguration` binder.

**appsettings.json:**
```json
{
  "Kafka": {
    "BootstrapServers": "localhost:9092",
    "GroupId": "order-processing-service",
    "AutoOffsetReset": "Earliest",
    "EnableAutoCommit": false
  }
}
```

**Binding in C#:**
```csharp
// Read config section
var kafkaOptions = builder.Configuration.GetSection("Kafka");

// Bind to strongly-typed ConsumerConfig
var config = new ConsumerConfig();
kafkaOptions.Bind(config);

// Use it
var consumer = new ConsumerBuilder<string, string>(config).Build();
```

---

## 5. Understanding Auto-Commit vs Manual Commit

> ⚠️ **This is one of the most important concepts for reliability.**

### Auto-Commit (Default — AVOID in production)

```csharp
EnableAutoCommit = true,
AutoCommitIntervalMs = 5000  // commits every 5 seconds
```

**Problem:**
```
Consumer receives message at offset 10 → starts processing
Auto-commit fires → commits offset 11 ✅
Consumer crashes mid-processing ❌
Consumer restarts → starts from offset 11 → MESSAGE LOST!
```

### Manual Commit (Recommended)

```csharp
EnableAutoCommit = false  // You control when to commit

// After successful processing:
consumer.Commit(consumeResult);
// OR commit all partitions:
consumer.Commit();
```

**Result:**
```
Consumer receives message at offset 10 → starts processing
Consumer crashes mid-processing ❌
Consumer restarts → starts from offset 10 → REPROCESSES safely ✅
```

---

## 6. Using Dependency Injection (.NET Style)

In real .NET projects, you use DI. Here's how to wire Kafka with `IHostedService`:

```csharp
// src/Consumer/KafkaConsumerService.cs
using Confluent.Kafka;
using KafkaDemo.Shared.Constants;
using KafkaDemo.Shared.Models;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Text.Json;

namespace KafkaDemo.Consumer;

public class OrderConsumerService : BackgroundService
{
    private readonly ILogger<OrderConsumerService> _logger;
    private readonly IConsumer<string, string> _consumer;

    public OrderConsumerService(ILogger<OrderConsumerService> logger)
    {
        _logger = logger;

        var config = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "order-processing-service",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,
        };

        _consumer = new ConsumerBuilder<string, string>(config).Build();
        _consumer.Subscribe(KafkaTopics.Orders);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order consumer started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var result = _consumer.Consume(stoppingToken);

                if (result?.Message == null) continue;

                await ProcessAsync(result, stoppingToken);
                _consumer.Commit(result);
            }
            catch (OperationCanceledException)
            {
                break; // Graceful shutdown
            }
            catch (ConsumeException ex) when (!ex.Error.IsFatal)
            {
                _logger.LogWarning("Non-fatal consume error: {Reason}", ex.Error.Reason);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unexpected error in consumer loop");
                await Task.Delay(1000, stoppingToken); // Back off before retrying
            }
        }

        _consumer.Close();
        _logger.LogInformation("Order consumer stopped");
    }

    private async Task ProcessAsync(ConsumeResult<string, string> result, CancellationToken ct)
    {
        var order = JsonSerializer.Deserialize<OrderEvent>(result.Message.Value);
        _logger.LogInformation("Processing order {OrderId} from partition {Partition} offset {Offset}",
            order?.OrderId, result.Partition.Value, result.Offset.Value);

        // Business logic here...
        await Task.Delay(100, ct);
    }

    public override void Dispose()
    {
        _consumer.Dispose();
        base.Dispose();
    }
}
```

```csharp
// src/Consumer/Program.cs (with DI)
using KafkaDemo.Consumer;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<OrderConsumerService>();
    })
    .Build();

await host.RunAsync();
```

---

## 7. ProduceAsync vs Produce (Fire-and-Forget)

```csharp
// Option 1: ProduceAsync — awaitable, get delivery report
var result = await producer.ProduceAsync("orders", message);
Console.WriteLine($"Delivered to {result.TopicPartitionOffset}");

// Option 2: Produce — async callback, higher throughput
producer.Produce("orders", message, deliveryReport =>
{
    if (deliveryReport.Error.IsError)
        Console.WriteLine($"Delivery failed: {deliveryReport.Error.Reason}");
    else
        Console.WriteLine($"Delivered: {deliveryReport.TopicPartitionOffset}");
});
producer.Flush(TimeSpan.FromSeconds(10)); // Must flush before exit!
```

> 💡 **Pro Tip:** Use `ProduceAsync` for clarity and correctness. Use `Produce` with callback only when you need maximum throughput and handle the async complexity carefully.

---

## 8. Serialization Summary

```csharp
// String key, String value (JSON serialized manually)
var producer = new ProducerBuilder<string, string>(config).Build();

// String key, Byte array value (for custom binary formats)
var producer = new ProducerBuilder<string, byte[]>(config).Build();

// Null key (when ordering doesn't matter)
var producer = new ProducerBuilder<Null, string>(config).Build();
var message = new Message<Null, string> { Key = Null.Value, Value = json };

// Custom type with custom serializer (see [File 07](./07-SERIALIZATION.md) for Avro/Protobuf)
var producer = new ProducerBuilder<string, OrderEvent>(config)
    .SetValueSerializer(new JsonSerializer<OrderEvent>())
    .Build();
```

---

## 9. 🧪 Exercise

1. Run the producer — send 20 orders
2. Start the consumer — observe messages being consumed
3. Stop the consumer mid-way (Ctrl+C)
4. Restart the consumer — it should resume from where it left off (not from the beginning)
5. Open Kafka UI at `http://localhost:8080` → Topics → orders → Consumers tab
6. Observe the LAG decreasing as the consumer catches up

**Bonus:** Start TWO instances of the consumer in separate terminals. Observe how Kafka distributes the 3 partitions between them (rebalancing).

---

## Summary

You've built:
- ✅ A .NET producer with proper configuration (acks, retry, idempotence)
- ✅ A .NET consumer with manual commit (at-least-once delivery)
- ✅ A hosted service consumer using `BackgroundService`
- ✅ Understood auto-commit vs manual commit
- ✅ Understand fire-and-forget vs awaitable produce

**Next:** [04-PRODUCERS-DEEP-DIVE.md](./04-PRODUCERS-DEEP-DIVE.md) — Advanced producer patterns
