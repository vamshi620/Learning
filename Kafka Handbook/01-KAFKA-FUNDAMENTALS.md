# Kafka Fundamentals
## File 01: What is Kafka? Core Concepts & Architecture

---

## What You'll Learn

- What Apache Kafka is and why it was created
- Core concepts: Topics, Partitions, Brokers, Producers, Consumers
- Kafka's architecture and how data flows
- When to use Kafka vs. other messaging systems
- Key terminology you'll use daily

**Time Required:** 45-60 minutes (reading + diagrams)

---

## 1. What is Apache Kafka?

Apache Kafka is a **distributed event streaming platform** originally built at LinkedIn in 2011 and open-sourced shortly after. It is now maintained by the Apache Software Foundation and used by companies like Netflix, Uber, Airbnb, and Microsoft.

### The Problem Kafka Solves

Imagine your company has 10 microservices. Each needs to communicate with others:

```
Without Kafka (Point-to-Point):
OrderService ──→ InventoryService
OrderService ──→ PaymentService
OrderService ──→ EmailService
OrderService ──→ AnalyticsService
OrderService ──→ ShippingService

10 services × 9 = up to 90 direct connections!
```

This becomes a **spaghetti architecture**. Adding one new service means updating everyone.

```
With Kafka (Event-Driven):
OrderService ──→ [orders topic] ──→ InventoryService
                               ──→ PaymentService
                               ──→ EmailService
                               ──→ AnalyticsService
                               ──→ ShippingService

Each service only talks to Kafka. Zero direct coupling.
```

### What Kafka Provides
- **High throughput** — Millions of messages per second
- **Durability** — Messages written to disk, replicated across brokers
- **Scalability** — Horizontally scalable, no single point of failure
- **Replay** — Consumers can re-read past messages
- **Decoupling** — Producers don't know or care who consumes their messages

---

## 2. Core Concepts

### 📌 2.1 Event (Message / Record)

An **event** is the fundamental unit of data in Kafka. Every piece of data sent to Kafka is an event.

An event has:
```
┌──────────────────────────────────────────┐
│ Key      : "order-12345"                 │
│ Value    : { "orderId": "12345",          │
│               "amount": 299.99,           │
│               "status": "placed" }        │
│ Timestamp: 2024-01-15T10:30:00Z           │
│ Headers  : { "source": "web-app" }        │
│ Offset   : 42                             │
│ Partition: 2                              │
└──────────────────────────────────────────┘
```

| Field | Description | Required? |
|-------|-------------|-----------|
| **Key** | Used for partitioning and ordering. Can be null. | No |
| **Value** | The actual data (JSON, Avro, bytes, etc.) | Yes |
| **Timestamp** | When the event occurred | Auto-assigned |
| **Headers** | Metadata key-value pairs | No |
| **Offset** | Sequential ID within a partition (assigned by Kafka) | Auto-assigned |

> 📌 **Key Concept:** The **key** is NOT a unique identifier. It controls which partition the message goes to. All messages with the same key go to the same partition — guaranteeing order for that key.

---

### 📌 2.2 Topic

A **topic** is a named category or feed where messages are published. Think of it like a table in a database, or a folder for related events.

```
┌─────────────────────────────┐
│      Topic: "orders"        │
│                             │
│  order.placed               │
│  order.paid                 │
│  order.shipped              │
│  order.delivered            │
│  order.cancelled            │
└─────────────────────────────┘
```

**Topic naming conventions** (use what your team agrees on):
```
# Domain-based
orders.placed
payments.processed
inventory.updated

# Service-based
order-service.events
payment-service.events

# Mixed approach (recommended for large teams)
{domain}.{entity}.{event-type}
ecommerce.orders.placed
ecommerce.payments.processed
```

> ⚠️ **Common Mistake:** Don't create one topic for everything. Design topics around business domains, not services. Services come and go; domains don't.

---

### 📌 2.3 Partition

A **partition** is an ordered, immutable sequence of messages within a topic. Each topic is split into one or more partitions.

```
Topic: "orders" (3 partitions)

Partition 0: [msg0] [msg3] [msg6] [msg9] ...
Partition 1: [msg1] [msg4] [msg7] [msg10] ...
Partition 2: [msg2] [msg5] [msg8] [msg11] ...
                                    ↑
                                 Offset (sequential position)
```

**Why partitions?**
- **Parallelism** — Different consumers can read different partitions simultaneously
- **Scalability** — More partitions = higher throughput
- **Ordering** — Order is guaranteed WITHIN a partition only

