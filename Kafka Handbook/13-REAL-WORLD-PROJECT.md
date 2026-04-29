# Real-World Project: Order Processing System
## File 13: Full Kafka Implementation — Zero to Production

---

## Project Overview

You'll build a complete **Order Processing System** using:
- Kafka as the event backbone
- .NET 8 microservices
- Azure Event Hubs (production) / Docker Kafka (development)
- MassTransit for saga orchestration
- Serilog + OpenTelemetry for observability
- Docker Compose for local development

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ORDER PROCESSING SYSTEM                                │
│                                                                               │
│  ┌──────────────┐    ┌──────────────────────────────────────────────────┐   │
│  │   API Gateway│    │                  KAFKA TOPICS                     │   │
│  │  (ASP.NET)   │    │  orders.placed  │ inventory.reserved │ orders.dlq │   │
│  │              │    │  payments.done  │ orders.shipped     │ audit.log  │   │
│  └──────┬───────┘    └──────────────────────────────────────────────────┘   │
│         │                                                                     │
│         ▼                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐ │
│  │   Order      │    │  Inventory   │    │  Payment     │    │  Email     │ │
│  │   Service    │    │  Service     │    │  Service     │    │  Service   │ │
│  │  (Producer)  │    │  (Consumer)  │    │  (Consumer)  │    │ (Consumer) │ │
│  │              │    │              │    │              │    │            │ │
│  │  SQL Server  │    │  SQL Server  │    │  SQL Server  │    │            │ │
│  └──────────────┘    └──────────────┘    └──────────────┘    └────────────┘ │
│                                                                               │
│  ┌──────────────┐    ┌──────────────┐                                        │
│  │  Saga        │    │  Read Model  │                                        │
│  │  Orchestrator│    │  Updater     │                                        │
│  │  (State Mach)│    │  (Consumer)  │                                        │
│  └──────────────┘    └──────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
OrderProcessingSystem/
├── docker-compose.yml                  ← Kafka + all services
├── docker-compose.dev.yml             ← Dev overrides
│
├── src/
│   ├── Shared/
│   │   ├── Contracts/                 ← Message contracts (interfaces)
│   │   │   ├── IOrderPlaced.cs
│   │   │   ├── IInventoryReserved.cs
│   │   │   ├── IPaymentProcessed.cs
│   │   │   └── IOrderShipped.cs
│   │   ├── Models/                    ← Shared domain models
│   │   └── Constants/                 ← Topic names, group IDs
│   │
│   ├── OrderService/                  ← ASP.NET Core API
│   │   ├── Controllers/
│   │   ├── Services/
│   │   ├── Infrastructure/
│   │   └── Program.cs
│   │
│   ├── InventoryService/              ← .NET Worker Service
│   │   ├── Consumers/
│   │   ├── Services/
│   │   └── Program.cs
│   │
│   ├── PaymentService/                ← .NET Worker Service
│   │   ├── Consumers/
│   │   ├── Services/
│   │   └── Program.cs
│   │
│   ├── EmailService/                  ← .NET Worker Service
│   │   ├── Consumers/
│   │   └── Program.cs
│   │
│   └── SagaOrchestrator/             ← .NET Worker Service (MassTransit)
│       ├── Sagas/
│       └── Program.cs
│
└── k8s/                               ← Kubernetes manifests
    ├── kafka/
    ├── order-service/
    └── inventory-service/
```

---

## Step 1: Create the Solution

```powershell
mkdir C:\Projects\OrderProcessingSystem
cd C:\Projects\OrderProcessingSystem

# Create solution
dotnet new sln -n OrderProcessingSystem

# Create projects
dotnet new classlib -n Shared -o src/Shared
dotnet new webapi -n OrderService -o src/OrderService
dotnet new worker -n InventoryService -o src/InventoryService
dotnet new worker -n PaymentService -o src/PaymentService
dotnet new worker -n EmailService -o src/EmailService
dotnet new worker -n SagaOrchestrator -o src/SagaOrchestrator

# Add all to solution
Get-ChildItem -Path src -Filter "*.csproj" -Recurse | ForEach-Object {
    dotnet sln add $_.FullName
}

# Add shared references
@("OrderService", "InventoryService", "PaymentService", "EmailService", "SagaOrchestrator") | ForEach-Object {
    dotnet add "src/$_/$_.csproj" reference "src/Shared/Shared.csproj"
}

