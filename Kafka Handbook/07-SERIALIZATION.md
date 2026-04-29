# Serialization in .NET Kafka
## File 07: JSON, Avro & Protobuf with Schema Registry

---

## What You'll Learn

- Why serialization matters for Kafka
- JSON serialization with schema validation
- Avro serialization with Schema Registry
- Schema evolution and compatibility rules
- Custom serializers in Confluent.Kafka

---

## 1. Why Serialization Matters

Kafka messages are just **bytes**. Serialization is how you convert your C# objects to bytes (and back).

```
Producer (C#) → Serialize → [bytes in Kafka] → Deserialize → Consumer (C#)

Producer sends:  OrderEvent { OrderId = "123", Amount = 99.99 }
Bytes in Kafka:  7B 22 4F 72 64 65 72 49 64 22 3A 22 31 32 33 22 ...
Consumer reads:  OrderEvent { OrderId = "123", Amount = 99.99 }
```

**Without a schema contract:** Consumer doesn't know what the bytes mean. Producer changes a field name → Consumer breaks.

**With Schema Registry:** Schemas are enforced centrally. Breaking changes are rejected before reaching consumers.

---

## 2. JSON Serialization

### 2.1 Manual JSON (Simplest Approach)

```csharp
// Producer side
var json = JsonSerializer.Serialize(order); // C# → string
var message = new Message<string, string> { Key = order.OrderId, Value = json };

// Consumer side
var order = JsonSerializer.Deserialize<OrderEvent>(result.Message.Value); // string → C#
```

**Pros:** Simple, human-readable, easy to debug in Kafka UI  
**Cons:** No schema enforcement, brittle on field changes, larger payload than binary formats

### 2.2 Custom JSON Serializer/Deserializer

```csharp
// Reusable generic JSON serializer
public class JsonSerializer<T> : ISerializer<T>
{
    private static readonly JsonSerializerOptions Options = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    };

    public byte[] Serialize(T data, SerializationContext context)
    {
        if (data is null) return Array.Empty<byte>();
        return JsonSerializer.SerializeToUtf8Bytes(data, Options);
    }
}

// Reusable generic JSON deserializer
public class JsonDeserializer<T> : IDeserializer<T>
{
    private static readonly JsonSerializerOptions Options = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    };

    public T Deserialize(ReadOnlySpan<byte> data, bool isNull, SerializationContext context)
    {
        if (isNull || data.IsEmpty) return default!;
        return JsonSerializer.Deserialize<T>(data, Options)!;
    }
}

// Use in producer
var producer = new ProducerBuilder<string, OrderEvent>(config)
    .SetValueSerializer(new JsonSerializer<OrderEvent>())
    .Build();

// Now you produce typed objects directly!
await producer.ProduceAsync("orders", new Message<string, OrderEvent>
{
    Key = order.OrderId,
    Value = order  // No manual serialization needed!
});

// Use in consumer
var consumer = new ConsumerBuilder<string, OrderEvent>(config)
    .SetValueDeserializer(new JsonDeserializer<OrderEvent>())
    .Build();

var result = consumer.Consume(token);
OrderEvent order = result.Message.Value; // Already deserialized!
```

---

## 3. Avro Serialization with Schema Registry

Avro is a compact binary format with a schema-first approach. It's the most common choice for production Kafka systems.

### 3.1 Install Packages

```powershell
dotnet add package Confluent.Kafka
dotnet add package Confluent.SchemaRegistry
dotnet add package Confluent.SchemaRegistry.Serdes.Avro
dotnet add package Chr.Avro.Confluent  # Better .NET Avro code gen
```

### 3.2 Define Your Avro Schema

```json
// schemas/order-event.avsc
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.mycompany.orders",
  "doc": "An event representing an order action",
  "fields": [
    {
      "name": "orderId",
      "type": "string",
      "doc": "Unique identifier for the order"
    },
    {
      "name": "customerId",
      "type": "string"
    },
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "OrderStatus",
        "symbols": ["PLACED", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
      }
    },
    {
      "name": "amount",
      "type": "double"
    },
    {
      "name": "occurredAt",
      "type": "long",
      "logicalType": "timestamp-millis"
    },
    {
      "name": "metadata",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {}
    },
    {
      "name": "cancelReason",
      "type": ["null", "string"],
      "default": null,
      "doc": "Optional: reason for cancellation"
    }
  ]
}
```

### 3.3 Avro Producer

