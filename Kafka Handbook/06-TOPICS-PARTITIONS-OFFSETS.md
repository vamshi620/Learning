# Topics, Partitions & Offsets — Deep Dive
## File 06: Designing Your Topic Architecture

---

## What You'll Learn

- How to decide partition counts for topics
- Topic naming conventions and governance
- Offset management strategies
- Log compaction for state topics
- Topic configuration tuning
- Topic design patterns for .NET microservices

---

## 1. Partition Count — How to Decide

Getting partition count right from the start is important — you can increase later, but **never decrease**.

### Formula for Partition Count

```
Partitions = max(
    Target throughput / Throughput per partition,
    Number of consumer instances you'll run
)
```

### Estimating Throughput per Partition

On typical cloud hardware, each partition handles:
- **Write:** ~50–100 MB/s
- **Read:** ~50–100 MB/s (more if multiple consumers)

For message-count thinking:
- ~10K-100K messages/sec per partition (depending on message size and config)

### Practical Examples

```
Scenario 1: Order events
- Expected throughput: 1,000 orders/min
- Max consumers you'd run: 6 instances
- Recommendation: 6 partitions

Scenario 2: Audit log events
- Expected throughput: 100,000 events/min
- Max consumers: 12 instances
- Recommendation: 12 partitions

Scenario 3: IoT sensor data
- Expected throughput: 10M events/min
- Max consumers: 24 instances
- Recommendation: 24+ partitions
```

### Rule of Thumb for Small Teams

```
Low volume (< 10K msg/day):     3 partitions   (enough for dev/test)
Medium volume (< 1M msg/day):   6 partitions   (good for most microservices)
High volume (> 10M msg/day):    12+ partitions (tune based on consumers)
```

> ⚠️ **Common Mistake:** Don't over-partition. More partitions = more file handles, more memory, more rebalancing overhead. Start small and increase when needed.

---

## 2. Topic Naming Conventions

Agree on a naming convention **before** creating any topics. Changing topic names later is painful.

### Recommended Format

```
{domain}.{entity}.{event-type}
{domain}.{entity}.{event-type}.{version}      # For versioned topics
{domain}.{entity}.{event-type}.dlq             # Dead letter queue
```

### Examples

```
# E-commerce domain
ecommerce.orders.placed
ecommerce.orders.confirmed
ecommerce.orders.shipped
ecommerce.orders.cancelled
ecommerce.orders.placed.dlq          # DLQ for failed processing

ecommerce.payments.initiated
ecommerce.payments.completed
ecommerce.payments.failed

ecommerce.inventory.reserved
ecommerce.inventory.released
ecommerce.inventory.restocked

# Internal/infrastructure topics
_kafka.consumer.offsets               # Built-in (don't touch)
_kafka.transaction.state              # Built-in (don't touch)

# Versioning (when you break the schema contract)
ecommerce.orders.placed.v2            # New version alongside old
```

### Naming Anti-Patterns

```
❌ orders                  # Too generic — which system, what event?
❌ OrderPlacedTopic        # CamelCase — dots are standard
❌ order_placed            # Underscores work but inconsistent with dots
❌ prod-orders-placed      # Don't include environment in topic name (use separate clusters/namespaces)
❌ order-service-events    # Service-based naming — topics outlive services
```

---

## 3. Topic Configuration Reference

```powershell
# Create a well-configured production topic
# Configuration explanations:
# retention.ms=604800000         (7 days)
# min.insync.replicas=2          (At least 2 replicas must acknowledge)
# max.message.bytes=1048576      (1 MB max message size)
# cleanup.policy=delete          (Delete old messages vs compact)
# compression.type=producer      (Use producer's compression setting)
docker exec kafka kafka-topics.sh `
  --bootstrap-server localhost:9092 `
  --create `
  --topic ecommerce.orders.placed `
  --partitions 6 `
  --replication-factor 3 `
  --config retention.ms=604800000 `
  --config min.insync.replicas=2 `
  --config max.message.bytes=1048576 `
  --config cleanup.policy=delete `
  --config compression.type=producer
```

### Important Topic Configurations

| Config | Description | Recommended |
|--------|-------------|-------------|
| `retention.ms` | How long to keep messages | 7 days (604800000) |
| `retention.bytes` | Max bytes per partition | -1 (unlimited, use time-based) |
| `min.insync.replicas` | Min replicas for write to succeed | 2 (for RF=3) |
| `max.message.bytes` | Max single message size | 1-10 MB |
| `cleanup.policy` | `delete` or `compact` | `delete` for events |
| `compression.type` | Broker-side compression | `producer` (use what producer sends) |
| `segment.bytes` | Size per log segment file | 1GB (default) |
| `segment.ms` | Roll to new segment after this time | 7 days |

---

## 4. Log Compaction (State Topics)

Normal topics delete old messages after retention expires.
**Compacted topics** keep only the LATEST message per key — great for storing current state.

```
Regular topic (delete policy):
Time 1: Key="user-123" Value={"name":"John","email":"john@old.com"}  ← deleted after 7 days
Time 2: Key="user-123" Value={"name":"John","email":"john@new.com"}  ← kept
Time 3: Key="user-456" Value={"name":"Alice","email":"alice@example.com"} ← kept

Compacted topic:
(After compaction)
Key="user-123" → {"name":"John","email":"john@new.com"}   ← only latest!
Key="user-456" → {"name":"Alice","email":"alice@example.com"}

Use cases:
- User profiles (latest profile per user ID)
- Configuration/settings (latest setting value per key)
- Account balances (latest balance per account)
- Inventory counts (latest count per product)
```