# Add NuGet packages
$allServices = @("OrderService", "InventoryService", "PaymentService", "EmailService", "SagaOrchestrator")
$allServices | ForEach-Object {
    dotnet add "src/$_/$_.csproj" package Confluent.Kafka
    dotnet add "src/$_/$_.csproj" package MassTransit
    dotnet add "src/$_/$_.csproj" package MassTransit.Kafka
    dotnet add "src/$_/$_.csproj" package Serilog.AspNetCore
    dotnet add "src/$_/$_.csproj" package Serilog.Sinks.Console
}

dotnet add src/OrderService/OrderService.csproj package MassTransit.EntityFrameworkCore
dotnet add src/SagaOrchestrator/SagaOrchestrator.csproj package MassTransit.EntityFrameworkCore
```

---

## Step 2: Message Contracts (Shared)

```csharp
// src/Shared/Contracts/IOrderPlaced.cs
namespace Shared.Contracts;

public interface IOrderPlaced
{
    Guid OrderId { get; }
    Guid CustomerId { get; }
    string CustomerEmail { get; }
    decimal TotalAmount { get; }
    IReadOnlyList<OrderLineItem> Items { get; }
    string ShippingAddress { get; }
    DateTime PlacedAt { get; }
}

public record OrderLineItem(string ProductId, string ProductName, int Quantity, decimal UnitPrice);
```

```csharp
// src/Shared/Contracts/IInventoryReserved.cs
namespace Shared.Contracts;

public interface IInventoryReserved
{
    Guid OrderId { get; }
    string ReservationId { get; }
    DateTime ReservedAt { get; }
}

public interface IInventoryUnavailable
{
    Guid OrderId { get; }
    string Reason { get; }
}
```

```csharp
// src/Shared/Contracts/IPaymentProcessed.cs
namespace Shared.Contracts;

public interface IPaymentProcessed
{
    Guid OrderId { get; }
    string TransactionId { get; }
    bool Success { get; }
    decimal AmountCharged { get; }
    string? FailureReason { get; }
    DateTime ProcessedAt { get; }
}
```

```csharp
// src/Shared/Constants/Topics.cs
namespace Shared.Constants;

public static class Topics
{
    public const string OrdersPlaced = "orders.placed";
    public const string InventoryReserved = "inventory.reserved";
    public const string InventoryUnavailable = "inventory.unavailable";
    public const string PaymentsProcessed = "payments.processed";
    public const string OrdersShipped = "orders.shipped";
    public const string OrdersFailed = "orders.failed";
    public const string AuditLog = "audit.log";

    // DLQ topics
    public const string OrdersPlacedDlq = "orders.placed.dlq";
    public const string PaymentsDlq = "payments.processed.dlq";
}

public static class ConsumerGroups
{
    public const string InventoryService = "inventory-service";
    public const string PaymentService = "payment-service";
    public const string EmailService = "email-service";
    public const string SagaOrchestrator = "order-saga";
    public const string ReadModelUpdater = "read-model-updater";
    public const string AuditLogger = "audit-logger";
}
```

---

## Step 3: Order Service (API + Producer)

```csharp
// src/OrderService/Controllers/OrdersController.cs
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    [HttpPost]
    [ProducesResponseType(typeof(PlaceOrderResponse), StatusCodes.Status202Accepted)]
    public async Task<IActionResult> PlaceOrder(
        [FromBody] PlaceOrderRequest request,
        CancellationToken cancellationToken)
    {
        var orderId = await _orderService.PlaceOrderAsync(request, cancellationToken);

        _logger.LogInformation("Order {OrderId} accepted", orderId);

        return Accepted(new PlaceOrderResponse(orderId, "Order placed, processing..."));
    }

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrder(Guid orderId)
    {
        var order = await _orderService.GetOrderAsync(orderId);
        return order == null ? NotFound() : Ok(order);
    }
}
```

```csharp
// src/OrderService/Services/OrderService.cs
public class OrderDomainService : IOrderService
{
    private readonly AppDbContext _db;
    private readonly ITopicProducer<IOrderPlaced> _orderPlacedProducer;
    private readonly ILogger<OrderDomainService> _logger;

