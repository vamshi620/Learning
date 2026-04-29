# MassTransit + Kafka
## File 11: Higher-Level Kafka Abstraction in .NET

---

## What You'll Learn

- Why use MassTransit over raw Confluent.Kafka
- Setting up MassTransit with Kafka
- Consumers, producers and sagas with MassTransit
- Outbox pattern built into MassTransit
- Testing MassTransit Kafka consumers

**Time Required:** 45 minutes

---

## 1. Why MassTransit?

Raw `Confluent.Kafka` gives you full control but requires a lot of boilerplate:
- Manual serialization/deserialization
- Consumer loop management
- Error handling and retry logic
- DI wiring for consumers
- Outbox pattern implementation

**MassTransit** provides all of this out of the box:

```
Raw Confluent.Kafka:                  MassTransit.Kafka:
- Manual consumer loop             ✅ Automatic consumer hosting
- Manual serialization             ✅ Automatic JSON/System.Text.Json
- Write retry logic yourself       ✅ Built-in retry middleware
- Write DLQ yourself               ✅ Built-in error queue support
- Write outbox yourself            ✅ Built-in transactional outbox
- Manual saga state management     ✅ Built-in saga state machines
- Hard to test                     ✅ InMemory test harness
```

> 💡 **When to use raw Confluent.Kafka:** High-throughput, performance-critical scenarios or when you need fine-grained control.  
> **When to use MassTransit:** Microservices business logic, sagas, standard CRUD-event flows.

---

## 2. Installation

```powershell
# Core MassTransit packages
dotnet add package MassTransit
dotnet add package MassTransit.Kafka

# Outbox (choose your EF provider)
dotnet add package MassTransit.EntityFrameworkCore

# For testing
dotnet add package MassTransit.Testing
```

---

## 3. Basic Setup

### 3.1 Define Messages (Contracts)

```csharp
// Contracts/IOrderPlaced.cs
// Use interfaces for message contracts — allows different implementations
namespace MyApp.Contracts;

public interface IOrderPlaced
{
    Guid OrderId { get; }
    Guid CustomerId { get; }
    decimal TotalAmount { get; }
    DateTime PlacedAt { get; }
}

public interface IPaymentProcessed
{
    Guid OrderId { get; }
    Guid TransactionId { get; }
    bool Success { get; }
    decimal AmountCharged { get; }
}

public interface IOrderShipped
{
    Guid OrderId { get; }
    string TrackingNumber { get; }
    string Carrier { get; }
}
```

### 3.2 Configure MassTransit with Kafka

```csharp
// Program.cs
using MassTransit;
using MyApp.Contracts;
using MyApp.Consumers;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMassTransit(cfg =>
{
    // Register consumers
    cfg.AddConsumer<OrderPlacedConsumer>();
    cfg.AddConsumer<PaymentProcessedConsumer>();

    // Configure Kafka rider
    cfg.UsingInMemory(); // In-memory bus for non-Kafka messages

    cfg.AddRider(rider =>
    {
        // Register producers
        rider.AddProducer<IOrderPlaced>("orders.placed");
        rider.AddProducer<IPaymentProcessed>("payments.processed");

        // Register consumers
        rider.AddConsumer<OrderPlacedConsumer>();
        rider.AddConsumer<PaymentProcessedConsumer>();

        rider.UsingKafka((ctx, kafka) =>
        {
            kafka.Host("localhost:9092");
            // For Azure Event Hubs:
            // kafka.Host("myns.servicebus.windows.net:9093", h =>
            // {
            //     h.UseSasl(sasl =>
            //     {
            //         sasl.Mechanism = SaslMechanism.Plain;
            //         sasl.Username = "$ConnectionString";
            //         sasl.Password = connectionString;
            //         sasl.SecurityProtocol = SecurityProtocol.SaslSsl;
            //     });
            // });

            // Configure topic-endpoint (consumer)
            kafka.TopicEndpoint<IOrderPlaced>("orders.placed", "order-processing-service", endpoint =>
            {
                endpoint.ConfigureConsumer<OrderPlacedConsumer>(ctx);

                // Retry middleware
                endpoint.UseMessageRetry(r => r.Exponential(
                    retryLimit: 3,
                    minInterval: TimeSpan.FromSeconds(1),
                    maxInterval: TimeSpan.FromSeconds(30),
                    intervalDelta: TimeSpan.FromSeconds(1)));

                // Error queue (DLQ)
                endpoint.DiscardSkippedMessages();
                // Or: endpoint.DeadLetterQueue("orders.placed.dlq");

                // Concurrent message processing
                endpoint.ConcurrentMessageLimit = 10;
            });

            kafka.TopicEndpoint<IPaymentProcessed>("payments.processed", "order-service", endpoint =>
            {
                endpoint.ConfigureConsumer<PaymentProcessedConsumer>(ctx);
                endpoint.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(5)));
            });
        });
    });
});

var app = builder.Build();
app.MapGet("/health", () => "OK");
await app.RunAsync();
```