**How Kafka decides which partition to use:**
```
If message has a KEY:   partition = hash(key) % numPartitions
If no key:              Round-robin across partitions
If custom partitioner:  Your logic
```

> 📌 **Key Concept:** You can **increase** partitions but **cannot decrease** them. Plan your partition count carefully.

**Partition Count Guidelines:**
| Throughput | Partitions |
|------------|-----------|
| Low (< 10K msg/sec) | 3-6 |
| Medium (< 100K msg/sec) | 12-24 |
| High (> 100K msg/sec) | 48-96+ |

---

### 📌 2.4 Offset

An **offset** is a sequential integer that uniquely identifies each message within a partition. Kafka assigns offsets automatically, starting from 0.

```
Partition 0:
  Offset 0: {"orderId": "001", "amount": 100}
  Offset 1: {"orderId": "002", "amount": 250}
  Offset 2: {"orderId": "003", "amount": 75}
  Offset 3: {"orderId": "004", "amount": 500}
           ↑ new messages go here
```

**Why offsets matter:**
- Consumers track which offset they've processed
- If a consumer crashes, it resumes from the last committed offset
- You can replay messages by resetting to an earlier offset
- Different consumer groups maintain their own offsets independently

---

### 📌 2.5 Broker

A **broker** is a Kafka server. In production, you run multiple brokers forming a **cluster**. Each broker:
- Stores partition data on disk
- Handles read/write requests
- Replicates data to other brokers

```
Kafka Cluster (3 brokers):

┌───────────┐   ┌───────────┐   ┌───────────┐
│ Broker 1  │   │ Broker 2  │   │ Broker 3  │
│           │   │           │   │           │
│ P0 (lead) │   │ P0 (rep)  │   │ P0 (rep)  │
│ P1 (rep)  │   │ P1 (lead) │   │ P1 (rep)  │
│ P2 (rep)  │   │ P2 (rep)  │   │ P2 (lead) │
└───────────┘   └───────────┘   └───────────┘
    9092            9093            9094
```

**Leader and Replica:**
- Each partition has one **Leader** broker (handles reads/writes)
- Other brokers have **Replicas** (backups)
- If a leader fails, Kafka automatically promotes a replica

---

### 📌 2.6 Replication Factor

**Replication factor** determines how many copies of each partition exist across brokers.

```yaml
# 3 brokers, replication factor = 3
# Every partition has 3 copies — one on each broker
# Can tolerate 2 broker failures and still work

replication-factor: 3  # Recommended for production
```

| Replication Factor | Brokers Needed | Fault Tolerance |
|-------------------|---------------|-----------------|
| 1 | 1 | None (data loss if broker dies) |
| 2 | 2 | 1 broker failure |
| 3 | 3 | 2 broker failures |

> ⚠️ **Common Mistake:** Development: replication-factor=1 is fine. **Never use 1 in production.**

---

### 📌 2.7 Producer

A **producer** is any application that **writes** (publishes) messages to a Kafka topic.

```csharp
// .NET Producer example (simplified)
var producer = new ProducerBuilder<string, string>(config).Build();
await producer.ProduceAsync("orders", new Message<string, string> {
    Key = "order-123",
    Value = JsonSerializer.Serialize(order)
});
```

**Producers handle:**
- Choosing which topic to write to
- Choosing (or letting Kafka choose) which partition
- Retry on failure
- Batching for performance
- Compression

---

### 📌 2.8 Consumer

A **consumer** is any application that **reads** (subscribes to) messages from a Kafka topic.

```csharp
// .NET Consumer example (simplified)
var consumer = new ConsumerBuilder<string, string>(config).Build();
consumer.Subscribe("orders");

while (true) {
    var message = consumer.Consume();
    ProcessOrder(message.Value);
    consumer.Commit(message);  // mark as processed
}
```

**Consumers handle:**
- Subscribing to topics
- Tracking their position (offset)
- Committing offsets after processing
- Joining consumer groups for parallel processing

---

### 📌 2.9 Consumer Group

A **consumer group** is a set of consumers that work together to consume a topic. Each partition is consumed by exactly one consumer in the group.

```
Topic: "orders" (4 partitions)
Consumer Group: "order-processing-service"

┌──────────┐  ┌──────────┐
│ Consumer │  │ Consumer │
│    A     │  │    B     │
│          │  │          │
│  P0, P1  │  │  P2, P3  │
└──────────┘  └──────────┘

Consumer A reads partitions 0 and 1.
Consumer B reads partitions 2 and 3.
They process in parallel!
```

