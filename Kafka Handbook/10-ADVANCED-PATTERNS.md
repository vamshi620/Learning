# Advanced Kafka Patterns
## File 10: CQRS, Saga & Outbox Patterns with Kafka in .NET

---

## What You'll Learn

- Event-Driven Architecture (EDA) fundamentals
- CQRS with Kafka as the event bus
- Saga pattern for distributed transactions (choreography & orchestration)
- Outbox pattern (reliable event publishing)
- Event sourcing overview with Kafka
- Domain events in .NET microservices

**Time Required:** 60 minutes

---

## 1. Event-Driven Architecture Recap

```
Traditional (Synchronous):
OrderController → [HTTP] → InventoryService → [HTTP] → PaymentService → [HTTP] → EmailService
Problems: tight coupling, cascading failures, blocking

Event-Driven (Kafka):
OrderController → saves order → publishes OrderPlaced event → returns 202 Accepted
                                     ↓
                        InventoryService consumes OrderPlaced → reserves items
                        PaymentService consumes OrderPlaced → charges customer
                        EmailService consumes OrderPlaced → sends confirmation

Benefits: loose coupling, independent scaling, failure isolation
```

---

## 2. Domain Events

Domain events represent something important that happened in your domain:

```csharp
// Base event record
public abstract record DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
    public int Version { get; init; } = 1;
}

// Concrete domain events
public record OrderPlaced : DomainEvent
{
    public required string OrderId { get; init; }
    public required string CustomerId { get; init; }
    public required decimal TotalAmount { get; init; }
    public required List<OrderLineItem> Items { get; init; }
    public required string ShippingAddress { get; init; }
}

public record OrderConfirmed : DomainEvent
{
    public required string OrderId { get; init; }
    public required string ConfirmedBy { get; init; }  // InventoryService
}

public record PaymentProcessed : DomainEvent
{
    public required string OrderId { get; init; }
    public required string TransactionId { get; init; }
    public required decimal AmountCharged { get; init; }
    public required bool Success { get; init; }
}

public record PaymentFailed : DomainEvent
{
    public required string OrderId { get; init; }
    public required string Reason { get; init; }
}

public record OrderShipped : DomainEvent
{
    public required string OrderId { get; init; }
    public required string TrackingNumber { get; init; }
    public required string Carrier { get; init; }
}
```

---

## 3. CQRS with Kafka

**CQRS** (Command Query Responsibility Segregation) separates write (Command) and read (Query) models.

```
Write Side (Command):
  API → Command → CommandHandler → Saves to Write DB → Publishes Event to Kafka

Read Side (Query):
  Event Consumer → Updates Read DB (denormalized view) → API serves from Read DB
```

### Command Handler (Write Side)

```csharp
// Commands/PlaceOrderCommand.cs
public record PlaceOrderCommand
{
    public required string CustomerId { get; init; }
    public required List<OrderLineItem> Items { get; init; }
    public required string ShippingAddress { get; init; }
}

// Handlers/PlaceOrderCommandHandler.cs
public class PlaceOrderCommandHandler
{
    private readonly IOrderRepository _orderRepo;
    private readonly IKafkaProducer<string, string> _producer;
    private readonly ILogger<PlaceOrderCommandHandler> _logger;

    public async Task<string> HandleAsync(PlaceOrderCommand command)
    {
        // 1. Create order aggregate
        var order = Order.Create(
            customerId: command.CustomerId,
            items: command.Items,
            shippingAddress: command.ShippingAddress);

        // 2. Validate business rules
        order.Validate(); // throws if invalid

        // 3. Save to write database
        await _orderRepo.SaveAsync(order);

        // 4. Publish domain event to Kafka
        var orderPlacedEvent = new OrderPlaced
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount,
            Items = order.Items,
            ShippingAddress = order.ShippingAddress,
        };

        await _producer.ProduceAsync("orders.placed", order.Id,
            JsonSerializer.Serialize(orderPlacedEvent));

        _logger.LogInformation("Order {OrderId} placed and event published", order.Id);
        return order.Id;
    }
}
```

### Read Model Updater (Read Side)