    public async Task<Guid> PlaceOrderAsync(PlaceOrderRequest request, CancellationToken ct)
    {
        // Validate
        if (!request.Items.Any())
            throw new ArgumentException("Order must have at least one item");

        // Create order entity
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            CustomerEmail = request.CustomerEmail,
            TotalAmount = request.Items.Sum(i => i.UnitPrice * i.Quantity),
            Status = OrderStatus.Pending,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice,
            }).ToList(),
            PlacedAt = DateTime.UtcNow,
        };

        // Save to DB and publish event atomically (outbox pattern via MassTransit)
        using var transaction = await _db.Database.BeginTransactionAsync(ct);

        _db.Orders.Add(order);

        // MassTransit outbox stores the event in the same DB transaction
        await _orderPlacedProducer.Produce(new
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            CustomerEmail = order.CustomerEmail,
            TotalAmount = order.TotalAmount,
            Items = order.Items.Select(i =>
                new OrderLineItem(i.ProductId, i.ProductName, i.Quantity, i.UnitPrice)).ToList(),
            ShippingAddress = request.ShippingAddress,
            PlacedAt = order.PlacedAt,
        }, ct);

        await _db.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);

        _logger.LogInformation("Order {OrderId} created and event queued", order.Id);
        return order.Id;
    }
}
```

---

## Step 4: Inventory Service (Consumer)

```csharp
// src/InventoryService/Consumers/OrderPlacedConsumer.cs
public class OrderPlacedConsumer : IConsumer<IOrderPlaced>
{
    private readonly IInventoryRepository _inventory;
    private readonly ITopicProducer<IInventoryReserved> _reservedProducer;
    private readonly ITopicProducer<IInventoryUnavailable> _unavailableProducer;
    private readonly ILogger<OrderPlacedConsumer> _logger;

    public async Task Consume(ConsumeContext<IOrderPlaced> context)
    {
        var order = context.Message;

        _logger.LogInformation("Reserving inventory for order {OrderId}", order.OrderId);

        // Check and reserve stock for each item
        var allAvailable = true;
        var reservationId = Guid.NewGuid().ToString();

        foreach (var item in order.Items)
        {
            var available = await _inventory.CheckAndReserveAsync(
                item.ProductId, item.Quantity, order.OrderId.ToString());

            if (!available)
            {
                allAvailable = false;
                _logger.LogWarning(
                    "Product {ProductId} out of stock for order {OrderId}",
                    item.ProductId, order.OrderId);
                break;
            }
        }

        if (allAvailable)
        {
            await _reservedProducer.Produce(new
            {
                OrderId = order.OrderId,
                ReservationId = reservationId,
                ReservedAt = DateTime.UtcNow,
            });

            _logger.LogInformation("Inventory reserved for order {OrderId}", order.OrderId);
        }
        else
        {
            // Release any partial reservations
            await _inventory.ReleaseReservationsAsync(order.OrderId.ToString());

            await _unavailableProducer.Produce(new
            {
                OrderId = order.OrderId,
                Reason = "One or more items are out of stock",
            });
        }
    }
}
```

---

## Step 5: Payment Service (Consumer)

```csharp
// src/PaymentService/Consumers/InventoryReservedConsumer.cs
public class InventoryReservedConsumer : IConsumer<IInventoryReserved>
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly IOrderDetailsClient _orderClient; // HTTP to OrderService
    private readonly ITopicProducer<IPaymentProcessed> _paymentProducer;
    private readonly ILogger<InventoryReservedConsumer> _logger;

    public async Task Consume(ConsumeContext<IInventoryReserved> context)
    {
        var reservation = context.Message;

        // Get order details
        var order = await _orderClient.GetOrderAsync(reservation.OrderId);

        _logger.LogInformation("Processing payment for order {OrderId}, amount: {Amount:C}",
            order.OrderId, order.TotalAmount);

        try
        {
            var result = await _paymentGateway.ChargeAsync(
                customerId: order.CustomerId,
                amount: order.TotalAmount,
                orderId: order.OrderId);

            await _paymentProducer.Produce(new
            {
                OrderId = order.OrderId,
                TransactionId = result.TransactionId,
                Success = result.Success,
                AmountCharged = order.TotalAmount,
                FailureReason = result.FailureReason,
                ProcessedAt = DateTime.UtcNow,
            });

            _logger.LogInformation(
                "Payment {Status} for order {OrderId}",
                result.Success ? "succeeded" : "failed",
                order.OrderId);
        }
        catch (PaymentGatewayException ex)
        {
            // Payment gateway error — let MassTransit retry
            _logger.LogError(ex, "Payment gateway error for order {OrderId}", order.OrderId);
            throw; // MassTransit will retry per configuration
        }
    }
}
```

---

## Step 6: Email Service (Consumer)

```csharp
// src/EmailService/Consumers/MultiEventEmailConsumer.cs
// Single consumer handles multiple event types for email notifications