**What happens when consumers join/leave:**
```
Start: 2 consumers → each reads 2 partitions
Add 3rd consumer: Kafka REBALANCES → each reads ~1-2 partitions  
Add 4th consumer: each reads 1 partition (maximum parallelism)
Add 5th consumer: 5th consumer is IDLE (more consumers than partitions!)
```

> 📌 **Key Concept:** **Max parallelism = number of partitions.** Adding consumers beyond the partition count wastes resources.

---

### 📌 2.10 ZooKeeper vs KRaft

Historically, Kafka needed **ZooKeeper** to manage cluster metadata, leader election, and configuration. 

Since **Kafka 2.8+ (KRaft mode)**, Kafka manages itself internally. ZooKeeper is being phased out.

```yaml
# Modern Kafka (KRaft - what we use in this handbook)
KAFKA_CFG_PROCESS_ROLES: broker,controller  # Kafka manages itself
# No ZooKeeper needed!
```

> 💡 **Pro Tip:** All examples in this handbook use **KRaft mode**. If you see ZooKeeper in older tutorials, that's legacy.

---

## 3. Kafka Architecture — Full Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KAFKA CLUSTER                                │
│                                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │    Broker 1     │  │    Broker 2     │  │    Broker 3     │      │
│  │                 │  │                 │  │                 │      │
│  │ orders-P0 (L)   │  │ orders-P0 (R)   │  │ orders-P0 (R)   │      │
│  │ orders-P1 (R)   │  │ orders-P1 (L)   │  │ orders-P1 (R)   │      │
│  │ orders-P2 (R)   │  │ orders-P2 (R)   │  │ orders-P2 (L)   │      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
          ↑ Write                                    ↓ Read