```csharp
// Consumers/OrderReadModelUpdater.cs
// Listens to all order events and maintains a denormalized read model

public class OrderReadModelUpdater : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildConsumer("order-read-model-updater");
        consumer.Subscribe(new[]
        {
            "orders.placed",
            "orders.confirmed",
            "payments.processed",
            "orders.shipped",
            "orders.delivered",
        });

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(stoppingToken);
            if (result?.Message == null) continue;

            // Determine event type from header or topic
            var topic = result.Topic;
            await UpdateReadModelAsync(topic, result.Message.Value);
            consumer.Commit(result);
        }
    }

    private async Task UpdateReadModelAsync(string topic, string eventJson)
    {
        // Route to appropriate handler based on topic
        switch (topic)
        {
            case "orders.placed":
                var placed = JsonSerializer.Deserialize<OrderPlaced>(eventJson)!;
                await _readDb.Orders.UpsertAsync(new OrderReadModel
                {
                    OrderId = placed.OrderId,
                    CustomerId = placed.CustomerId,
                    Status = "Placed",
                    TotalAmount = placed.TotalAmount,
                    PlacedAt = placed.OccurredAt,
                });
                break;

            case "payments.processed":
                var payment = JsonSerializer.Deserialize<PaymentProcessed>(eventJson)!;
                await _readDb.Orders.UpdateStatusAsync(
                    payment.OrderId,
                    payment.Success ? "Confirmed" : "PaymentFailed");
                break;

            case "orders.shipped":
                var shipped = JsonSerializer.Deserialize<OrderShipped>(eventJson)!;
                await _readDb.Orders.UpdateAsync(shipped.OrderId, o =>
                {
                    o.Status = "Shipped";
                    o.TrackingNumber = shipped.TrackingNumber;
                    o.Carrier = shipped.Carrier;
                });
                break;
        }
    }
}
```

### Query API (Read Side)

```csharp
// Controllers/OrderQueryController.cs
[ApiController]
[Route("api/orders")]
public class OrderQueryController : ControllerBase
{
    private readonly IOrderReadRepository _readRepo;

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrder(string orderId)
    {
        // Query from FAST read-optimized database (no joins needed!)
        var order = await _readRepo.GetByIdAsync(orderId);
        return order == null ? NotFound() : Ok(order);
    }

    [HttpGet("customer/{customerId}")]
    public async Task<IActionResult> GetCustomerOrders(string customerId)
    {
        var orders = await _readRepo.GetByCustomerAsync(customerId);
        return Ok(orders);
    }
}
```

---

## 4. Saga Pattern

Sagas manage distributed transactions across multiple services without 2-phase commit.

### 4.1 Choreography Saga (No Central Coordinator)

Each service reacts to events and publishes its own events:

```
OrderPlaced
    ↓
InventoryService reacts → reserves stock → publishes StockReserved
                                        OR publishes StockUnavailable
    ↓
PaymentService reacts to StockReserved → charges card → publishes PaymentProcessed
                                                      OR publishes PaymentFailed
    ↓
FulfillmentService reacts to PaymentProcessed → creates shipment → publishes OrderShipped
    ↓
EmailService reacts to OrderShipped → sends confirmation email

─── Compensation (if payment fails): ───
PaymentFailed
    ↓
InventoryService reacts → releases reservation → publishes StockReleased
OrderService reacts → marks order as Failed
```

```csharp
// InventoryService — Choreography Saga participant
public class InventoryEventHandler : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildConsumer("inventory-saga");
        consumer.Subscribe(new[] { "orders.placed", "payments.failed" });

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(stoppingToken);
            if (result?.Message == null) continue;

            switch (result.Topic)
            {
                case "orders.placed":
                    await HandleOrderPlacedAsync(result.Message.Value);
                    break;

                case "payments.failed":
                    await HandlePaymentFailedAsync(result.Message.Value);
                    break;
            }

            consumer.Commit(result);
        }
    }

    private async Task HandleOrderPlacedAsync(string eventJson)
    {
        var orderPlaced = JsonSerializer.Deserialize<OrderPlaced>(eventJson)!;

        bool stockAvailable = await _inventory.TryReserveAsync(orderPlaced.OrderId, orderPlaced.Items);

        if (stockAvailable)
        {
            await _producer.ProduceAsync("inventory.reserved",
                orderPlaced.OrderId,
                JsonSerializer.Serialize(new InventoryReserved
                {
                    OrderId = orderPlaced.OrderId,
                    ReservationId = Guid.NewGuid().ToString(),
                }));
        }
        else
        {
            await _producer.ProduceAsync("inventory.unavailable",
                orderPlaced.OrderId,
                JsonSerializer.Serialize(new InventoryUnavailable
                {
                    OrderId = orderPlaced.OrderId,
                    Reason = "Out of stock",
                }));
        }
    }

    private async Task HandlePaymentFailedAsync(string eventJson)
    {
        // Compensation: release the reservation
        var paymentFailed = JsonSerializer.Deserialize<PaymentFailed>(eventJson)!;
        await _inventory.ReleaseReservationAsync(paymentFailed.OrderId);

        await _producer.ProduceAsync("inventory.released",
            paymentFailed.OrderId,
            JsonSerializer.Serialize(new InventoryReleased { OrderId = paymentFailed.OrderId }));
    }
}
```