public class OrderPlacedEmailConsumer : IConsumer<IOrderPlaced>
{
    private readonly IEmailSender _emailSender;

    public async Task Consume(ConsumeContext<IOrderPlaced> context)
    {
        await _emailSender.SendAsync(
            to: context.Message.CustomerEmail,
            subject: $"Order #{context.Message.OrderId} Received",
            body: $"Thank you! Your order for {context.Message.TotalAmount:C} has been received.");
    }
}

public class PaymentProcessedEmailConsumer : IConsumer<IPaymentProcessed>
{
    private readonly IEmailSender _emailSender;
    private readonly IOrderDetailsClient _orderClient;

    public async Task Consume(ConsumeContext<IPaymentProcessed> context)
    {
        var order = await _orderClient.GetOrderAsync(context.Message.OrderId);

        if (context.Message.Success)
        {
            await _emailSender.SendAsync(
                to: order.CustomerEmail,
                subject: $"Payment Confirmed — Order #{context.Message.OrderId}",
                body: $"Your payment of {context.Message.AmountCharged:C} was successful. " +
                      $"Transaction: {context.Message.TransactionId}");
        }
        else
        {
            await _emailSender.SendAsync(
                to: order.CustomerEmail,
                subject: $"Payment Failed — Order #{context.Message.OrderId}",
                body: $"Unfortunately, your payment failed: {context.Message.FailureReason}. " +
                       "Please try again or contact support.");
        }
    }
}
```

---

## Step 7: Docker Compose (Full Stack)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ─── Infrastructure ────────────────────────────────────────
  kafka:
    image: bitnami/kafka:3.7
    container_name: kafka
    ports:
      - "29092:29092"
    environment:
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,PLAINTEXT_HOST://:29092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_KRAFT_CLUSTER_ID=MkU3OEVBNTcwNTJENDM2Qk
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false
    volumes:
      - kafka_data:/bitnami/kafka
    healthcheck:
      test: ["CMD", "kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      retries: 5

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      kafka:
        condition: service_healthy

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    ports:
      - "1433:1433"
    environment:
      SA_PASSWORD: "YourStr0ngPassword!"
      ACCEPT_EULA: "Y"
    volumes:
      - sqlserver_data:/var/opt/mssql

  # ─── Topic Initializer (runs once, then exits) ─────────────
  kafka-init:
    image: bitnami/kafka:3.7
    depends_on:
      kafka:
        condition: service_healthy
    entrypoint: ["/bin/bash", "-c"]
    command: |
      "kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic orders.placed --partitions 6 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic inventory.reserved --partitions 6 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic inventory.unavailable --partitions 3 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic payments.processed --partitions 6 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic orders.shipped --partitions 3 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic orders.failed --partitions 3 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic audit.log --partitions 6 --replication-factor 1
      kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic orders.placed.dlq --partitions 3 --replication-factor 1
      echo 'All topics created'"

  # ─── Microservices ─────────────────────────────────────────
  order-service:
    build:
      context: .
      dockerfile: src/OrderService/Dockerfile
    ports:
      - "5000:8080"
    environment:
      - Kafka__BootstrapServers=kafka:9092
      - ConnectionStrings__Default=Server=sqlserver;Database=Orders;User=sa;Password=YourStr0ngPassword!;TrustServerCertificate=True
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      kafka:
        condition: service_healthy
      kafka-init:
        condition: service_completed_successfully

  inventory-service:
    build:
      context: .
      dockerfile: src/InventoryService/Dockerfile
    environment:
      - Kafka__BootstrapServers=kafka:9092
      - ConnectionStrings__Default=Server=sqlserver;Database=Inventory;User=sa;Password=YourStr0ngPassword!;TrustServerCertificate=True
    depends_on:
      kafka:
        condition: service_healthy
      kafka-init:
        condition: service_completed_successfully

  payment-service:
    build:
      context: .
      dockerfile: src/PaymentService/Dockerfile
    environment:
      - Kafka__BootstrapServers=kafka:9092
    depends_on:
      kafka:
        condition: service_healthy

  email-service:
    build:
      context: .
      dockerfile: src/EmailService/Dockerfile
    environment:
      - Kafka__BootstrapServers=kafka:9092
      - Smtp__Host=smtp.example.com
    depends_on:
      kafka:
        condition: service_healthy

volumes:
  kafka_data:
  sqlserver_data:
```

