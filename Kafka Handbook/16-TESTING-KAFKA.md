# Testing Kafka Applications in .NET
## File 16: Unit Testing and Integration Testing with Testcontainers

---

## What You'll Learn

- The challenges of testing Kafka consumers and producers
- Mocking `IProducer<K, V>` and `IConsumer<K, V>` for fast Unit Tests
- Writing real Integration Tests using **Testcontainers**

**Time Required:** 30 minutes

---

## 1. Why is Kafka Hard to Test?

Unlike HTTP APIs where you can easily mock `HttpClient` or use `WebApplicationFactory`, Kafka is an asynchronous event broker.
1. **Fire-and-forget:** Producers send messages and continue. You need to verify the message was actually produced.
2. **Infinite loops:** Consumers run in an endless `while` loop. You need a way to stop them gracefully during tests.
3. **Stateful dependencies:** Real Kafka requires Zookeeper/KRaft, making it hard to run reliably in CI pipelines without external setup.

---

## 2. Unit Testing (Mocking Confluent.Kafka)

For pure unit testing, you should **never** spin up a real Kafka broker. Instead, you mock the `IProducer` and `IConsumer` interfaces provided by the `Confluent.Kafka` library.

### 2.1 Testing a Producer

Assume we have an `OrderService` that saves to a DB and publishes an event.

```csharp
public class OrderService
{
    private readonly IProducer<string, string> _producer;

    public OrderService(IProducer<string, string> producer)
    {
        _producer = producer;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        var message = new Message<string, string> 
        { 
            Key = order.Id, 
            Value = JsonSerializer.Serialize(order) 
        };

        await _producer.ProduceAsync("orders.placed", message);
    }
}
```

**Unit Test with Moq:**

```csharp
using Moq;
using Confluent.Kafka;
using Xunit;

public class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrder_ShouldProduceToKafka()
    {
        // Arrange
        var mockProducer = new Mock<IProducer<string, string>>();
        
        // Setup the mock to return a fake DeliveryResult
        mockProducer
            .Setup(p => p.ProduceAsync(
                "orders.placed", 
                It.IsAny<Message<string, string>>(), 
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(new DeliveryResult<string, string> 
            { 
                Topic = "orders.placed", 
                Partition = 0, 
                Offset = 1 
            });

        var service = new OrderService(mockProducer.Object);
        var testOrder = new Order { Id = "123", Amount = 50.0m };

        // Act
        await service.PlaceOrderAsync(testOrder);

        // Assert - Verify the producer was called exactly once with the right key
        mockProducer.Verify(p => p.ProduceAsync(
            "orders.placed",
            It.Is<Message<string, string>>(m => m.Key == "123" && m.Value.Contains("50.0")),
            It.IsAny<CancellationToken>()), 
            Times.Once);
    }
}
```

### 2.2 Testing a Consumer is Harder

Because `Consumer.Consume()` is a blocking call, mocking it requires careful setup so your test doesn't hang infinitely. 

> 💡 **Pro Tip:** In .NET, it is highly recommended to extract your message processing logic out of the Kafka `BackgroundService` into a separate handler class. Then, you only unit test the handler class, ignoring Kafka completely!

```csharp
// DON'T test this directly
public class OrderConsumerService : BackgroundService 
{
    // ...
}

// DO test this directly!
public class OrderProcessor 
{
    public async Task ProcessOrderMessageAsync(string jsonPayload) 
    {
        // Test this logic normally
    }
}
```

---

## 3. Integration Testing with Testcontainers

Unit tests prove your logic works. **Integration tests prove your Kafka configuration works.**

**Testcontainers** is a library that automatically spins up Docker containers (like Kafka) just for the duration of your test, and tears them down after.

### 3.1 Install Packages

```powershell
dotnet add package Testcontainers.Kafka
dotnet add package Confluent.Kafka
dotnet add package xunit
```

### 3.2 The Test Setup

