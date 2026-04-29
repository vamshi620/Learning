# Kafka with Azure
## File 09: Azure Event Hubs for Kafka & AKS Deployment

---

## What You'll Learn

- Azure Event Hubs — the managed Kafka service on Azure
- Connecting .NET apps to Event Hubs using Kafka protocol
- Azure Event Hubs vs self-managed Kafka on AKS
- Deploying Kafka-based .NET apps to Azure Kubernetes Service (AKS)
- Security: Azure AD authentication, managed identity

**Time Required:** 60 minutes

---

## 1. Azure Event Hubs for Kafka

**Azure Event Hubs** provides a Kafka-compatible endpoint, meaning your existing `Confluent.Kafka` .NET code works with **zero code changes** — just change the connection string!

```
Your .NET App (Confluent.Kafka) → Azure Event Hubs
                                  (Kafka-compatible surface)
```

### What Event Hubs Maps To

| Kafka Concept | Azure Event Hubs Equivalent |
|--------------|----------------------------|
| Cluster | Event Hubs Namespace |
| Topic | Event Hub (within the namespace) |
| Partition | Partition |
| Consumer Group | Consumer Group |
| Broker | Managed by Azure |
| ZooKeeper | Managed by Azure |
| Schema Registry | Azure Schema Registry (built into Event Hubs Premium/Dedicated) |

### Tier Comparison

| Feature | Basic | Standard | Premium | Dedicated |
|---------|-------|----------|---------|-----------|
| Kafka Support | ❌ | ✅ | ✅ | ✅ |
| Retention | 1 day | 7 days | 90 days | Unlimited |
| Partitions | 32 | 32 | 100 | 2000+ |
| Schema Registry | ❌ | ❌ | ✅ | ✅ |
| VNet Integration | ❌ | ❌ | ✅ | ✅ |
| Monthly Cost | Cheap | ~$10+ | ~$650+ | ~$65K+ |

> 💡 **Pro Tip:** Start with **Standard** tier for dev/staging. Upgrade to **Premium** for production if you need schema registry, longer retention, or VNet isolation.

---

## 2. Create Event Hubs Namespace via Azure CLI

```powershell
# Login
az login

# Variables
$rg = "myproject-kafka-rg"
$location = "eastus"
$namespaceName = "myproject-kafka-ns"  # Must be globally unique
$eventhubName = "orders"               # Topic name

# Create resource group
az group create --name $rg --location $location

# Create Event Hubs Namespace (Standard tier for Kafka support)
az eventhubs namespace create `
  --name $namespaceName `
  --resource-group $rg `
  --location $location `
  --sku Standard `
  --enable-kafka true `
  --minimum-tls-version 1.2

# Create an Event Hub (= Kafka Topic)
az eventhubs eventhub create `
  --name $eventhubName `
  --namespace-name $namespaceName `
  --resource-group $rg `
  --partition-count 4 `
  --retention-time 7  # Days

# Create consumer group
az eventhubs eventhub consumer-group create `
  --eventhub-name $eventhubName `
  --consumer-group-name "order-service" `
  --namespace-name $namespaceName `
  --resource-group $rg

# Get connection string
az eventhubs namespace authorization-rule keys list `
  --name RootManageSharedAccessKey `
  --namespace-name $namespaceName `
  --resource-group $rg `
  --query primaryConnectionString `
  --output tsv
```

---

## 3. Connect .NET App to Event Hubs

**The magic:** Your `Confluent.Kafka` code stays the same. Only the config changes!

```csharp
// appsettings.json (for development with SAS key)
{
  "EventHubs": {
    "BootstrapServers": "myproject-kafka-ns.servicebus.windows.net:9093",
    "ConnectionString": "Endpoint=sb://myproject-kafka-ns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=YOUR_KEY_HERE"
  }
}
```

```csharp
// KafkaEventHubsConfig.cs — build Kafka config from Event Hubs connection string
public static class EventHubsKafkaConfig
{
    public static ProducerConfig GetProducerConfig(string connectionString, string bootstrapServers)
    {
        var saslPassword = BuildSaslPassword(connectionString);

        return new ProducerConfig
        {
            // Event Hubs Kafka endpoint
            BootstrapServers = bootstrapServers,

            // REQUIRED: TLS + SASL/PLAIN authentication
            SecurityProtocol = SecurityProtocol.SaslSsl,
            SaslMechanism = SaslMechanism.Plain,
            SaslUsername = "$ConnectionString",  // Literal string — always this value
            SaslPassword = connectionString,      // Full SAS connection string as password

            // Reliability
            Acks = Acks.All,
            EnableIdempotence = true,
            MessageSendMaxRetries = 5,
        };
    }

    public static ConsumerConfig GetConsumerConfig(
        string connectionString, string bootstrapServers, string groupId)
    {
        return new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            SecurityProtocol = SecurityProtocol.SaslSsl,
            SaslMechanism = SaslMechanism.Plain,
            SaslUsername = "$ConnectionString",
            SaslPassword = connectionString,

            GroupId = groupId,
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,
        };
    }
}
```

