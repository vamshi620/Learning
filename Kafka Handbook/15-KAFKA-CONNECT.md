# Kafka Connect Overview
## File 15: Data Integration Without Code

---

## What You'll Learn

- What Kafka Connect is and why it exists
- Source vs. Sink connectors
- Change Data Capture (CDC) with Debezium
- Why Kafka Connect is often better than custom Outbox relays

**Time Required:** 20 minutes

---

## 1. What is Kafka Connect?

Kafka Connect is a free, open-source tool included with Apache Kafka. It is a separate JVM process (or cluster of processes) designed to **reliably move data between Kafka and other systems without writing any code.**

Instead of writing a .NET application to poll a database and publish to Kafka, you configure a Connector using JSON, and Kafka Connect handles the rest.

```
[ Database ] ----(Source Connector)----> [ Kafka ] ----(Sink Connector)----> [ Elasticsearch ]
                                             |
                                      [ .NET Consumer ]
```

---

## 2. Source Connectors (Data INTO Kafka)

A **Source Connector** ingests data from an external system into Kafka topics.

### The "Custom Outbox" Problem
In [File 10](./10-ADVANCED-PATTERNS.md), we built an `OutboxRelayService` in C#. It polled an EF Core table every second and called `producer.ProduceAsync()`.
This works, but:
1. Polling a database table continuously creates DB load.
2. If the app crashes midway, you have to handle complex retry and locking logic.
3. You have to write and maintain this boilerplate in every microservice.

### The Connect Solution: Debezium (CDC)
**Change Data Capture (CDC)** reads the database's internal transaction log (e.g., SQL Server Transaction Log, PostgreSQL WAL) instead of running `SELECT` queries.

**Debezium** is the most popular suite of source connectors for CDC.

```
SQL Server (Outbox Table) -> [ Debezium SQL Server Source Connector ] -> Kafka Topic
```

**Benefits:**
- **Zero Polling Load:** It listens to the transaction log. It pushes events instantly.
- **No Missed Events:** Even if a row is inserted and deleted in milliseconds, it's captured in the log.
- **Language Agnostic:** No C# code required. Configuration is just a JSON POST to the Connect REST API.

**Example Configuration (POST to Connect API):**
```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "database.hostname": "sqlserver",
    "database.port": "1433",
    "database.user": "sa",
    "database.password": "Secret123!",
    "database.dbname": "OrderDB",
    "table.include.list": "dbo.OutboxMessages",
    "topic.prefix": "ecommerce"
  }
}
```

---

## 3. Sink Connectors (Data OUT OF Kafka)

A **Sink Connector** takes data from Kafka topics and pushes it to external systems.

### Common Use Cases:
- **Search:** Pushing `orders` events into Elasticsearch / OpenSearch for fast text searching.
- **Data Lake / Analytics:** Dumping Kafka topics into Azure Blob Storage, AWS S3, or Snowflake for batch analytics.
- **Caching:** Updating a Redis cache whenever a record changes.

Rather than writing a C# `BackgroundService` consumer just to take an event and `INSERT` it into a database, you use a **JDBC Sink Connector**.

---

## 4. When to Use Connect vs .NET Code

### Use .NET Code (Consumers/Producers) When:
- You need to execute **Business Logic** (e.g., "If order > $1000, send an email").
- You are transforming data significantly.
- You are implementing the Saga pattern.

### Use Kafka Connect When:
- You are just moving data from Point A to Point B without changing it.
- You want to stream database changes (CDC) reliably.
- You are archiving data to a data lake.

**Time Required:** 20 minutes

---

## 5. Running Kafka Connect

Kafka Connect is deployed as its own cluster, separate from your brokers.

In a Docker Compose environment, it looks like this:

```yaml
  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    depends_on:
      - kafka
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'kafka:9092'
      CONNECT_REST_ADVERTISED_HOST_NAME: 'kafka-connect'
      CONNECT_GROUP_ID: 'compose-connect-group'
      CONNECT_CONFIG_STORAGE_TOPIC: 'connect-configs'
      CONNECT_OFFSET_STORAGE_TOPIC: 'connect-offsets'
      CONNECT_STATUS_STORAGE_TOPIC: 'connect-status'
      # Path where you install connector plugins (like Debezium)
      CONNECT_PLUGIN_PATH: '/usr/share/java,/usr/share/confluent-hub-components'
```

Once running, you interact with it entirely via its REST API on port `8083`.

---

## Summary

You now know:
- ✅ **Kafka Connect** is an ecosystem tool for moving data without code.
- ✅ **Source Connectors** pull data into Kafka (e.g., Debezium for database CDC).
- ✅ **Sink Connectors** push data out of Kafka (e.g., dumping to Azure Blob Storage or Elasticsearch).
- ✅ Connect replaces custom C# relay code when the only goal is data movement.

**Next:** [16-TESTING-KAFKA.md](./16-TESTING-KAFKA.md) — Unit testing and integration testing Kafka in .NET.