---

## Step 8: Run the System

```powershell
# 1. Start infrastructure + services
docker compose up -d

# 2. Wait for healthy state
docker compose ps

# 3. Test: Place an order
$body = @{
    customerId = "CUST-001"
    customerEmail = "john@example.com"
    items = @(
        @{ productId = "P001"; productName = "Laptop"; quantity = 1; unitPrice = 999.99 }
    )
    shippingAddress = "123 Main St, Seattle WA 98101"
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Uri "http://localhost:5000/api/orders" `
  -Method POST `
  -ContentType "application/json" `
  -Body $body

# 4. Watch events flow in Kafka UI
Start-Process "http://localhost:8080"

# 5. Watch service logs
docker compose logs -f inventory-service payment-service email-service
```

---

## Step 9: Event Flow Verification

```
1. POST /api/orders → 202 Accepted (orderId: "abc-123")
2. Kafka UI: orders.placed topic receives message
3. InventoryService logs: "Reserving inventory for order abc-123"
4. Kafka UI: inventory.reserved topic receives message
5. PaymentService logs: "Processing payment for order abc-123"
6. Kafka UI: payments.processed topic receives message
7. EmailService logs: "Sending confirmation email"
8. GET /api/orders/abc-123 → status: "Confirmed"
```

---

## Step 10: Production Checklist

```
BEFORE GOING TO PRODUCTION:
□ Switch to Azure Event Hubs (remove local Docker Kafka)
□ Use Managed Identity instead of connection strings
□ Enable schema registry validation
□ Set up consumer lag alerts (threshold: based on your SLA)
□ Configure DLQ monitoring alert (any DLQ message = alert)
□ Enable distributed tracing (send to Azure Monitor / Jaeger)
□ Set appropriate retention policies per topic
□ Configure replication factor = 3 for all topics
□ Load test with realistic message volumes
□ Chaos test: kill consumer instances, verify recovery
□ Document run books for common issues (consumer lag, DLQ, rebalancing)
□ Set up Kafka UI (or Azure Event Hubs portal) for ops team
□ Enable TLS encryption for all Kafka connections
□ Review and tune MaxPollIntervalMs for your slowest consumer
□ Set partition counts based on expected throughput × expected consumers
```

---

## Congratulations! 🎉

You've completed the **Kafka .NET Handbook**. You now know:

| Topic | Status |
|-------|--------|
| Kafka fundamentals (topics, partitions, offsets) | ✅ |
| Local setup with Docker | ✅ |
| .NET producer and consumer basics | ✅ |
| Advanced producer (acks, idempotence, transactions) | ✅ |
| Advanced consumer (groups, rebalancing, batch) | ✅ |
| Serialization (JSON, Avro, Protobuf) | ✅ |
| Error handling, retry, DLQ | ✅ |
| Azure Event Hubs and AKS deployment | ✅ |
| CQRS, Saga, Outbox patterns | ✅ |
| MassTransit abstraction | ✅ |
| Monitoring and observability | ✅ |
| Full production project | ✅ |

---

## Further Reading

- [Confluent .NET Developer Guide](https://developer.confluent.io/get-started/dotnet/)
- [MassTransit Documentation](https://masstransit.io/documentation/configuration/transports/kafka)
- [Azure Event Hubs for Kafka](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-for-kafka-ecosystem-overview)
- [Strimzi Kafka on Kubernetes](https://strimzi.io/documentation/)
- [Designing Event-Driven Systems — O'Reilly (Free)](https://www.confluent.io/designing-event-driven-systems/)