### 4.2 Orchestration Saga (Central Coordinator)

One service (the orchestrator) tells each service what to do:

```csharp
// OrderSagaOrchestrator.cs
public class OrderSagaOrchestrator : BackgroundService
{
    // Orchestrator listens to all saga-related events and coordinates steps
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildConsumer("order-saga-orchestrator");
        consumer.Subscribe(new[]
        {
            "saga.inventory.reserved",
            "saga.inventory.unavailable",
            "saga.payment.processed",
            "saga.payment.failed",
        });

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(stoppingToken);
            if (result?.Message == null) continue;

            var sagaState = await _sagaStateRepo.GetByOrderIdAsync(result.Message.Key);

            switch (result.Topic)
            {
                case "saga.inventory.reserved":
                    await HandleInventoryReservedAsync(sagaState);
                    break;

                case "saga.inventory.unavailable":
                    await HandleInventoryUnavailableAsync(sagaState);
                    break;

                case "saga.payment.processed":
                    await HandlePaymentProcessedAsync(sagaState);
                    break;

                case "saga.payment.failed":
                    await CompensateAsync(sagaState); // Run compensation actions
                    break;
            }

            consumer.Commit(result);
        }
    }

    private async Task HandleInventoryReservedAsync(OrderSagaState sagaState)
    {
        sagaState.Status = SagaStatus.InventoryReserved;
        await _sagaStateRepo.SaveAsync(sagaState);

        // Tell PaymentService to process payment
        await _producer.ProduceAsync("saga.commands.process-payment",
            sagaState.OrderId,
            JsonSerializer.Serialize(new ProcessPaymentCommand
            {
                OrderId = sagaState.OrderId,
                Amount = sagaState.TotalAmount,
                CustomerId = sagaState.CustomerId,
            }));
    }

    private async Task CompensateAsync(OrderSagaState sagaState)
    {
        sagaState.Status = SagaStatus.Failed;
        await _sagaStateRepo.SaveAsync(sagaState);

        // Compensate: tell InventoryService to release reservation
        if (sagaState.Status >= SagaStatus.InventoryReserved)
        {
            await _producer.ProduceAsync("saga.commands.release-inventory",
                sagaState.OrderId,
                JsonSerializer.Serialize(new ReleaseInventoryCommand
                {
                    OrderId = sagaState.OrderId
                }));
        }

        // Notify order service
        await _producer.ProduceAsync("orders.failed",
            sagaState.OrderId,
            JsonSerializer.Serialize(new OrderFailed
            {
                OrderId = sagaState.OrderId,
                Reason = "Payment failed"
            }));
    }
}
```

### Choreography vs Orchestration

| | Choreography | Orchestration |
|--|---|---|
| **Coordination** | Decentralized | Centralized |
| **Coupling** | Low | Medium (to orchestrator) |
| **Visibility** | Hard to trace | Easy to trace (single place) |
| **Complexity** | Grows with participants | Grows with orchestrator logic |
| **Best for** | Simple sagas (< 4 steps) | Complex sagas (4+ steps) |

---

## 5. Outbox Pattern (Full Implementation)

Guarantees: if the DB write succeeds, the event WILL eventually be published to Kafka.

```csharp
// Models/OutboxMessage.cs
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Topic { get; set; } = string.Empty;
    public string MessageKey { get; set; } = string.Empty;
    public string MessageValue { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public OutboxStatus Status { get; set; } = OutboxStatus.Pending;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

public enum OutboxStatus { Pending, Sent, Failed }
```

```csharp
// EF Core configuration
public class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.MessageValue).HasMaxLength(int.MaxValue);

        // Index for efficient polling of pending messages
        builder.HasIndex(o => new { o.Status, o.CreatedAt });
    }
}
```