### Producer with Event Hubs

```csharp
// Program.cs
var connectionString = configuration["EventHubs:ConnectionString"];
var bootstrapServers = configuration["EventHubs:BootstrapServers"];

var producerConfig = EventHubsKafkaConfig.GetProducerConfig(connectionString, bootstrapServers);

using var producer = new ProducerBuilder<string, string>(producerConfig).Build();

// Produce exactly the same way as with local Kafka!
await producer.ProduceAsync("orders", new Message<string, string>
{
    Key = order.OrderId,
    Value = JsonSerializer.Serialize(order)
});
```

### Consumer with Event Hubs

```csharp
var consumerConfig = EventHubsKafkaConfig.GetConsumerConfig(
    connectionString, bootstrapServers, "order-service");

using var consumer = new ConsumerBuilder<string, string>(consumerConfig).Build();
consumer.Subscribe("orders");  // Same as local Kafka!

var result = consumer.Consume(token);
```

---

## 4. Managed Identity Authentication (Production Best Practice)

Instead of SAS keys, use **Managed Identity** — no secrets to manage!

```powershell
# Enable System-Assigned Managed Identity on your App Service / AKS
az webapp identity assign --name myapp --resource-group myrg

# Or for AKS with workload identity:
az aks update --name myaks --resource-group myrg --enable-workload-identity

# Grant the identity access to Event Hubs
az role assignment create `
  --role "Azure Event Hubs Data Owner" `
  --assignee <managed-identity-client-id> `
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.EventHub/namespaces/<namespace>
```

```csharp
// Using DefaultAzureCredential with Event Hubs Kafka
// Requires: Azure.Identity and Microsoft.Azure.EventHubs packages
using Azure.Identity;
using Azure.Messaging.EventHubs;

// For Kafka protocol with Managed Identity:
// Event Hubs requires OAuth token as SASL password

public class ManagedIdentityKafkaConfig
{
    private readonly TokenCredential _credential;

    public ManagedIdentityKafkaConfig()
    {
        // DefaultAzureCredential tries: MI → VS → CLI → env vars
        _credential = new DefaultAzureCredential();
    }

    public async Task<string> GetAccessTokenAsync()
    {
        var tokenRequest = await _credential.GetTokenAsync(
            new Azure.Core.TokenRequestContext(
                new[] { "https://eventhubs.azure.net/.default" }));
        return tokenRequest.Token;
    }

    public async Task<ProducerConfig> GetProducerConfigAsync(string bootstrapServers)
    {
        var token = await GetAccessTokenAsync();

        return new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            SecurityProtocol = SecurityProtocol.SaslSsl,
            SaslMechanism = SaslMechanism.OAuthBearer,
            SaslOauthbearerConfig = $"principal={token}",  // token as config
        };
    }
}
```

> 💡 **Pro Tip:** OAuth token with `SaslMechanism.OAuthBearer` works better with Managed Identity. Tokens expire after ~1 hour, so implement token refresh in production.

---

## 5. Azure Schema Registry

Event Hubs Premium/Dedicated includes Azure Schema Registry — use it instead of running your own.

```powershell
dotnet add package Microsoft.Azure.Data.SchemaRegistry.ApacheAvro
dotnet add package Azure.Data.SchemaRegistry
```

```csharp
using Azure.Data.SchemaRegistry;
using Microsoft.Azure.Data.SchemaRegistry.ApacheAvro;
using Azure.Identity;

// Configure Azure Schema Registry serializer
var schemaRegistryClient = new SchemaRegistryClient(
    fullyQualifiedNamespace: "myproject-kafka-ns.servicebus.windows.net",
    credential: new DefaultAzureCredential());

var serializer = new SchemaRegistryAvroSerializer(
    schemaRegistryClient,
    groupName: "my-schema-group",  // Schema group in Azure Schema Registry
    new SchemaRegistryAvroSerializerOptions { AutoRegisterSchemas = true });

// Produce with Azure Schema Registry Avro serialization
using var producer = new ProducerBuilder<string, OrderEvent>(producerConfig)
    .SetValueSerializer(await serializer.CreateAvroSerializerAsync<OrderEvent>())
    .Build();
```

---

## 6. Self-Managed Kafka on AKS

For full control over Kafka (features, configuration, cost), run Kafka on AKS using the **Strimzi operator**.

### Deploy Strimzi Kafka Operator

```powershell
# Create namespace
kubectl create namespace kafka

# Install Strimzi operator via Helm
helm repo add strimzi https://strimzi.io/charts/
helm repo update

