# Advanced Consumers in .NET
## File 05: Consumer Groups, Rebalancing & Advanced Patterns

---

## What You'll Learn

- Consumer group mechanics in depth
- Partition assignment strategies
- Handling rebalances gracefully
- Concurrent and parallel consumption patterns
- Batch consumption
- Consumer lag monitoring
- Seeking and replaying messages

---

## 1. Consumer Group Deep Dive

### How Kafka Assigns Partitions

```
Topic: "orders" (6 partitions: P0-P5)

─── Scenario: 1 consumer ───
ConsumerA: [P0, P1, P2, P3, P4, P5]  ← All partitions

─── Scenario: 2 consumers ───
ConsumerA: [P0, P1, P2]
ConsumerB: [P3, P4, P5]

─── Scenario: 3 consumers ───
ConsumerA: [P0, P1]
ConsumerB: [P2, P3]
ConsumerC: [P4, P5]

─── Scenario: 6 consumers (max parallelism) ───
ConsumerA: [P0]
ConsumerB: [P1]
ConsumerC: [P2]
ConsumerD: [P3]
ConsumerE: [P4]
ConsumerF: [P5]

─── Scenario: 7 consumers (1 idle!) ───
ConsumerA-F: [P0-P5]
ConsumerG: []  ← IDLE — no partition assigned!
```

### Consumer Group Assignment Strategies

```csharp
var config = new ConsumerConfig
{
    // Range assignor (default): assigns ranges of partitions to consumers
    // Consumers get consecutive partitions (P0-P2, P3-P5)
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.Range,

    // RoundRobin: rotates partitions across consumers
    // Better load balance when topics have different partition counts
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.RoundRobin,

    // CooperativeSticky: incremental rebalance (Kafka 2.4+)
    // BEST OPTION: consumers keep their current partitions during rebalance
    // Only partitions that need to move are revoked — less disruption!
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.CooperativeSticky,
};
```

> 💡 **Pro Tip:** Use `CooperativeSticky` in production. Traditional rebalancing stops ALL consumers from processing during rebalance ("stop-the-world"). CooperativeSticky does it incrementally — much less disruption.

---

## 2. Handling Rebalances Properly

```csharp
using var consumer = new ConsumerBuilder<string, string>(config)
    .SetPartitionsAssignedHandler((c, partitions) =>
    {
        // Called when NEW partitions are assigned to this consumer
        Console.WriteLine($"Assigned: {string.Join(", ", partitions)}");

        // Optionally: seek to a specific offset on assignment
        // Useful for custom offset management
        var offsets = partitions.Select(tp =>
            new TopicPartitionOffset(tp, Offset.Unset)); // Use committed offset
        return offsets;

        // Or reset to beginning:
        // return partitions.Select(tp => new TopicPartitionOffset(tp, Offset.Beginning));
    })
    .SetPartitionsRevokedHandler((c, partitions) =>
    {
        // Called when partitions are being taken away (BEFORE rebalance)
        Console.WriteLine($"Revoking: {string.Join(", ", partitions)}");

        // CRITICAL: Commit any pending offsets before losing these partitions!
        // Otherwise, work already done might be reprocessed by the new owner
        try
        {
            c.Commit();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Commit before revoke failed: {ex.Message}");
        }
    })
    .SetPartitionsLostHandler((c, partitions) =>
    {
        // Called when partitions are lost unexpectedly (e.g., consumer timeout)
        // Different from revoked — no chance to commit!
        Console.WriteLine($"Lost partitions: {string.Join(", ", partitions)}");
        // Log for investigation — may need to reprocess from last committed offset
    })
    .Build();
```

---

## 3. Concurrent Consumer Pattern

Run multiple consumption threads (one per partition):

