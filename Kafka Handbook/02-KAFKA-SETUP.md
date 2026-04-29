# Kafka Setup
## File 02: Local Setup with Docker — Kafka Running in 5 Minutes

---

## What You'll Do

- Run Kafka locally using Docker Compose
- Create topics using CLI
- Use Kafka UI to visualize your cluster
- Test sending and receiving messages
- Understand the configuration options

**Time Required:** 20-30 minutes

---

## 1. Prerequisites Check

```powershell
# Check Docker is running
docker --version
# Expected: Docker version 24.x or higher

docker compose version
# Expected: Docker Compose version v2.x
```

---

## 2. Docker Compose Setup (Recommended)

Create a file `docker-compose.yml` in a folder like `C:\kafka-local\`:

```yaml
# docker-compose.yml
# Kafka in KRaft mode (no ZooKeeper needed!)
# Includes Kafka UI for easy visualization

version: '3.8'

services:
  kafka:
    image: bitnami/kafka:3.7
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      # KRaft settings (no ZooKeeper)
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      
      # Listeners
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      
      # Cluster ID (generate once, keep consistent)
      - KAFKA_KRAFT_CLUSTER_ID=MkU3OEVBNTcwNTJENDM2Qk
      
      # Topic defaults
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false
      - KAFKA_CFG_DEFAULT_REPLICATION_FACTOR=1
      - KAFKA_CFG_NUM_PARTITIONS=3
      
      # Message size limits
      - KAFKA_CFG_MESSAGE_MAX_BYTES=10485760        # 10 MB
      - KAFKA_CFG_REPLICA_FETCH_MAX_BYTES=10485760  # 10 MB
      
    volumes:
      - kafka_data:/bitnami/kafka
    healthcheck:
      test: ["CMD", "kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 5s
      retries: 5

  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    container_name: schema-registry
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    depends_on:
      kafka:
        condition: service_healthy

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    depends_on:
      - kafka
      - schema-registry

volumes:
  kafka_data:
    driver: local
```

### Start Everything

```powershell
# Navigate to your docker-compose folder
cd C:\kafka-local

# Start all services in background
docker compose up -d

# Watch logs (optional)
docker compose logs -f kafka

# Verify everything is running
docker compose ps
```

**Expected output:**
```
NAME              IMAGE                                    STATUS
kafka             bitnami/kafka:3.7                        Up (healthy)
schema-registry   confluentinc/cp-schema-registry:7.6.0   Up
kafka-ui          provectuslabs/kafka-ui:latest            Up
```

### Open Kafka UI
Navigate to `http://localhost:8080` in your browser.
You should see your cluster with:
- Brokers: 1
- Topics: (none yet)

---

## 3. Kafka CLI Commands

All CLI tools run inside the Kafka container. Use this pattern:

```powershell
docker exec kafka kafka-{tool}.sh --bootstrap-server localhost:9092 [options]
```

### 3.1 Topics

```powershell
# List all topics
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# Create a topic
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1

# Create with retention (7 days = 604800000 ms)
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic audit-logs \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete

# Describe a topic (see partitions, replication, config)
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic orders

# Delete a topic
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic orders

# Alter partition count (can only increase, never decrease!)
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --topic orders \
  --partitions 6
```

**Sample `--describe` output:**
```
Topic: orders
PartitionCount: 3
ReplicationFactor: 1
Configs: 

Topic: orders  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
Topic: orders  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
Topic: orders  Partition: 2  Leader: 1  Replicas: 1  Isr: 1
```

### 3.2 Console Producer (Quick Testing)

```powershell
# Start an interactive producer
docker exec -it kafka kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders

# Type messages line by line, press Enter to send:
> {"orderId": "001", "amount": 99.99}
> {"orderId": "002", "amount": 149.50}
> Ctrl+C to exit

# With keys (format: key:value using key.separator)
docker exec -it kafka kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --property "parse.key=true" \
  --property "key.separator=:"

# Type: key:value
> order-001:{"orderId": "001", "amount": 99.99}
```

### 3.3 Console Consumer (Quick Testing)

```powershell
# Read new messages (from now on)
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders

# Read ALL messages from beginning
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning

# Read with keys and metadata shown
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true \
  --property print.timestamp=true

# As part of a consumer group
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --group my-consumer-group \
  --from-beginning
```

### 3.4 Consumer Groups

```powershell
# List all consumer groups
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list

# Describe a group (see offsets and LAG)
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group my-consumer-group

# Reset offsets (reprocess from beginning)
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic orders \
  --reset-offsets \
  --to-earliest \
  --execute

# Reset offsets to specific time
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic orders \
  --reset-offsets \
  --to-datetime 2024-01-15T00:00:00.000 \
  --execute
```

**Sample `--describe` output (pay attention to LAG):**
```
GROUP                TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
my-consumer-group    orders  0         42              42              0    consumer-1
my-consumer-group    orders  1         38              40              2    consumer-1   ← LAG = 2 messages behind!
my-consumer-group    orders  2         55              55              0    consumer-2
```

> 📌 **Key Concept:** **LAG** = how many messages the consumer is behind. LAG should trend toward 0 in a healthy system. High LAG = consumer is slow or stuck.

---

## 4. Create Topics for This Handbook

Let's create all the topics we'll use through this handbook:

```powershell
# Create a PowerShell script to set up all topics

$topics = @(
    @{ name = "orders"; partitions = 3 },
    @{ name = "payments"; partitions = 3 },
    @{ name = "inventory"; partitions = 3 },
    @{ name = "notifications"; partitions = 2 },
    @{ name = "audit-logs"; partitions = 6 },
    @{ name = "dead-letter-queue"; partitions = 3 }
)

foreach ($topic in $topics) {
    Write-Host "Creating topic: $($topic.name)..."
    docker exec kafka kafka-topics.sh `
        --bootstrap-server localhost:9092 `
        --create `
        --topic $($topic.name) `
        --partitions $($topic.partitions) `
        --replication-factor 1 `
        --if-not-exists
}

Write-Host "All topics created!"
docker exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

## 5. Kafka UI Walkthrough

Open `http://localhost:8080` — you should see the Kafka UI dashboard.

### Dashboard Tabs
| Tab | What You Can Do |
|-----|----------------|
| **Dashboard** | Cluster overview, broker health |
| **Brokers** | Broker details, disk usage |
| **Topics** | Browse topics, see partitions |
| **Consumers** | Consumer groups and LAG |
| **Schema Registry** | Manage Avro/JSON schemas |

### Browse Messages in Kafka UI
1. Click **Topics** → Select `orders`
2. Click the **Messages** tab
3. Use **Produce Message** to send a test message from the UI
4. See messages with key, value, offset, partition

---

## 6. Useful Docker Commands

```powershell
# Stop everything (keeps data)
docker compose stop

# Start again (data preserved)
docker compose start

# Stop and REMOVE everything (fresh start)
docker compose down -v

# Check Kafka logs
docker compose logs kafka --tail=50

# Enter Kafka container shell
docker exec -it kafka bash

# Check Kafka configuration
docker exec kafka cat /opt/bitnami/kafka/config/kraft/server.properties
```

---

## 7. Common Issues & Fixes

### Issue: Cannot connect to Kafka from .NET
```
Error: localhost:9092 is not a valid bootstrap server
```
**Fix:** Ensure `KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092` (not kafka:9092).
From your .NET code on the host machine, use `localhost:9092`.
From inside Docker containers, use `kafka:9092`.

### Issue: Docker container unhealthy
```powershell
# Check what's wrong
docker compose logs kafka --tail=100

# Try restarting
docker compose restart kafka
```

### Issue: "No space left on device"
```powershell
# Clean up Docker disk space
docker system prune -f
docker volume prune -f
```

### Issue: Port 9092 already in use
```powershell
# Find what's using the port
netstat -ano | findstr :9092

# Kill the process or change the port in docker-compose.yml
# ports:
#   - "9093:9092"  ← use 9093 externally
```

---

## 8. Environment Summary

After this setup, you have:

| Service | URL | Purpose |
|---------|-----|---------|
| **Kafka Broker** | `localhost:9092` | Your .NET apps connect here |
| **Schema Registry** | `http://localhost:8081` | Avro/JSON schema management |
| **Kafka UI** | `http://localhost:8080` | Visual cluster management |

---

## 🧪 Exercise

1. Create a topic called `exercise-topic` with 4 partitions
2. Open two terminal windows
3. In terminal 1: Start the console consumer (from-beginning)
4. In terminal 2: Start the console producer
5. Send 10 messages from the producer
6. Observe them appearing in the consumer
7. Check the Kafka UI to see messages distributed across partitions

```powershell
# Terminal 1 - Consumer
docker exec -it kafka kafka-console-consumer.sh `
  --bootstrap-server localhost:9092 `
  --topic exercise-topic `
  --from-beginning `
  --property print.partition=true `
  --property print.offset=true

# Terminal 2 - Producer (run separately)
docker exec -it kafka kafka-console-producer.sh `
  --bootstrap-server localhost:9092 `
  --topic exercise-topic
```

---

## Summary

You now have:
- ✅ Kafka running locally in Docker
- ✅ Kafka UI at `http://localhost:8080`
- ✅ Schema Registry at `http://localhost:8081`
- ✅ Know all essential CLI commands
- ✅ Topics created for the handbook exercises

**Next:** [03-DOTNET-KAFKA-BASICS.md](./03-DOTNET-KAFKA-BASICS.md) — Write your first .NET Kafka producer and consumer