```csharp
using Testcontainers.Kafka;
using Confluent.Kafka;
using Xunit;

public class KafkaIntegrationTests : IAsyncLifetime
{
    // This defines a real Kafka container!
    private readonly KafkaContainer _kafkaContainer = new KafkaBuilder()
        .WithImage("confluentinc/cp-kafka:7.4.0")
        .Build();

    // Runs BEFORE tests start
    public async Task InitializeAsync()
    {
        await _kafkaContainer.StartAsync();
    }

    // Runs AFTER tests finish
    public async Task DisposeAsync()
    {
        await _kafkaContainer.DisposeAsync();
    }

    [Fact]
    public async Task ProduceAndConsume_ShouldWork()
    {
        // 1. Arrange - Get the dynamic port assigned by Docker
        var bootstrapServers = _kafkaContainer.GetBootstrapAddress();
        var topic = "test-topic";

        // Setup Producer
        var producerConfig = new ProducerConfig { BootstrapServers = bootstrapServers };
        using var producer = new ProducerBuilder<string, string>(producerConfig).Build();

        // Setup Consumer
        var consumerConfig = new ConsumerConfig 
        { 
            BootstrapServers = bootstrapServers,
            GroupId = "test-group",
            AutoOffsetReset = AutoOffsetReset.Earliest 
        };
        using var consumer = new ConsumerBuilder<string, string>(consumerConfig).Build();
        consumer.Subscribe(topic);

        // 2. Act - Produce a message
        var message = new Message<string, string> { Key = "key1", Value = "value1" };
        var deliveryResult = await producer.ProduceAsync(topic, message);
        
        // 3. Assert - Consume the message
        // We use a cancellation token with a timeout so the test fails instead of hanging
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        
        var consumeResult = consumer.Consume(cts.Token);

        Assert.NotNull(consumeResult);
        Assert.Equal("key1", consumeResult.Message.Key);
        Assert.Equal("value1", consumeResult.Message.Value);
    }
}
```

### Why Testcontainers is Awesome:
1. **No local setup:** Developers don't need to manually run `docker compose up` before running tests.
2. **CI/CD Ready:** It works perfectly in GitHub Actions or Azure DevOps pipelines (as long as Docker is available on the agent).
3. **Isolation:** Every test run gets a brand-new, clean Kafka cluster. No accidental data pollution between runs.

---

## 4. Testing with MassTransit

If you followed File 11 and used MassTransit, testing is significantly easier. MassTransit provides an **In-Memory Test Harness** that entirely replaces Kafka during unit tests!

```csharp
using MassTransit.Testing;
using Xunit;

public class MassTransitTests
{
    [Fact]
    public async Task OrderConsumer_ShouldConsumeMessage()
    {
        // Arrange
        await using var provider = new ServiceCollection()
            .AddMassTransitTestHarness(x =>
            {
                x.AddConsumer<OrderPlacedConsumer>();
            })
            .BuildServiceProvider(true);

        var harness = provider.GetRequiredService<ITestHarness>();
        await harness.Start();

        // Act - Publish to the in-memory bus
        await harness.Bus.Publish(new OrderPlaced { OrderId = "123" });

        // Assert - Verify the consumer received it
        var consumed = await harness.Consumed.Any<OrderPlaced>();
        Assert.True(consumed);
    }
}
```

---

## Summary

You now know:
- ✅ **Unit Tests:** Mock `IProducer<K,V>` with Moq for fast, isolated tests.
- ✅ **Consumer Logic:** Extract processing logic out of the `BackgroundService` to make unit testing easy.
- ✅ **Integration Tests:** Use **Testcontainers** to spin up real Kafka clusters on the fly in xUnit.
- ✅ **MassTransit:** Use the `AddMassTransitTestHarness` for native in-memory testing without Docker.

**Next:** Review [13-REAL-WORLD-PROJECT.md](./13-REAL-WORLD-PROJECT.md) to see how everything fits together in a full .NET Microservices architecture.