---

## 4. Consumers

```csharp
// Consumers/OrderPlacedConsumer.cs
using MassTransit;
using MyApp.Contracts;

public class OrderPlacedConsumer : IConsumer<IOrderPlaced>
{
    private readonly IOrderRepository _orderRepo;
    private readonly ILogger<OrderPlacedConsumer> _logger;

    public OrderPlacedConsumer(IOrderRepository orderRepo, ILogger<OrderPlacedConsumer> logger)
    {
        _orderRepo = orderRepo;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<IOrderPlaced> context)
    {
        var order = context.Message;

        _logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
            order.OrderId, order.CustomerId);

        // MassTransit handles:
        // - Deserialization ✅
        // - Retry on exception ✅ (configured in setup)
        // - Offset commit after success ✅
        // - DI injection ✅

        // YOUR BUSINESS LOGIC:
        await _orderRepo.ReserveInventoryAsync(order.OrderId, order.Items);

        _logger.LogInformation("Inventory reserved for order {OrderId}", order.OrderId);

        // If you throw an exception, MassTransit will retry per your configuration
        // If retries exhausted, message goes to error queue
    }
}
```

---

## 5. Producers

```csharp
// Services/OrderService.cs
using MassTransit;
using MyApp.Contracts;

public class OrderService
{
    private readonly ITopicProducer<IOrderPlaced> _producer;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        ITopicProducer<IOrderPlaced> producer,  // Injected by MassTransit
        ILogger<OrderService> logger)
    {
        _producer = producer;
        _logger = logger;
    }

    public async Task PlaceOrderAsync(PlaceOrderRequest request)
    {
        // Save order to database...
        var orderId = Guid.NewGuid();

        // Publish to Kafka — MassTransit handles serialization, routing
        await _producer.Produce(new
        {
            OrderId = orderId,
            CustomerId = request.CustomerId,
            TotalAmount = request.TotalAmount,
            PlacedAt = DateTime.UtcNow,
        });

        _logger.LogInformation("Order {OrderId} placed and event published", orderId);
    }
}

// Register in DI:
// builder.Services.AddScoped<OrderService>();
// ITopicProducer<IOrderPlaced> is automatically injected by MassTransit
```

---

## 6. Saga with MassTransit (State Machine)

MassTransit has built-in saga support using state machines:

```csharp
// Sagas/OrderSagaState.cs
public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }  // OrderId as correlation
    public string CurrentState { get; set; } = string.Empty;

    // Saga data
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public string? TransactionId { get; set; }
    public string? TrackingNumber { get; set; }
    public DateTime? CompletedAt { get; set; }
}

// Sagas/OrderSaga.cs
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    // States
    public State WaitingForPayment { get; private set; } = null!;
    public State WaitingForShipment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    // Events that transition state
    public Event<IOrderPlaced> OrderPlaced { get; private set; } = null!;
    public Event<IPaymentProcessed> PaymentProcessed { get; private set; } = null!;
    public Event<IOrderShipped> OrderShipped { get; private set; } = null!;

    public OrderSaga()
    {
        // OrderId is the correlation ID
        Event(() => OrderPlaced, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => OrderShipped, x => x.CorrelateById(m => m.Message.OrderId));

        // Initial state
        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                })
                .TransitionTo(WaitingForPayment)
                .Publish(ctx => new  // Publish next command
                {
                    __TypeId = "process-payment",
                    OrderId = ctx.Message.OrderId,
                    Amount = ctx.Message.TotalAmount,
                })
        );

        // WaitingForPayment state transitions
        During(WaitingForPayment,
            When(PaymentProcessed, ctx => ctx.Message.Success)
                .Then(ctx => ctx.Saga.TransactionId = ctx.Message.TransactionId.ToString())
                .TransitionTo(WaitingForShipment),

            When(PaymentProcessed, ctx => !ctx.Message.Success)
                .Then(ctx => Console.WriteLine($"Payment failed for {ctx.Saga.CorrelationId}"))
                .TransitionTo(Failed)
                .Finalize()
        );

        // WaitingForShipment state transitions
        During(WaitingForShipment,
            When(OrderShipped)
                .Then(ctx =>
                {
                    ctx.Saga.TrackingNumber = ctx.Message.TrackingNumber;
                    ctx.Saga.CompletedAt = DateTime.UtcNow;
                })
                .TransitionTo(Completed)
                .Finalize()
        );

        SetCompletedWhenFinalized(); // Remove saga state when done
    }
}
```

### Register Saga with MassTransit

```csharp
builder.Services.AddMassTransit(cfg =>
{
    // Add saga with Entity Framework Core state storage
    cfg.AddSagaStateMachine<OrderSaga, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<DbContext, SagaDbContext>((sp, opts) =>
            {
                opts.UseSqlServer(connectionString);
            });
        });

    cfg.AddRider(rider =>
    {
        rider.UsingKafka((ctx, kafka) =>
        {
            kafka.Host("localhost:9092");

            // Saga topic endpoints
            kafka.TopicEndpoint<IOrderPlaced>("orders.placed", "order-saga", e =>
            {
                e.ConfigureSaga<OrderSagaState>(ctx);
            });
            kafka.TopicEndpoint<IPaymentProcessed>("payments.processed", "order-saga", e =>
            {
                e.ConfigureSaga<OrderSagaState>(ctx);
            });
            kafka.TopicEndpoint<IOrderShipped>("orders.shipped", "order-saga", e =>
            {
                e.ConfigureSaga<OrderSagaState>(ctx);
            });
        });
    });
});
```

---

## 7. Transactional Outbox with MassTransit

MassTransit has a built-in outbox — much simpler than building your own:

```csharp
builder.Services.AddMassTransit(cfg =>
{
    // Enable the outbox
    cfg.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        // Outbox cleanup: remove sent messages after 1 hour
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.DuplicateDetectionWindow = TimeSpan.FromMinutes(30);
        o.UseSqlServer(); // or UsePostgres(), UseMySql()
        o.UseBusOutbox(); // Use the bus outbox (not saga outbox)
    });

    cfg.AddRider(rider =>
    {
        rider.AddProducer<IOrderPlaced>("orders.placed");
        rider.UsingKafka((ctx, kafka) =>
        {
            kafka.Host("localhost:9092");
        });
    });
});
```

