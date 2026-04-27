# Service Bus Messaging for .NET Projects
## File 13: Practical Event-Driven Architecture

---

## What You'll Do in This File

By the end of this guide, you'll have:
- ✅ Created an Azure Service Bus namespace
- ✅ Implemented a **Sender** (Web API) to push messages
- ✅ Implemented a **Receiver** (Background Worker) to process messages
- ✅ Handled "Dead-letter" queues (failed messages)
- ✅ Secured messaging with Managed Identity (no keys!)

**Time Required:** 1.5 hours

---

## Why Use Messaging?

```
Without Messaging (Synchronous):
[API] ─── HTTP Wait ───► [Invoice Service]
  │                        │
  └─ If Invoice is slow,    └─ If Invoice is down,
     the API hangs.            the order fails.

With Messaging (Asynchronous):
[API] ─── Send Message ───► [Service Bus] ─── Pull Message ───► [Invoice Service]
  │                            │                                   │
  └─ API responds instantly.   └─ Message stays safe in queue.     └─ Invoice service 
                                                                    works at its own pace.
```

---

## Step 1: Create Service Bus Infrastructure

```powershell
# ─── Variables ──────────────────────────────────────────────────
$PROJECT = "myproject"
$ENV = "dev"
$RG = "$PROJECT-$ENV-rg"
$LOCATION = "eastus"
$SB_NAME = "$PROJECT-$ENV-sb"      # Must be globally unique
$QUEUE_NAME = "order-processing"

# ─── Step 1a: Create Namespace ──────────────────────────────────
# Standard tier is required for Topics/Subscriptions
# Basic tier only has Queues

az servicebus namespace create `
  --name $SB_NAME `
  --resource-group $RG `
  --location $LOCATION `
  --sku Standard

# ─── Step 1b: Create a Queue ────────────────────────────────────

az servicebus queue create `
  --name $QUEUE_NAME `
  --namespace-name $SB_NAME `
  --resource-group $RG `
  --enable-dead-lettering-on-message-expiration true
```

---

## Step 2: Set Up Your .NET Projects

You typically have two projects:
1. **Web API** (Sender)
2. **Worker Service** (Receiver)

```powershell
# Add the Service Bus package to both projects
dotnet add package Azure.Messaging.ServiceBus
dotnet add package Azure.Identity
```

---

## Step 3: Implement the Sender (Web API)

### 1. Register Service Bus Client

```csharp
// Program.cs
using Azure.Messaging.ServiceBus;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// Register ServiceBusClient using Managed Identity
builder.Services.AddSingleton(sp => 
{
    var namespaceName = builder.Configuration["ServiceBus:Namespace"];
    return new ServiceBusClient($"{namespaceName}.servicebus.windows.net", new DefaultAzureCredential());
});

var app = builder.Build();
```

### 2. Create the Message Sender

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly ServiceBusClient _client;
    private readonly string _queueName = "order-processing";

    public OrdersController(ServiceBusClient client)
    {
        _client = client;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(OrderRequest order)
    {
        // 1. Create a sender
        ServiceBusSender sender = _client.CreateSender(_queueName);

        // 2. Create the message
        var messagePayload = JsonSerializer.Serialize(order);
        ServiceBusMessage message = new ServiceBusMessage(messagePayload)
        {
            ContentType = "application/json",
            Subject = "NewOrder",
            MessageId = Guid.NewGuid().ToString() // For idempotency
        };

        // 3. Send it!
        await sender.SendMessageAsync(message);

        return Accepted(new { Message = "Order received and queued" });
    }
}
```

---

## Step 4: Implement the Receiver (Worker Service)

Create a new .NET Worker Service project.

```csharp
// Worker.cs
public class OrderProcessor : BackgroundService
{
    private readonly ILogger<OrderProcessor> _logger;
    private readonly ServiceBusProcessor _processor;

    public OrderProcessor(ILogger<OrderProcessor> logger, ServiceBusClient client)
    {
        _logger = logger;
        // Create a processor for the queue
        _processor = client.CreateProcessor("order-processing", new ServiceBusProcessorOptions
        {
            AutoCompleteMessages = false, // We manually complete them
            MaxConcurrentCalls = 2       // Process 2 messages at once
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 1. Register handlers
        _processor.ProcessMessageAsync += MessageHandler;
        _processor.ProcessErrorAsync += ErrorHandler;

        // 2. Start processing
        await _processor.StartProcessingAsync(stoppingToken);

        // 3. Wait until told to stop
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        _logger.LogInformation("Processing order: {Body}", body);

        try 
        {
            // Do your work (Save to DB, Send Email, etc.)
            await Task.Delay(1000); 

            // 4. Complete the message (removes it from queue)
            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order");
            // 5. Abandon the message (it will be retried later)
            await args.AbandonMessageAsync(args.Message);
        }
    }

    private Task ErrorHandler(ProcessErrorEventArgs args)
    {
        _logger.LogError(args.Exception, "Service Bus Error");
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        await _processor.CloseAsync(cancellationToken);
        await base.StopAsync(cancellationToken);
    }
}
```

---

## Step 5: Secure with Managed Identity

### 1. Enable Identity
(Done in File 07)

### 2. Assign Roles

| Entity | Role | Purpose |
|--------|------|---------|
| **Web API** | `Azure Service Bus Data Sender` | Send messages to queues |
| **Worker Service** | `Azure Service Bus Data Receiver` | Read and delete messages |

```powershell
# Assign Sender role to Web API
az role assignment create `
  --assignee $WEB_API_IDENTITY_ID `
  --role "Azure Service Bus Data Sender" `
  --scope (az servicebus namespace show --name $SB_NAME -g $RG --query id -o tsv)

# Assign Receiver role to Worker
az role assignment create `
  --assignee $WORKER_IDENTITY_ID `
  --role "Azure Service Bus Data Receiver" `
  --scope (az servicebus namespace show --name $SB_NAME -g $RG --query id -o tsv)
```

---

## Step 6: Handling Failures (Dead-letter Queue)

If a message fails too many times (default 10), Service Bus moves it to the **Dead-letter Queue (DLQ)**.

### How to Monitor the DLQ
1. Go to Azure Portal → Service Bus → Queue → **Service Bus Explorer**.
2. Select "Dead-letter" and peek at messages.
3. Fix the bug in your code, then "Resubmit" the messages back to the main queue.

---

## ✅ Messaging Checklist

- [ ] Service Bus Namespace and Queue created
- [ ] `Azure.Messaging.ServiceBus` package added
- [ ] Managed Identity roles assigned (Sender/Receiver)
- [ ] API sends JSON messages with `DefaultAzureCredential`
- [ ] Worker processes messages with `ServiceBusProcessor`
- [ ] Dead-lettering enabled for failed messages
- [ ] Idempotency handled using `MessageId`

---

> **Next Step:** Managing costs → [08-COST-MANAGEMENT-GUIDE.md](08-COST-MANAGEMENT-GUIDE.md)