```powershell
# Create a compacted topic
# min.cleanable.dirty.ratio=0.1: Compact when 10% of log is "dirty"
# segment.ms=3600000: New segment every hour (triggers compaction)
docker exec kafka kafka-topics.sh `
  --bootstrap-server localhost:9092 `
  --create `
  --topic user.profiles `
  --partitions 6 `
  --replication-factor 1 `
  --config cleanup.policy=compact `
  --config min.cleanable.dirty.ratio=0.1 `
  --config segment.ms=3600000

# Delete a key (tombstone): publish message with null value
# Kafka will keep the null-value record briefly, then remove the key entirely
await producer.ProduceAsync("user.profiles", new Message<string, string>
{
    Key = "user-123",
    Value = null  // Tombstone — deletes this key after compaction
});
```

```csharp
// Reading a compacted topic to rebuild state (like a local cache)
public async Task<Dictionary<string, UserProfile>> LoadCurrentStateAsync()
{
    var state = new Dictionary<string, UserProfile>();

    using var consumer = new ConsumerBuilder<string, string>(config).Build();
    var tp = new TopicPartition("user.profiles", 0); // Read each partition
    consumer.Assign(tp);
    consumer.Seek(new TopicPartitionOffset(tp, Offset.Beginning));

    var watermarks = consumer.GetWatermarkOffsets(tp);

    while (true)
    {
        var result = consumer.Consume(TimeSpan.FromSeconds(1));

        if (result == null) break; // No more messages right now

        if (result.Message.Value == null)
            state.Remove(result.Message.Key); // Tombstone — remove from state
        else
            state[result.Message.Key] = JsonSerializer.Deserialize<UserProfile>(result.Message.Value)!;

        if (result.Offset >= watermarks.High - 1) break; // Reached end
    }

    return state;
}
```

---

## 5. Offset Management Strategies

### Strategy 1: Auto Commit (Avoid in Production)

```csharp
EnableAutoCommit = true,
AutoCommitIntervalMs = 5000 // Commits every 5s regardless of processing status
// Risk: message processed = false but committed = true → data loss on crash
```

### Strategy 2: Synchronous Manual Commit (Safe, Lower Throughput)

```csharp
EnableAutoCommit = false,
// After each message:
consumer.Commit(result); // Blocks until broker acknowledges commit
```

### Strategy 3: Async Batch Commit (Best Throughput + Safety)

```csharp
EnableAutoCommit = false,
EnableAutoOffsetStore = false, // We control when offsets are stored

// Process messages, store offset after each
consumer.StoreOffset(result); // Non-blocking — stores locally

// Commit stored offsets periodically (every N messages or every T seconds)
if (messageCount % 100 == 0 || DateTime.UtcNow - lastCommit > TimeSpan.FromSeconds(5))
{
    consumer.Commit(); // Commit all stored offsets
    lastCommit = DateTime.UtcNow;
}
```

### Strategy 4: External Offset Storage

Store offsets in your own database, not Kafka. Allows atomic DB write + offset advance:

```csharp
// Read committed offset from your DB
var lastOffset = await _db.KafkaOffsets
    .Where(o => o.GroupId == "my-group" && o.Topic == "orders" && o.Partition == 0)
    .Select(o => o.Offset)
    .FirstOrDefaultAsync();

// Seek to that offset
consumer.Assign(new TopicPartitionOffset("orders", 0, lastOffset + 1));

// After processing, save to your DB (same transaction as your business write)
await _db.KafkaOffsets.UpsertAsync(new KafkaOffset
{
    GroupId = "my-group", Topic = "orders", Partition = 0,
    Offset = result.Offset.Value
});
// Don't call consumer.Commit() — you manage offsets yourself
```

---

## 6. Topic Design Patterns

### Pattern 1: One Topic Per Event Type (Recommended)

```
ecommerce.orders.placed
ecommerce.orders.confirmed
ecommerce.orders.cancelled
```
**Pros:** Clean separation, easy schema evolution, clear consumer responsibilities  
**Cons:** More topics to manage

### Pattern 2: One Topic Per Aggregate (Event Stream)

```
ecommerce.orders          ← all order events go here, filtered by event type header
ecommerce.payments
ecommerce.inventory
```
**Pros:** Fewer topics, easier ordering across event types for one aggregate  
**Cons:** Consumers must filter; schema management harder

### Pattern 3: Fanout Pattern (Topic Per Consumer)

```
# Producer writes to one canonical topic
ecommerce.orders.placed

# A "topic fanout service" copies to consumer-specific topics:
ecommerce.orders.placed.for-inventory
ecommerce.orders.placed.for-payment
ecommerce.orders.placed.for-email

# Each consumer reads from its own topic
```
**Pros:** Consumer-specific partitioning and retention  
**Cons:** Duplication, operational overhead

> 💡 **Pro Tip for .NET teams:** Start with **Pattern 1** (one topic per event type). It's the most intuitive and scales well.

---

## 7. Topics Governance Checklist

Before creating a new topic, ask:

```
□ Does a similar topic already exist?
□ What's the expected message volume (msgs/day)?
□ How many consumer groups will read this?
□ What retention is needed (how long to keep messages)?
□ Should it be compacted (state) or deleted (events)?
□ What's the key strategy (what field is the key)?
□ Is the schema documented in Schema Registry?
□ Who owns this topic? Which team?
□ Is there a DLQ configured?
□ Is consumer lag monitoring set up?
```

---

## Summary

You now know:
- ✅ How to calculate partition counts for your workload
- ✅ Topic naming conventions for large teams
- ✅ All key topic configuration settings
- ✅ Log compaction for state topics (vs delete policy for event topics)
- ✅ Offset management strategies (auto vs manual vs batch vs external)
- ✅ Topic design patterns for microservices

**Next:** [07-SERIALIZATION.md](./07-SERIALIZATION.md) — JSON, Avro, and Protobuf serialization