```csharp
// Services/OutboxRelayService.cs — polls outbox and publishes to Kafka
public class OutboxRelayService : BackgroundService
{
    private readonly IServiceProvider _sp;
    private readonly IProducer<string, string> _producer;
    private readonly ILogger<OutboxRelayService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingMessagesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }

    private async Task ProcessPendingMessagesAsync(CancellationToken ct)
    {
        using var scope = _sp.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Fetch pending messages (ordered by creation time for FIFO)
        var pending = await db.OutboxMessages
            .Where(m => m.Status == OutboxStatus.Pending && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(100)
            .ToListAsync(ct);

        if (!pending.Any()) return;

        foreach (var outbox in pending)
        {
            try
            {
                var result = await _producer.ProduceAsync(
                    outbox.Topic,
                    new Message<string, string>
                    {
                        Key = outbox.MessageKey,
                        Value = outbox.MessageValue,
                        Headers = new Headers
                        {
                            { "outbox-id", Encoding.UTF8.GetBytes(outbox.Id.ToString()) },
                            { "event-type", Encoding.UTF8.GetBytes(outbox.EventType) },
                        }
                    },
                    ct);

                outbox.Status = OutboxStatus.Sent;
                outbox.ProcessedAt = DateTime.UtcNow;

                _logger.LogDebug("Outbox {Id} published to {TopicPartitionOffset}",
                    outbox.Id, result.TopicPartitionOffset);
            }
            catch (Exception ex)
            {
                outbox.RetryCount++;
                outbox.Error = ex.Message;

                if (outbox.RetryCount >= 5)
                {
                    outbox.Status = OutboxStatus.Failed;
                    _logger.LogError(ex, "Outbox {Id} permanently failed", outbox.Id);
                }
                else
                {
                    _logger.LogWarning(ex, "Outbox {Id} failed (attempt {Count})", outbox.Id, outbox.RetryCount);
                }
            }
        }

        await db.SaveChangesAsync(ct);
    }
}
```

---

## 6. Event Sourcing with Kafka

In Event Sourcing, the current state is derived by replaying all past events:

```csharp
// The topic IS the event store
// Each partition = one aggregate type's event stream
// Offset = event sequence number

public class OrderEventStore
{
    private readonly IProducer<string, string> _producer;
    private readonly IConsumer<string, string> _consumer;

    // Append event (produce)
    public async Task AppendAsync(string orderId, DomainEvent domainEvent)
    {
        await _producer.ProduceAsync("order-events", new Message<string, string>
        {
            Key = orderId,   // Same key = same partition = ordered per order
            Value = JsonSerializer.Serialize(domainEvent),
            Headers = new Headers
            {
                { "event-type", Encoding.UTF8.GetBytes(domainEvent.EventType) }
            }
        });
    }

    // Rebuild aggregate state from events (consume)
    public async Task<Order?> GetOrderAsync(string orderId, int partition)
    {
        var tp = new TopicPartition("order-events", partition);
        _consumer.Assign(tp);
        _consumer.Seek(new TopicPartitionOffset(tp, Offset.Beginning));

        var order = new Order();
        var watermark = _consumer.GetWatermarkOffsets(tp);

        while (true)
        {
            var result = _consumer.Consume(TimeSpan.FromSeconds(1));
            if (result == null) break;

            // Only replay events for THIS order
            if (result.Message.Key == orderId)
                order.Apply(result.Message.Value); // Apply event to rebuild state

            if (result.Offset >= watermark.High - 1)
                break; // Reached end of partition
        }

        return order.OrderId == null ? null : order;
    }
}

// Order aggregate applying events
public class Order
{
    public string? OrderId { get; private set; }
    public string? Status { get; private set; }
    public decimal TotalAmount { get; private set; }

    public void Apply(string eventJson)
    {
        // Determine event type and apply
        using var doc = JsonDocument.Parse(eventJson);
        var eventType = doc.RootElement.GetProperty("EventType").GetString();

        switch (eventType)
        {
            case nameof(OrderPlaced):
                var placed = JsonSerializer.Deserialize<OrderPlaced>(eventJson)!;
                OrderId = placed.OrderId;
                Status = "Placed";
                TotalAmount = placed.TotalAmount;
                break;

            case nameof(PaymentProcessed):
                Status = "Confirmed";
                break;

            case nameof(OrderShipped):
                Status = "Shipped";
                break;
        }
    }
}
```

---

## 7. Pattern Decision Guide

```
Is your operation a simple CRUD with no downstream effects?
└── Yes: Skip Kafka, just use REST/gRPC

Does one action trigger multiple downstream services?
└── Yes: Use Domain Events on Kafka

Do you need atomic multi-service operations?
├── < 4 services: Choreography Saga
└── 4+ services or complex compensation: Orchestration Saga

Do you need write + event to be atomic (no message loss)?
└── Use Outbox Pattern

Do you need full audit history and time travel?
└── Consider Event Sourcing
```

---

## Summary

You now know:
- ✅ Domain events design in .NET
- ✅ CQRS with Kafka as the event bus
- ✅ Choreography Saga (decentralized, event-driven)
- ✅ Orchestration Saga (centralized coordinator)
- ✅ Outbox Pattern (full implementation with EF Core)
- ✅ Event Sourcing fundamentals with Kafka

**Next:** [11-MASSTRANSIT-KAFKA.md](./11-MASSTRANSIT-KAFKA.md) — MassTransit abstraction over Kafka