helm install strimzi-operator strimzi/strimzi-kafka-operator `
  --namespace kafka `
  --set watchNamespaces="{kafka}"
```

### Deploy Kafka Cluster on AKS

```yaml
# kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: myproject-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 3          # 3 broker pods
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: loadbalancer  # Expose externally via Azure Load Balancer
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      log.retention.hours: 168   # 7 days
      log.segment.bytes: 1073741824  # 1 GB
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          class: managed-premium  # Azure Premium SSD
          deleteClaim: false
    resources:
      requests:
        memory: 2Gi
        cpu: "500m"
      limits:
        memory: 4Gi
        cpu: "2000m"
  
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: managed-premium
    resources:
      requests:
        memory: 512Mi
        cpu: "250m"
      limits:
        memory: 1Gi
        cpu: "500m"

  entityOperator:
    topicOperator: {}   # Manages KafkaTopic CRDs
    userOperator: {}    # Manages KafkaUser CRDs
```

### Define Topics via CRD (GitOps Friendly!)

```yaml
# topics.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  namespace: kafka
  labels:
    strimzi.io/cluster: myproject-kafka
spec:
  partitions: 6
  replicas: 3
  config:
    retention.ms: 604800000    # 7 days
    cleanup.policy: delete
    min.insync.replicas: "2"
    max.message.bytes: "10485760"  # 10 MB
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders-dlq
  namespace: kafka
  labels:
    strimzi.io/cluster: myproject-kafka
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 2592000000   # 30 days — keep DLQ longer
    cleanup.policy: delete
```

```bash
# Apply
kubectl apply -f kafka-cluster.yaml
kubectl apply -f topics.yaml

# Watch cluster come up
kubectl get pods -n kafka -w

# Get external bootstrap address
kubectl get kafka myproject-kafka -n kafka -o jsonpath='{.status.listeners[?(@.name=="external")].bootstrapServers}'
```

---

## 7. Deploy .NET Kafka Consumer to AKS

```yaml
# k8s/consumer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-consumer
  namespace: default
spec:
  replicas: 3  # 3 instances = 3 partitions consumed in parallel (if 3+ partitions)
  selector:
    matchLabels:
      app: order-consumer
  template:
    metadata:
      labels:
        app: order-consumer
    spec:
      containers:
        - name: order-consumer
          image: myregistry.azurecr.io/order-consumer:latest
          env:
            - name: Kafka__BootstrapServers
              valueFrom:
                secretKeyRef:
                  name: kafka-config
                  key: bootstrap-servers
            - name: Kafka__GroupId
              value: "order-service"
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
---
# kafka-config secret
apiVersion: v1
kind: Secret
metadata:
  name: kafka-config
type: Opaque
stringData:
  bootstrap-servers: "myproject-kafka-kafka-bootstrap.kafka.svc.cluster.local:9092"
```

```powershell
# Deploy
kubectl apply -f k8s/consumer-deployment.yaml

# Scale consumers (add more instances)
kubectl scale deployment order-consumer --replicas=6

# View logs from all consumer instances
kubectl logs -l app=order-consumer --follow
```

---

## 8. Event Hubs vs AKS Kafka — Decision Guide

| Consider | Azure Event Hubs | Kafka on AKS |
|----------|-----------------|-------------|
| **Team Kafka expertise** | Low | High needed |
| **Operational overhead** | None (managed) | High |
| **Cost (< 10K msg/sec)** | Cheaper | More expensive |
| **Cost (> 100K msg/sec)** | Expensive | Cheaper at scale |
| **Retention** | Max 90 days (Premium) | Unlimited |
| **Latency** | ~5ms | ~1ms |
| **Custom Kafka config** | ❌ Limited | ✅ Full control |
| **Multi-region** | ✅ Geo-DR built-in | Manual setup |
| **Connector ecosystem** | Limited | Full Kafka Connect |
| **Compliance/data locality** | Easy | Custom |

**Recommendation for most .NET teams:**  
👉 Start with **Azure Event Hubs** (Standard/Premium).  
👉 Migrate to **Kafka on AKS** only if you hit Event Hubs limits or need Kafka-specific features.

---

## Summary

You now know:
- ✅ Azure Event Hubs as a managed Kafka service
- ✅ Connecting .NET Confluent.Kafka to Event Hubs (SAS key + Managed Identity)
- ✅ Azure Schema Registry with Avro
- ✅ Self-managed Kafka on AKS with Strimzi operator
- ✅ Deploying .NET Kafka consumers to AKS
- ✅ When to use Event Hubs vs. AKS Kafka

**Next:** [10-ADVANCED-PATTERNS.md](./10-ADVANCED-PATTERNS.md) — CQRS, Saga, and Outbox patterns with Kafka