```csharp
// Usage in OrderService — now uses outbox automatically!
public class OrderService
{
    private readonly AppDbContext _db;
    private readonly IPublishEndpoint _publishEndpoint; // MassTransit publish

    public async Task PlaceOrderAsync(PlaceOrderRequest request)
    {
        using var transaction = await _db.Database.BeginTransactionAsync();

        // Save order (your business entity)
        var order = new Order { Id = Guid.NewGuid(), ... };
        _db.Orders.Add(order);

        // Publish event — MassTransit outbox stores it in DB (same transaction!)
        await _publishEndpoint.Publish<IOrderPlaced>(new
        {
            OrderId = order.Id,
            CustomerId = request.CustomerId,
            TotalAmount = order.TotalAmount,
        });

        await _db.SaveChangesAsync(); // Order + Outbox message in ONE transaction
        await transaction.CommitAsync();
        // MassTransit relay service picks up the outbox message and sends to Kafka
    }
}
```

---

## 8. Testing with MassTransit Test Harness

```csharp
// Unit test using MassTransit's in-memory test harness
using MassTransit.Testing;

[TestClass]
public class OrderPlacedConsumerTests
{
    private ITestHarness _harness = null!;

    [TestInitialize]
    public async Task Initialize()
    {
        _harness = new InMemoryTestHarness();
        _harness.Consumer<OrderPlacedConsumer>();
        await _harness.Start();
    }

    [TestCleanup]
    public async Task Cleanup()
    {
        await _harness.Stop();
    }

    [TestMethod]
    public async Task OrderPlaced_ShouldReserveInventory()
    {
        // Arrange
        var orderId = Guid.NewGuid();

        // Act — publish a message
        await _harness.Bus.Publish<IOrderPlaced>(new
        {
            OrderId = orderId,
            CustomerId = Guid.NewGuid(),
            TotalAmount = 99.99m,
            PlacedAt = DateTime.UtcNow,
        });

        // Assert — consumer received and processed the message
        Assert.IsTrue(await _harness.Consumed.Any<IOrderPlaced>());

        var consumerHarness = _harness.ConsumerOf<OrderPlacedConsumer>();
        Assert.IsTrue(await consumerHarness.Consumed.Any<IOrderPlaced>());

        // Verify no messages went to error queue
        Assert.IsFalse(await _harness.Published.Any<Fault<IOrderPlaced>>());
    }

    [TestMethod]
    public async Task OrderPlaced_WhenInventoryFails_ShouldRetryAndFault()
    {
        // Inject a failing mock
        // ...

        await _harness.Bus.Publish<IOrderPlaced>(new { ... });

        // Verify fault published (consumer failed after retries)
        Assert.IsTrue(await _harness.Published.Any<Fault<IOrderPlaced>>());
    }
}
```

---

## 9. MassTransit vs Raw Confluent.Kafka — Choosing

### Use MassTransit when:
- ✅ You're building microservices business logic
- ✅ You need sagas / distributed transactions
- ✅ You want the Outbox pattern without building it
- ✅ You need easy unit testing
- ✅ You might switch transports later (RabbitMQ ↔ Kafka ↔ Service Bus)
- ✅ Team is new to Kafka (less boilerplate to manage)

### Use Raw Confluent.Kafka when:
- ✅ Maximum throughput (no abstraction overhead)
- ✅ Custom partitioning strategies
- ✅ Fine-grained offset control
- ✅ Batch consumption patterns
- ✅ Exactly-once semantics with transactions
- ✅ Event streaming / log processing

**Time Required:** 45 minutes

---

## Summary

You now know:
- ✅ Why and when to use MassTransit over raw Confluent.Kafka
- ✅ Setting up MassTransit with Kafka rider
- ✅ Message contracts (interfaces)
- ✅ Consumers with automatic retry and DI
- ✅ Producers with `ITopicProducer<T>`
- ✅ Saga state machines with MassTransit
- ✅ Transactional outbox built into MassTransit
- ✅ Unit testing with the InMemory test harness

**Next:** [12-MONITORING-OBSERVABILITY.md](./12-MONITORING-OBSERVABILITY.md) — Monitoring Kafka in production