```csharp
// ConcurrentConsumerService.cs
public class ConcurrentOrderConsumer : BackgroundService
{
    private readonly ILogger<ConcurrentOrderConsumer> _logger;
    private readonly IServiceProvider _serviceProvider;
    private const string Topic = "orders";
    private const string GroupId = "order-service";

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Discover how many partitions the topic has
        var adminConfig = new AdminClientConfig { BootstrapServers = "localhost:9092" };
        using var adminClient = new AdminClientBuilder(adminConfig).Build();
        var metadata = adminClient.GetMetadata(Topic, TimeSpan.FromSeconds(10));
        var partitionCount = metadata.Topics.First(t => t.Topic == Topic).Partitions.Count;

        _logger.LogInformation("Starting {Count} consumer threads", partitionCount);

        // Launch one consumer per partition
        var tasks = Enumerable.Range(0, partitionCount)
            .Select(partition => ConsumePartitionAsync(partition, stoppingToken))
            .ToArray();

        await Task.WhenAll(tasks);
    }

    private async Task ConsumePartitionAsync(int partitionNumber, CancellationToken ct)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = GroupId,
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,
        };

        using var consumer = new ConsumerBuilder<string, string>(config).Build();

        // Manually assign a specific partition (not subscribe)
        // This bypasses Kafka's automatic partition assignment
        consumer.Assign(new TopicPartition(Topic, new Partition(partitionNumber)));

        _logger.LogInformation("Consumer thread started for partition {Partition}", partitionNumber);

        try
        {
            while (!ct.IsCancellationRequested)
            {
                var result = consumer.Consume(ct);
                if (result?.Message == null) continue;

                await ProcessAsync(result, consumer, ct);
            }
        }
        catch (OperationCanceledException) { }
        finally
        {
            consumer.Close();
        }
    }

    private async Task ProcessAsync(
        ConsumeResult<string, string> result,
        IConsumer<string, string> consumer,
        CancellationToken ct)
    {
        // ... your processing logic ...
        await Task.Delay(100, ct);
        consumer.Commit(result);
    }
}
```

> ⚠️ **Warning:** Using `Assign` instead of `Subscribe` bypasses Kafka's consumer group management. You're responsible for offset management and won't benefit from automatic rebalancing. Use this only when you need tight control.

---

## 4. Channel-Based Parallel Processing

Better pattern: single consumer thread reads from Kafka, worker threads process in parallel:

```csharp
// Channel-based concurrent consumer: separates I/O from CPU work
public class ChannelConsumerService : BackgroundService
{
    private readonly Channel<ConsumeResult<string, string>> _channel;
    private readonly int _workerCount;
    private IConsumer<string, string>? _consumer;

    public ChannelConsumerService(ILogger<ChannelConsumerService> logger)
    {
        _workerCount = Environment.ProcessorCount; // One worker per CPU
        _channel = Channel.CreateBounded<ConsumeResult<string, string>>(
            new BoundedChannelOptions(1000)
            {
                SingleWriter = true,                   // Only one consumer thread writes
                SingleReader = false,                   // Multiple workers read
                FullMode = BoundedChannelFullMode.Wait  // Backpressure: pause if full
            });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer = BuildConsumer();
        _consumer.Subscribe("orders");

        // Start worker tasks
        var workers = Enumerable.Range(0, _workerCount)
            .Select(_ => WorkerAsync(stoppingToken))
            .ToArray();

        // Main consumer loop (single thread)
        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                var result = _consumer.Consume(stoppingToken);
                if (result?.Message != null)
                    await _channel.Writer.WriteAsync(result, stoppingToken);
            }
        }
        catch (OperationCanceledException) { }
        finally
        {
            _channel.Writer.Complete();
            await Task.WhenAll(workers);
            _consumer.Close();
        }
    }

    private async Task WorkerAsync(CancellationToken ct)
    {
        await foreach (var result in _channel.Reader.ReadAllAsync(ct))
        {
            try
            {
                // Process the message
                var order = JsonSerializer.Deserialize<OrderEvent>(result.Message.Value);
                await ProcessOrderAsync(order!, ct);

                // Commit after processing
                // NOTE: In a multi-worker scenario, commits may be out of order
                // Consider batch commits from the consumer thread instead
                _consumer!.StoreOffset(result);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Worker error: {ex.Message}");
            }
        }
    }

    private IConsumer<string, string> BuildConsumer()
    {
        return new ConsumerBuilder<string, string>(new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "order-service",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = true,    // Use auto-commit with StoreOffset for batch commits
            AutoCommitIntervalMs = 5000,
            EnableAutoOffsetStore = false, // We manually call StoreOffset
        }).Build();
    }

    private async Task ProcessOrderAsync(OrderEvent order, CancellationToken ct)
    {
        await Task.Delay(50, ct); // Simulate processing
    }
}
```