```csharp
using Confluent.Kafka;
using Confluent.SchemaRegistry;
using Confluent.SchemaRegistry.Serdes;

// Configure Schema Registry connection
var schemaRegistryConfig = new SchemaRegistryConfig
{
    Url = "http://localhost:8081",
};

var producerConfig = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    Acks = Acks.All,
};

// Avro serializer configuration
var avroSerializerConfig = new AvroSerializerConfig
{
    // AUTO_REGISTER_SCHEMAS: Register schema automatically on first produce
    // Set to false in production (schemas should be pre-registered)
    AutoRegisterSchemas = true,
    SubjectNameStrategy = SubjectNameStrategy.TopicRecord, // {topic}-{record-type}
};

using var schemaRegistry = new CachedSchemaRegistryClient(schemaRegistryConfig);
using var producer = new ProducerBuilder<string, OrderEvent>(producerConfig)
    .SetValueSerializer(new AvroSerializer<OrderEvent>(schemaRegistry, avroSerializerConfig))
    .Build();

var orderEvent = new OrderEvent
{
    OrderId = "ORD-001",
    CustomerId = "CUST-042",
    Status = OrderStatus.PLACED,
    Amount = 299.99,
    OccurredAt = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
};

await producer.ProduceAsync("orders", new Message<string, OrderEvent>
{
    Key = orderEvent.OrderId,
    Value = orderEvent
});
```

### 3.4 Avro Consumer

```csharp
using var schemaRegistry = new CachedSchemaRegistryClient(schemaRegistryConfig);
using var consumer = new ConsumerBuilder<string, OrderEvent>(consumerConfig)
    .SetValueDeserializer(new AvroDeserializer<OrderEvent>(schemaRegistry).AsSyncOverAsync())
    .Build();

consumer.Subscribe("orders");

var result = consumer.Consume(token);
OrderEvent order = result.Message.Value; // Fully typed!
Console.WriteLine($"Order: {order.OrderId}, Amount: {order.Amount}");
```

### 3.5 Register Schema via REST API

```powershell
# Register a schema manually (before your producer runs)
$schema = Get-Content "schemas/order-event.avsc" -Raw
$body = @{ schema = $schema } | ConvertTo-Json

# Schema subject follows naming convention: {topic}-value or {topic}-key
Invoke-RestMethod `
  -Uri "http://localhost:8081/subjects/orders-value/versions" `
  -Method Post `
  -ContentType "application/vnd.schemaregistry.v1+json" `
  -Body $body

# List all registered schemas
Invoke-RestMethod -Uri "http://localhost:8081/subjects"

# Get a specific schema version
Invoke-RestMethod -Uri "http://localhost:8081/subjects/orders-value/versions/latest"

# Check if a schema is compatible with the existing one
$newSchema = Get-Content "schemas/order-event-v2.avsc" -Raw
$body = @{ schema = $newSchema } | ConvertTo-Json
Invoke-RestMethod `
  -Uri "http://localhost:8081/compatibility/subjects/orders-value/versions/latest" `
  -Method Post `
  -ContentType "application/vnd.schemaregistry.v1+json" `
  -Body $body
```

---

## 4. Schema Evolution Rules

Schema Registry enforces **compatibility** between versions. This prevents breaking changes.

### Compatibility Modes

| Mode | What It Allows |
|------|---------------|
| **BACKWARD** (default) | New schema can read old data. Add optional fields only. |
| **FORWARD** | Old schema can read new data. Remove only optional fields. |
| **FULL** | Both backward and forward compatible. Safest. |
| **NONE** | No compatibility check. Dangerous in production. |

### Safe vs Breaking Changes

```
✅ SAFE (Backward Compatible — default):
- Add a new OPTIONAL field with a default value
- Remove a field with a default value (FORWARD compatible too)
- Change a field type to a compatible type (int → long)

❌ BREAKING (Will be rejected by Schema Registry):
- Rename a field
- Change a field type incompatibly (string → int)
- Add a REQUIRED field without default
- Remove a REQUIRED field without default
- Change the record namespace
```

### Evolution Example

```json
// Version 1 — original
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount", "type": "double" }
  ]
}

// Version 2 — add optional field (SAFE ✅)
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "currency", "type": "string", "default": "USD" }  // ← default required!
  ]
}

// Version 3 — add nullable field (SAFE ✅)
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "currency", "type": "string", "default": "USD" },
    { "name": "discountCode", "type": ["null", "string"], "default": null }  // ← nullable
  ]
}
```

---

## 5. Protobuf Serialization

