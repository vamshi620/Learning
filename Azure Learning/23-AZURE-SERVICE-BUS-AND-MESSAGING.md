# Azure Service Bus & Messaging Patterns
## Document 23: Enterprise Messaging on Azure

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Service Bus queues, topics, sessions, dead-letter, Event Grid, Event Hub comparison

---

## TABLE OF CONTENTS

1. [Messaging Overview & When to Use What](#overview)
2. [Azure Service Bus Fundamentals](#fundamentals)
3. [Queues — Point-to-Point Messaging](#queues)
4. [Topics & Subscriptions — Pub/Sub Messaging](#topics)
5. [Advanced Features](#advanced)
6. [Event Grid](#event-grid)
7. [Event Hub](#event-hub)
8. [Messaging Decision Guide](#decision-guide)
9. [Code Examples](#code-examples)
10. [Best Practices](#best-practices)

---

## Messaging Overview & When to Use What {#overview}

```
┌──────────────────────────────────────────────────────────────────┐
│                    AZURE MESSAGING SERVICES                      │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │  Storage Queue    │  │ Service Bus  │  │   Event Grid    │   │
│  │  Simple FIFO      │  │ Enterprise   │  │  Event Routing  │   │
│  │  64 KB messages   │  │ Messaging    │  │  Push-based     │   │
│  │  7-day retention  │  │ 256 KB-100MB │  │  Reactive       │   │
│  └──────────────────┘  └──────────────┘  └─────────────────┘   │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────┐                         │
│  │  Event Hub        │  │ Azure Queue  │                         │
│  │  Stream Ingestion │  │ (Storage)    │                         │
│  │  Millions/sec     │  │ Simple tasks │                         │
│  │  Kafka-compatible │  │              │                         │
│  └──────────────────┘  └──────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

| Service | Pattern | Message Size | Retention | Best For |
|---------|---------|-------------|-----------|----------|
| **Storage Queue** | Point-to-point | 64 KB | 7 days | Simple task queuing |
| **Service Bus Queue** | Point-to-point | 256 KB–100 MB | 14 days+ | Enterprise messaging, ordering, transactions |
| **Service Bus Topic** | Pub/Sub | 256 KB–100 MB | 14 days+ | Broadcast to multiple subscribers |
| **Event Grid** | Event routing | 1 MB | None (push) | Reactive events (blob created, resource changed) |
| **Event Hub** | Event streaming | 1 MB | 1–90 days | Telemetry ingestion, log streaming, Kafka |

---

## Azure Service Bus Fundamentals {#fundamentals}

### What is Service Bus?

Azure Service Bus is a fully managed **enterprise message broker** with message queues and publish-subscribe topics. It provides:

- **Guaranteed FIFO ordering** (with sessions)
- **At-least-once / at-most-once delivery**
- **Transactions** (atomic operations across messages)
- **Dead-letter queue** (DLQ) for failed messages
- **Scheduled delivery** and message deferral
- **Duplicate detection**
- **Auto-forwarding** between queues/topics

### Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Queues | ✅ | ✅ | ✅ |
| Topics & Subscriptions | ❌ | ✅ | ✅ |
| Sessions | ❌ | ✅ | ✅ |
| Transactions | ❌ | ✅ | ✅ |
| Message size | 256 KB | 256 KB | 100 MB |
| Throughput | Shared | Shared | Dedicated (1–16 MU) |
| VNet integration | ❌ | ❌ | ✅ |
| Price | ~$0.05/M ops | ~$10/month | ~$668/month/MU |

### Create Service Bus

```bash
# Create namespace
az servicebus namespace create \
  --resource-group myRG \
  --name myservicebus \
  --location eastus \
  --sku Standard

# Get connection string
az servicebus namespace authorization-rule keys list \
  --resource-group myRG \
  --namespace-name myservicebus \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv
```

---

## Queues — Point-to-Point Messaging {#queues}

```
Producer(s) ──► [ Queue ] ──► Consumer(s)

- Each message consumed by EXACTLY one consumer
- Messages stored until consumed or expired
- Supports competing consumers (multiple consumers share load)
```

```bash
# Create queue
az servicebus queue create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --name orders \
  --max-size 1024 \
  --default-message-time-to-live P14D \
  --lock-duration PT1M \
  --enable-dead-lettering-on-message-expiration true \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M \
  --max-delivery-count 10
```

### Key Queue Properties

| Property | Purpose | Default |
|----------|---------|---------|
| `lock-duration` | Time consumer has to process before lock expires | 30 seconds |
| `max-delivery-count` | Attempts before dead-lettering | 10 |
| `default-message-time-to-live` | Auto-expire unprocessed messages | 14 days |
| `enable-duplicate-detection` | Reject duplicate messages by MessageId | false |
| `enable-dead-lettering-on-expiration` | Dead-letter expired messages | false |
| `enable-sessions` | Enable ordered, grouped message processing | false |

---

## Topics & Subscriptions — Pub/Sub Messaging {#topics}

```
Publisher ──► [ Topic ]
               ├──► [ Subscription A ] ──► Consumer A  (filter: type='order')
               ├──► [ Subscription B ] ──► Consumer B  (filter: type='payment')
               └──► [ Subscription C ] ──► Consumer C  (all messages)

- Each message can be consumed by MULTIPLE subscribers
- Subscriptions can have SQL-like filters
- Each subscription acts like its own queue
```

```bash
# Create topic
az servicebus topic create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --name events \
  --max-size 1024

# Create subscriptions with filters
az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --topic-name events \
  --name order-processor

az servicebus topic subscription rule create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --topic-name events \
  --subscription-name order-processor \
  --name OrderFilter \
  --filter-sql-expression "EventType = 'OrderCreated'"

az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --topic-name events \
  --name payment-processor

az servicebus topic subscription rule create \
  --resource-group myRG \
  --namespace-name myservicebus \
  --topic-name events \
  --subscription-name payment-processor \
  --name PaymentFilter \
  --filter-sql-expression "EventType = 'PaymentReceived'"
```

---

## Advanced Features {#advanced}

### Dead-Letter Queue (DLQ)

Messages that can't be processed (max delivery count exceeded, expired, filter evaluation exceptions) are moved to the DLQ.

```
Queue / Subscription
├── Active Messages (normal processing)
└── Dead-Letter Queue ($DeadLetterQueue)
    ├── Message 1: MaxDeliveryCountExceeded
    ├── Message 2: TTLExpired
    └── Message 3: FilterEvaluationException
```

### Sessions (Ordered Processing)

Sessions guarantee FIFO ordering for messages with the same `SessionId`. Only one consumer can hold a session lock at a time.

```
Use case: Processing messages for a specific order in sequence

Message 1: { SessionId: "order-123", Step: "created" }
Message 2: { SessionId: "order-123", Step: "paid" }
Message 3: { SessionId: "order-123", Step: "shipped" }

→ All three processed in order by the SAME consumer
```

### Scheduled Messages

```csharp
// Schedule a message for 1 hour from now
var message = new ServiceBusMessage("Reminder: Your trial expires today!");
long sequenceNumber = await sender.ScheduleMessageAsync(
    message,
    DateTimeOffset.UtcNow.AddHours(1)
);

// Cancel a scheduled message
await sender.CancelScheduledMessageAsync(sequenceNumber);
```

### Transactions

```csharp
// Atomic: send to queue A AND complete from queue B — all or nothing
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await senderA.SendMessageAsync(new ServiceBusMessage("Forward to A"));
await receiverB.CompleteMessageAsync(receivedMessage);
scope.Complete();
```

---

## Event Grid {#event-grid}

Event Grid is a **fully managed event routing service** that uses the publish-subscribe model. It's **push-based** — events are delivered to subscribers within seconds.

```
Event Sources              Event Grid              Event Handlers
─────────────              ──────────              ──────────────
Blob Storage    ──►  ┌──────────────────┐  ──►  Azure Functions
Resource Groups ──►  │   Event Grid     │  ──►  Logic Apps
Custom Apps     ──►  │   (Topics +      │  ──►  Webhooks
IoT Hub         ──►  │    Subscriptions) │  ──►  Event Hub
Azure AD        ──►  └──────────────────┘  ──►  Service Bus
Key Vault       ──►                        ──►  Storage Queue
```

### System Topics (Built-in Events)

```bash
# Subscribe to blob creation events → trigger Azure Function
az eventgrid event-subscription create \
  --name blob-created-sub \
  --source-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mysa \
  --endpoint /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myfunc/functions/ProcessBlob \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/
```

### Custom Topics

```bash
# Create custom topic
az eventgrid topic create \
  --name my-app-events \
  --resource-group myRG \
  --location eastus

# Publish event (REST)
TOPIC_ENDPOINT=$(az eventgrid topic show --name my-app-events -g myRG --query "endpoint" -o tsv)
TOPIC_KEY=$(az eventgrid topic key list --name my-app-events -g myRG --query "key1" -o tsv)

curl -X POST $TOPIC_ENDPOINT \
  -H "aeg-sas-key: $TOPIC_KEY" \
  -H "Content-Type: application/json" \
  -d '[{
    "id": "1001",
    "eventType": "OrderCreated",
    "subject": "orders/1001",
    "data": { "orderId": 1001, "total": 99.99 },
    "dataVersion": "1.0"
  }]'
```

---

## Event Hub {#event-hub}

Event Hub is a **big data streaming platform** capable of ingesting **millions of events per second**. It's Apache Kafka-compatible.

```
Producers ──► [ Event Hub (partitions) ] ──► Consumer Groups
                 ├── Partition 0              ├── Consumer Group 1 (real-time analytics)
                 ├── Partition 1              └── Consumer Group 2 (batch to ADLS)
                 ├── Partition 2
                 └── Partition 3
```

```bash
# Create Event Hub namespace + hub
az eventhubs namespace create \
  --name myeventhub-ns \
  --resource-group myRG \
  --sku Standard \
  --location eastus

az eventhubs eventhub create \
  --name telemetry \
  --namespace-name myeventhub-ns \
  --resource-group myRG \
  --partition-count 4 \
  --message-retention 7
```

| Feature | Event Hub | Kafka |
|---------|-----------|-------|
| Partitions | ✅ | ✅ |
| Consumer groups | ✅ | ✅ |
| Offset-based reads | ✅ | ✅ |
| API compatible | ✅ (Kafka protocol) | Native |
| Managed | Fully | Self-managed or Confluent |
| Capture to ADLS/Blob | ✅ Built-in | ❌ (need Kafka Connect) |

---

## Messaging Decision Guide {#decision-guide}

```
Need simple task queue?
  └── Azure Storage Queue

Need enterprise features (ordering, transactions, DLQ, sessions)?
  └── Azure Service Bus Queue

Need broadcast to multiple subscribers?
  └── Azure Service Bus Topic

Need reactive event routing (blob created, resource changed)?
  └── Azure Event Grid

Need high-throughput telemetry / log streaming?
  └── Azure Event Hub

Need Kafka compatibility?
  └── Azure Event Hub (with Kafka endpoint)
```

---

## Code Examples {#code-examples}

### C# — Send and Receive (Service Bus Queue)

```csharp
using Azure.Messaging.ServiceBus;
using Azure.Identity;

// Send
await using var client = new ServiceBusClient(
    "myservicebus.servicebus.windows.net",
    new DefaultAzureCredential());

var sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage("Order #1001 created"));

// Send batch
using var batch = await sender.CreateMessageBatchAsync();
batch.TryAddMessage(new ServiceBusMessage("Order #1002"));
batch.TryAddMessage(new ServiceBusMessage("Order #1003"));
await sender.SendMessageBatchAsync(batch);

// Receive
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{
    AutoCompleteMessages = false,
    MaxConcurrentCalls = 5
});

processor.ProcessMessageAsync += async args =>
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body}");
    await args.CompleteMessageAsync(args.Message);
};

processor.ProcessErrorAsync += args =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
    return Task.CompletedTask;
};

await processor.StartProcessingAsync();
```

### Python — Send and Receive

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
client = ServiceBusClient("myservicebus.servicebus.windows.net", credential)

# Send
with client.get_queue_sender("orders") as sender:
    sender.send_messages(ServiceBusMessage("Order created"))

# Receive
with client.get_queue_receiver("orders", max_wait_time=30) as receiver:
    for msg in receiver:
        print(f"Received: {str(msg)}")
        receiver.complete_message(msg)
```

---

## Best Practices {#best-practices}

```
✅ Design Patterns
├─ Use competing consumers for horizontal scaling
├─ Use sessions for ordered processing per entity
├─ Use topics for fan-out (one event → many handlers)
├─ Use dead-letter queues to capture failed messages
├─ Implement idempotent handlers (messages may be delivered twice)
└─ Use message deduplication for exactly-once semantics

✅ Reliability
├─ Set max-delivery-count to a reasonable value (5-10)
├─ Monitor DLQ depth with Azure Monitor alerts
├─ Implement retry with exponential backoff
├─ Use PeekLock mode (not ReceiveAndDelete) for production
└─ Set message TTL to prevent infinite queue growth

✅ Security
├─ Use Managed Identity (DefaultAzureCredential), not connection strings
├─ Use Shared Access Policies with least-privilege (Send, Listen, Manage)
├─ Enable Private Endpoints for Premium tier
└─ Encrypt messages at application level for sensitive data

✅ Performance
├─ Batch sends (CreateMessageBatch) for throughput
├─ Set MaxConcurrentCalls for parallel processing
├─ Use Premium tier with Messaging Units for predictable latency
├─ Enable auto-forwarding to chain queues without extra hops
└─ Use partitioned queues/topics for higher throughput (Standard tier)
```