---

## 5. Batch Consumer Pattern

Read multiple messages at once for bulk operations (e.g., bulk DB inserts):

```csharp
public class BatchConsumerService : BackgroundService
{
    private const int BatchSize = 100;
    private const int BatchTimeoutMs = 5000; // Flush after 5 seconds even if batch not full

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "analytics-service",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,
            // Fetch more data per poll for batching
            MaxPartitionFetchBytes = 10485760, // 10 MB per partition
            FetchMinBytes = 1048576,           // Wait for 1 MB before returning
            FetchWaitMaxMs = 500,
        };

        using var consumer = new ConsumerBuilder<string, string>(config).Build();
        consumer.Subscribe("orders");

        var batch = new List<ConsumeResult<string, string>>(BatchSize);
        var batchStart = DateTime.UtcNow;
        ConsumeResult<string, string>? lastResult = null;

        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                // Try to consume (short timeout to check batch conditions)
                var result = consumer.Consume(TimeSpan.FromMilliseconds(200));

                if (result?.Message != null)
                {
                    batch.Add(result);
                    lastResult = result;
                }

                // Flush batch when: full OR timeout elapsed
                bool batchFull = batch.Count >= BatchSize;
                bool timeoutElapsed = (DateTime.UtcNow - batchStart).TotalMilliseconds >= BatchTimeoutMs;

                if ((batchFull || timeoutElapsed) && batch.Count > 0)
                {
                    await ProcessBatchAsync(batch, stoppingToken);

                    // Commit the last offset of the batch
                    if (lastResult != null)
                        consumer.Commit(lastResult);

                    Console.WriteLine($"Processed batch of {batch.Count} messages");
                    batch.Clear();
                    batchStart = DateTime.UtcNow;
                    lastResult = null;
                }
            }
        }
        catch (OperationCanceledException) { }
        finally
        {
            // Process remaining items in partial batch
            if (batch.Count > 0)
                await ProcessBatchAsync(batch, CancellationToken.None);

            consumer.Close();
        }
    }

    private async Task ProcessBatchAsync(
        List<ConsumeResult<string, string>> batch,
        CancellationToken ct)
    {
        // Parse all messages
        var orders = batch
            .Select(r => JsonSerializer.Deserialize<OrderEvent>(r.Message.Value))
            .Where(o => o != null)
            .ToList();

        // Bulk insert to database (much more efficient than one-by-one)
        // await _repository.BulkInsertAsync(orders!, ct);
        Console.WriteLine($"Bulk inserting {orders.Count} orders...");
        await Task.Delay(50, ct); // Simulate bulk DB insert
    }
}
```

---

## 6. Seeking & Replaying Messages

```csharp
// Seek to a specific offset
consumer.Seek(new TopicPartitionOffset("orders", partition: 2, offset: 100));

// Seek to the beginning of all assigned partitions
foreach (var tp in consumer.Assignment)
    consumer.Seek(new TopicPartitionOffset(tp, Offset.Beginning));

// Seek to a specific timestamp (replay last hour)
var timestampToSeekTo = DateTimeOffset.UtcNow.AddHours(-1).ToUnixTimeMilliseconds();
var offsetsForTimes = consumer.OffsetsForTimes(
    consumer.Assignment.Select(tp => new TopicPartitionTimestamp(tp, new Timestamp(timestampToSeekTo))),
    TimeSpan.FromSeconds(10)
);

foreach (var offsetResult in offsetsForTimes)
{
    if (offsetResult.Offset != Offset.Unset)
        consumer.Seek(offsetResult);
}
Console.WriteLine("Rewound to 1 hour ago — replaying messages...");
```

### Reset Offsets via CLI (for Reprocessing)

```powershell
# Reset consumer group to beginning (reprocess ALL messages)
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service \
  --topic orders \
  --reset-offsets \
  --to-earliest \
  --execute

# Reset to specific datetime
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service \
  --topic orders \
  --reset-offsets \
  --to-datetime 2024-06-01T00:00:00.000 \
  --execute
```

---

