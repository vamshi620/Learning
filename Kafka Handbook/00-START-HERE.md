# 📚 Kafka Handbook for .NET Teams
## Zero to Hero — Complete Learning Guide

---

## 🎯 Who Is This For?

This handbook is for **.NET developers** who are new to Apache Kafka and want to:
- Understand Kafka concepts from scratch
- Build real producers and consumers in C#
- Integrate Kafka into .NET microservices
- Deploy Kafka-based solutions on Azure
- Handle real-world production concerns

**No prior Kafka experience needed.** You need basic C# and some understanding of APIs.

---

## 📂 Handbook Structure

| File | Topic | Level |
|------|-------|-------|
| `00-START-HERE.md` | Overview & Learning Path | 🟢 Beginner |
| `01-KAFKA-FUNDAMENTALS.md` | What is Kafka? Core Concepts & Architecture | 🟢 Beginner |
| `02-KAFKA-SETUP.md` | Local Setup — Docker, CLI & First Topic | 🟢 Beginner |
| `03-DOTNET-KAFKA-BASICS.md` | First .NET Producer & Consumer | 🟢 Beginner |
| `04-PRODUCERS-DEEP-DIVE.md` | Advanced Producer Patterns in .NET | 🟡 Intermediate |
| `05-CONSUMERS-DEEP-DIVE.md` | Advanced Consumer Patterns & Consumer Groups | 🟡 Intermediate |
| `06-TOPICS-PARTITIONS-OFFSETS.md` | Topics, Partitions, Offsets — Deep Dive | 🟡 Intermediate |
| `07-SERIALIZATION.md` | JSON, Avro & Protobuf Serialization in .NET | 🟡 Intermediate |
| `08-ERROR-HANDLING.md` | Retry, Dead Letter Queue & Error Patterns | 🟡 Intermediate |
| `09-KAFKA-WITH-AZURE.md` | Azure Event Hubs for Kafka & AKS Deployment | 🔴 Advanced |
| `10-ADVANCED-PATTERNS.md` | CQRS, Saga, Outbox Pattern with Kafka | 🔴 Advanced |
| `11-MASSTRANSIT-KAFKA.md` | MassTransit + Kafka in .NET | 🔴 Advanced |
| `12-MONITORING-OBSERVABILITY.md` | Monitoring, Metrics & Alerting | 🔴 Advanced |
| `13-REAL-WORLD-PROJECT.md` | Full Project: Order Processing System | 🔴 Advanced |

---

## 🗺️ Learning Paths

### Path A — I'm Completely New to Kafka
```
01 → 02 → 03 → 04 → 05 → 06 → 07 → 08
```

### Path B — I Know Kafka, I want .NET Integration
```
03 → 04 → 05 → 07 → 08 → 11
```

### Path C — I'm going to Azure
```
01 → 03 → 09 → 10 → 12
```

### Path D — Full Project (Learn Everything)
```
01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13
```

---

## 🛠️ Prerequisites & Tools

### Required
- [.NET 8 SDK](https://dotnet.microsoft.com/download)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Visual Studio 2022](https://visualstudio.microsoft.com/) or [VS Code](https://code.visualstudio.com/)

### NuGet Packages Used in This Handbook
| Package | Purpose |
|---------|---------|
| `Confluent.Kafka` | Core Kafka client for .NET |
| `Confluent.SchemaRegistry` | Schema Registry client |
| `Confluent.SchemaRegistry.Serdes.Avro` | Avro serialization |
| `Confluent.SchemaRegistry.Serdes.Json` | JSON schema serialization |
| `MassTransit.Kafka` | Higher-level abstraction for Kafka |
| `MassTransit.RabbitMq` | For comparison examples |
| `Serilog.Sinks.Console` | Structured logging |
| `OpenTelemetry` | Observability |

### Verify Your Setup
```bash
# Check .NET
dotnet --version   # Should be 8.x or higher

# Check Docker
docker --version
docker compose version
```

---

## 💡 Key Mental Models Before You Start

### Kafka is NOT a Queue — It's a Log
Traditional message queues (RabbitMQ, Azure Service Bus) **delete** messages after consumption.
Kafka **keeps** messages for a configurable retention period. Multiple consumers can read the same message.

```
Traditional Queue:
Producer → [Queue] → Consumer (message deleted after read)

Kafka:
Producer → [Topic/Log] → Consumer A (reads at its own pace)
                       → Consumer B (reads independently)
                       → Consumer C (can replay from beginning)
```

### The Three Things You Always Think About
1. **Topic** — WHERE does the data go?
2. **Partition** — HOW is it distributed?
3. **Offset** — WHERE is my consumer in the stream?

---

## 🚀 Quick Start (5 minutes)

If you just want to see Kafka working immediately:

```bash
# 1. Start Kafka with Docker
docker run -d --name kafka \
  -p 9092:9092 \
  -e KAFKA_ENABLE_KRAFT=yes \
  -e KAFKA_CFG_NODE_ID=1 \
  -e KAFKA_CFG_PROCESS_ROLES=broker,controller \
  -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  bitnami/kafka:latest

# 2. Create a topic
docker exec kafka kafka-topics.sh \
  --create --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1

# 3. Then go to File 03 for your first .NET producer/consumer
```

---

## 📖 Conventions Used in This Handbook

- `📌 Key Concept` — Important definition to memorize
- `⚠️ Common Mistake` — Watch out for this!
- `💡 Pro Tip` — Best practice from real-world experience
- `🔥 Real World` — How this works in production
- `🧪 Exercise` — Hands-on task to reinforce learning

---

*Start with [01-KAFKA-FUNDAMENTALS.md](./01-KAFKA-FUNDAMENTALS.md)*