Protocol Buffers (Protobuf) is Google's binary serialization format. Fastest and most compact.

### Install Packages

```powershell
dotnet add package Confluent.SchemaRegistry.Serdes.Protobuf
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools  # For .proto code generation
```

### Define .proto File

```protobuf
// protos/order_event.proto
syntax = "proto3";

package com.mycompany.orders;
option csharp_namespace = "KafkaDemo.Shared.Protos";

import "google/protobuf/timestamp.proto";

message OrderEvent {
  string order_id = 1;
  string customer_id = 2;
  OrderStatus status = 3;
  double amount = 4;
  google.protobuf.Timestamp occurred_at = 5;
  repeated OrderItem items = 6;
  map<string, string> metadata = 7;
  optional string cancel_reason = 8;
}

message OrderItem {
  string product_id = 1;
  string product_name = 2;
  int32 quantity = 3;
  double unit_price = 4;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PLACED = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

### Configure .csproj for Code Gen

```xml
<!-- Add to your .csproj -->
<ItemGroup>
  <Protobuf Include="protos\order_event.proto" GrpcServices="None" />
</ItemGroup>
```

### Protobuf Producer

```csharp
using Confluent.SchemaRegistry.Serdes;

using var producer = new ProducerBuilder<string, OrderEvent>(producerConfig)
    .SetValueSerializer(new ProtobufSerializer<OrderEvent>(schemaRegistry))
    .Build();

var orderEvent = new OrderEvent
{
    OrderId = "ORD-001",
    CustomerId = "CUST-042",
    Status = OrderStatus.Placed,
    Amount = 299.99,
    OccurredAt = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTime(DateTime.UtcNow),
};

await producer.ProduceAsync("orders", new Message<string, OrderEvent>
{
    Key = orderEvent.OrderId,
    Value = orderEvent
});
```

---

## 6. Format Comparison

| Feature | String/JSON | Avro | Protobuf |
|---------|-------------|------|----------|
| **Human Readable** | ✅ Yes | ❌ Binary | ❌ Binary |
| **Schema Enforcement** | ❌ No | ✅ Via Registry | ✅ Via Registry |
| **Message Size** | Large | Small | Smallest |
| **Serialization Speed** | Medium | Fast | Fastest |
| **Schema Evolution** | Manual | Built-in | Built-in |
| **Language Support** | Universal | Good | Universal |
| **Debugging Ease** | Easy | Moderate | Hard |
| **Best For** | Dev/prototyping | Production, analytics | High-performance, multi-language |

### Recommendation for .NET Teams

```
Development / Internal tools:  JSON (easy to debug)
Production microservices:       Avro (good balance, Schema Registry support)
High-performance / polyglot:    Protobuf (if multiple languages)
```

---

## 7. CloudEvents Standard

CloudEvents is a CNCF standard for event metadata. Great for interoperability:

```csharp
// CloudEvents wrapper — adds standard metadata envelope
public record CloudEvent<T>
{
    // Spec fields (required)
    public string SpecVersion { get; init; } = "1.0";
    public string Id { get; init; } = Guid.NewGuid().ToString();
    public string Source { get; init; } = string.Empty;  // "/orders/service"
    public string Type { get; init; } = string.Empty;    // "com.mycompany.order.placed"
    public string Time { get; init; } = DateTime.UtcNow.ToString("O");
    public string DataContentType { get; init; } = "application/json";

    // Custom extensions
    public string? CorrelationId { get; init; }
    public string? TenantId { get; init; }

    // Actual event data
    public T? Data { get; init; }
}

// Usage in producer
var cloudEvent = new CloudEvent<OrderEvent>
{
    Source = "/services/order-service",
    Type = "com.mycompany.orders.placed.v1",
    CorrelationId = Activity.Current?.TraceId.ToString(),
    TenantId = tenantId,
    Data = orderEvent,
};

await producer.ProduceAsync("orders", new Message<string, string>
{
    Key = orderEvent.OrderId,
    Value = JsonSerializer.Serialize(cloudEvent)
});
```

---

## Summary

You now know:
- ✅ Manual JSON serialization (good for development)
- ✅ Custom typed JSON serializers/deserializers
- ✅ Avro serialization with Schema Registry (recommended for production)
- ✅ Schema evolution rules and compatibility modes
- ✅ Protobuf for high-performance scenarios
- ✅ CloudEvents standard for interoperability

**Next:** [08-ERROR-HANDLING.md](./08-ERROR-HANDLING.md) — Retry patterns and Dead Letter Queues