## 7. Consumer Lag Monitoring

```csharp
// Check lag programmatically
public class ConsumerLagMonitor
{
    public async Task<Dictionary<string, long>> GetLagAsync(
        string bootstrapServers, string groupId, string topic)
    {
        var adminConfig = new AdminClientConfig { BootstrapServers = bootstrapServers };
        using var adminClient = new AdminClientBuilder(adminConfig).Build();

        // Get committed offsets for the group
        var groupOffsets = adminClient.ListConsumerGroupOffsets(
            new[] { new ConsumerGroupTopicPartitions(groupId) });

        // Get end offsets (latest position)
        var topicMetadata = adminClient.GetMetadata(topic, TimeSpan.FromSeconds(10));
        var partitions = topicMetadata.Topics.First(t => t.Topic == topic).Partitions;

        var lag = new Dictionary<string, long>();

        foreach (var partition in partitions)
        {
            var tp = new TopicPartition(topic, partition.PartitionId);
            // Get log end offset
            // Get consumer committed offset
            // Lag = end - committed
            // (simplified — in practice use watermarks)
        }

        return lag;
    }
}
```

---

## 8. Consumer Configuration Quick Reference

```csharp
var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    GroupId = "my-service",

    // Offset behavior
    AutoOffsetReset = AutoOffsetReset.Earliest, // or Latest
    EnableAutoCommit = false,                    // ALWAYS false for reliability

    // Heartbeat & session
    SessionTimeoutMs = 30000,      // 30 seconds
    HeartbeatIntervalMs = 10000,   // Every 10 seconds (< 1/3 of session timeout)
    MaxPollIntervalMs = 300000,    // 5 minutes max to process before "dead"

    // Fetch tuning
    FetchMinBytes = 1,             // Return immediately if any data available
    FetchWaitMaxMs = 500,          // Wait up to 500ms for FetchMinBytes
    MaxPartitionFetchBytes = 1048576, // 1 MB per partition per fetch

    // Partition assignment
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.CooperativeSticky,

    // Reconnect
    ReconnectBackoffMs = 100,
    ReconnectBackoffMaxMs = 10000,
};
```

---

## 9. Consumer Group Pitfalls

### Pitfall 1: MaxPollIntervalMs Exceeded

```
Symptom: Consumer gets kicked out of group with:
"Application maximum poll interval (300000ms) exceeded by Xms"

Cause: Processing takes too long between Consume() calls.
Kafka thinks the consumer is dead → triggers rebalance.

Fix options:
1. Increase MaxPollIntervalMs
2. Process faster (async, parallel)
3. Move heavy work to a background thread and poll frequently
```

```csharp
// Anti-pattern: heavy processing blocks the poll loop
while (true)
{
    var result = consumer.Consume(token);
    await HeavyProcessingThatTakes10Minutes(result); // ❌ MaxPollInterval exceeded!
}

// Better: Use a Channel to separate consumption from processing
// (see Channel-based pattern in section 4)
```

### Pitfall 2: Same GroupId Across Environments

```csharp
// ❌ WRONG: dev and prod share the same consumer group!
GroupId = "order-service"

// ✅ CORRECT: include environment
GroupId = $"order-service-{environment}" // "order-service-dev", "order-service-prod"
// Or set in appsettings.json per environment
```

### Pitfall 3: Committing Before Processing

```csharp
// ❌ WRONG: commit before processing — message lost if crash occurs
consumer.Commit(result);
await ProcessAsync(result); // Crashes here → message skipped!

// ✅ CORRECT: commit AFTER successful processing
await ProcessAsync(result);
consumer.Commit(result);
```

---

## Summary

You now know:
- ✅ How Kafka assigns partitions to consumer group members
- ✅ Partition assignment strategies (use CooperativeSticky!)
- ✅ How to handle rebalances (commit on revoke)
- ✅ Concurrent consumer patterns (Channel-based)
- ✅ Batch consumption for bulk processing
- ✅ Seeking to replay messages
- ✅ Consumer configuration and common pitfalls

**Next:** [06-TOPICS-PARTITIONS-OFFSETS.md](./06-TOPICS-PARTITIONS-OFFSETS.md) — Deep dive into topics, partitions, and offsets