┌──────────────────┐                    ┌──────────────────────────────┐
│    PRODUCERS     │                    │         CONSUMERS            │
│                  │                    │                              │
│  OrderService    │                    │  Consumer Group A            │
│  (C# .NET)       │                    │   - InventoryService (C#)    │
│                  │                    │   - PaymentService (C#)      │
│  CheckoutService │                    │                              │
│  (C# .NET)       │                    │  Consumer Group B            │
│                  │                    │   - AnalyticsService (C#)    │
└──────────────────┘                    └──────────────────────────────┘
```

**(L) = Leader partition, (R) = Replica partition**

---

## 4. How Data Flows (Step by Step)

### Producing a Message

```
1. Producer creates message:
   Key: "order-123"
   Value: { orderId: "123", amount: 299.99 }

2. Producer determines partition:
   hash("order-123") % 3 = 1  →  Partition 1

3. Producer sends to Broker 2 (leader of Partition 1)

4. Broker 2 writes to its local log

5. Brokers 1 and 3 replicate from Broker 2

6. Broker 2 sends acknowledgment to Producer
   (depending on acks setting — more on this in File 04)
```

### Consuming a Message

```
1. Consumer in Group "order-service" subscribes to "orders"

2. Kafka assigns Partition 1 to this consumer

3. Consumer polls: "give me messages from offset 7 onwards"

4. Broker 2 returns messages at offset 7, 8, 9...

5. Consumer processes message at offset 7

6. Consumer commits offset 8 (next to read)
   "I've processed everything up to offset 7, next time start from 8"

7. If consumer crashes and restarts:
   It reads its committed offset (8) and continues from there
```

---

## 5. Kafka vs. Other Messaging Systems

| Feature | Kafka | RabbitMQ | Azure Service Bus | Azure Event Hubs |
|---------|-------|----------|------------------|-----------------|
| **Message Retention** | Days/Weeks (configurable) | Until consumed | Up to 14 days | Up to 7 days |
| **Throughput** | Millions/sec | Thousands/sec | Thousands/sec | Millions/sec |
| **Message Replay** | ✅ Yes | ❌ No | ❌ No | ✅ Yes |
| **Consumer Groups** | ✅ Yes | ❌ No (exchanges) | ✅ Yes | ✅ Yes |
| **Ordering** | Per partition | Per queue | Per session | Per partition |
| **Protocol** | Kafka Protocol | AMQP | AMQP/HTTPS | Kafka/AMQP/HTTPS |
| **Azure Managed** | Event Hubs | ❌ | ✅ | ✅ |
| **Best For** | Event streaming, logs, high throughput | Task queues, RPC | Enterprise messaging | Azure event streaming |

### When to Use Kafka
✅ High throughput (millions of events/day)  
✅ Multiple independent consumers of the same data  
✅ Need to replay/reprocess historical events  
✅ Event sourcing or CQRS patterns  
✅ Real-time analytics pipelines  
✅ Microservices event-driven communication  

### When NOT to Use Kafka
❌ Simple task queues (use RabbitMQ or Service Bus)  
❌ Request-reply patterns (use gRPC or HTTP)  
❌ You need message routing/filtering complexity (use RabbitMQ exchanges)  
❌ Very small scale (< 1K msg/day) — operational overhead not worth it  

---

## 6. Key Terminology Reference

| Term | Definition |
|------|-----------|
| **Event / Record / Message** | A single data point published to Kafka |
| **Topic** | Named category for related events |
| **Partition** | Ordered sub-division of a topic |
| **Offset** | Sequential ID of a message within a partition |
| **Broker** | A Kafka server node |
| **Cluster** | Multiple brokers working together |
| **Producer** | Application that writes to Kafka |
| **Consumer** | Application that reads from Kafka |
| **Consumer Group** | Set of consumers sharing a topic's partitions |
| **Replication Factor** | How many copies of each partition exist |
| **Leader** | The broker handling reads/writes for a partition |
| **Replica** | A backup copy of a partition |
| **ISR** | In-Sync Replicas — replicas caught up with leader |
| **Lag** | How far behind a consumer is from the latest message |
| **Retention** | How long Kafka keeps messages before deleting |
| **Commit** | Consumer telling Kafka "I've processed this offset" |
| **Rebalance** | Kafka redistributing partitions among consumers |
| **KRaft** | Kafka Raft — Kafka managing itself without ZooKeeper |

---

## 7. Kafka's Guarantees

### Delivery Guarantees (Producer Side)
| Setting | Guarantee | Risk |
|---------|-----------|------|
| `acks=0` | Fire and forget | May lose messages |
| `acks=1` | Leader acknowledges | Lose if leader dies before replication |
| `acks=all` | All ISR replicas acknowledge | No loss (use in production) |

### Consumer Delivery Semantics
| Semantic | Description | How |
|----------|-------------|-----|
| **At Most Once** | Messages may be lost, never duplicated | Commit before processing |
| **At Least Once** | Messages may be duplicated, never lost | Commit after processing |
| **Exactly Once** | No loss, no duplication | Kafka transactions (complex) |

> 💡 **Pro Tip:** Most .NET teams use **at-least-once** delivery with **idempotent consumers** (make processing the same message twice safe). Exactly-once is complex and usually overkill.

---

## 8. 🧪 Exercise: Design a Kafka Topology

Before moving to setup, practice designing for a real scenario:

**Scenario:** You're building an e-commerce platform with these services:
- OrderService (.NET API)
- InventoryService (.NET Worker)
- PaymentService (.NET Worker)
- EmailNotificationService (.NET Worker)
- AnalyticsService (.NET Worker)

**Design questions:**
1. What topics would you create?
2. How many partitions for each?
3. Which services are producers? Which are consumers?
4. Which services should be in the same consumer group?
5. Which should be in different groups?

**Sample Answer:**
```
Topics:
- ecommerce.orders.placed          (6 partitions)
- ecommerce.orders.confirmed       (6 partitions)
- ecommerce.payments.processed     (6 partitions)
- ecommerce.inventory.reserved     (6 partitions)
- ecommerce.notifications.email    (3 partitions)

Producers:
- OrderService → ecommerce.orders.placed
- PaymentService → ecommerce.payments.processed
- InventoryService → ecommerce.inventory.reserved

Consumers:
- InventoryService reads ecommerce.orders.placed  (group: inventory-svc)
- PaymentService reads ecommerce.orders.placed    (group: payment-svc)
- EmailService reads ecommerce.orders.placed      (group: email-svc)
- EmailService reads ecommerce.payments.processed (group: email-svc)
- AnalyticsService reads ALL topics               (group: analytics-svc)

Note: Each service is its own consumer group so they all get every message!
```

---

## Summary

You now understand:
- ✅ What Kafka is and what problem it solves
- ✅ Core concepts: Events, Topics, Partitions, Offsets, Brokers
- ✅ How producers and consumers work
- ✅ Consumer groups and parallel processing
- ✅ When to use Kafka vs. alternatives
- ✅ Kafka's delivery guarantees

**Next:** [02-KAFKA-SETUP.md](./02-KAFKA-SETUP.md) — Get Kafka running locally with Docker
