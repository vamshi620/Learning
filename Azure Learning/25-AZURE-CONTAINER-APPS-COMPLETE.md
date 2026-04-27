# Azure Container Apps — Complete Guide
## Document 25: Serverless Containers, KEDA Scaling & Dapr Integration

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Container Apps vs AKS, KEDA auto-scaling, Dapr service invocation, revision management, Ingress

---

## TABLE OF CONTENTS

1. [What is Azure Container Apps?](#1-what-is-container-apps)
2. [Container Apps vs AKS vs ACI](#2-comparison)
3. [Architecture & Concepts](#3-architecture)
4. [Creating Container Apps](#4-creating)
5. [Ingress & Traffic Splitting](#5-ingress)
6. [KEDA Auto-Scaling](#6-keda)
7. [Dapr Integration](#7-dapr)
8. [Managed Identity & Secrets](#8-security)
9. [CI/CD Deployment](#9-cicd)
10. [Jobs (Background Tasks)](#10-jobs)
11. [Observability](#11-observability)
12. [Best Practices](#12-best-practices)

---

## 1. What is Azure Container Apps?

Azure Container Apps is a **serverless container platform** that lets you run containerized applications without managing Kubernetes infrastructure. It's built on top of Kubernetes, KEDA, and Dapr but abstracts away all the complexity.

```
You Manage:                      Azure Manages:
─────────────                    ──────────────
✅ Application code              ✅ Kubernetes cluster
✅ Container image               ✅ Node provisioning
✅ Scaling rules                 ✅ OS patching
✅ Ingress configuration         ✅ Load balancing
✅ Environment variables         ✅ Certificate management
                                 ✅ KEDA controllers
                                 ✅ Dapr sidecars
                                 ✅ Networking (Envoy)
```

---

## 2. Container Apps vs AKS vs ACI {#2-comparison}

| Feature | Container Apps | AKS | ACI |
|---------|---------------|-----|-----|
| **Abstraction** | Serverless containers | Managed Kubernetes | Single container instance |
| **K8s knowledge needed** | None | Yes (significant) | None |
| **Auto-scaling** | KEDA (built-in) | HPA + KEDA (configure yourself) | None |
| **Scale to zero** | ✅ | ❌ (min 1 node) | N/A |
| **Ingress** | Built-in (Envoy) | Install yourself (NGINX/Traefik) | Public IP or VNet |
| **Service mesh** | Dapr (built-in) | Install yourself (Istio/Linkerd) | None |
| **Jobs/batch** | ✅ Container Apps Jobs | ✅ K8s Jobs/CronJobs | ✅ (task containers) |
| **VNet integration** | ✅ | ✅ | ✅ |
| **Custom domains + TLS** | ✅ (managed certs) | ✅ (cert-manager) | ❌ |
| **GPU support** | ✅ (preview) | ✅ | ✅ |
| **Cost model** | Pay per vCPU/memory/sec | Pay for VM nodes | Pay per vCPU/memory/sec |
| **Best for** | Microservices, APIs, background jobs | Complex K8s workloads, full control | Short-lived tasks, CI/CD agents |

**Decision Guide:**
```
Need scale-to-zero? → Container Apps
Need full K8s API access? → AKS
Need simple one-off container? → ACI
Need event-driven scaling? → Container Apps (KEDA built-in)
Need service mesh? → Container Apps (Dapr) or AKS (Istio)
Need GPU workloads at scale? → AKS
```

---

## 3. Architecture & Concepts {#3-architecture}

```
┌─────────────────── Container Apps Environment ──────────────────┐
│  (Shared VNet, Log Analytics, Dapr components)                   │
│                                                                   │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │  Container App: api  │  │  Container App: web  │             │
│  │  ┌─────────────────┐ │  │  ┌─────────────────┐ │             │
│  │  │ Revision 1 (70%)│ │  │  │ Revision 1      │ │             │
│  │  │ image:v1.0      │ │  │  │ image:latest    │ │             │
│  │  │ replicas: 1-10  │ │  │  │ replicas: 2-20  │ │             │
│  │  ├─────────────────┤ │  │  └─────────────────┘ │             │
│  │  │ Revision 2 (30%)│ │  │  Ingress: external   │             │
│  │  │ image:v1.1      │ │  │  Domain: web.myapp   │             │
│  │  │ replicas: 1-5   │ │  └──────────────────────┘             │
│  │  └─────────────────┘ │                                        │
│  │  Ingress: internal   │  ┌──────────────────────┐             │
│  └──────────────────────┘  │  Container App Job   │             │
│                              │  (batch/scheduled)  │             │
│  Dapr Sidecar ──► State     │  image:worker        │             │
│  Dapr Sidecar ──► PubSub    └──────────────────────┘             │
└───────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Environment** | Shared boundary for container apps (like a K8s cluster) |
| **Container App** | A running application (one or more containers) |
| **Revision** | An immutable snapshot of a container app version |
| **Replica** | An instance of a revision (like a K8s pod) |
| **Ingress** | HTTP/TCP entry point (external or internal) |
| **Dapr** | Built-in service-to-service invocation, state, pub/sub |
| **KEDA** | Event-driven auto-scaling (HTTP, queue depth, custom) |

---

## 4. Creating Container Apps {#4-creating}

```bash
# Install the extension
az extension add --name containerapp --upgrade

# Register provider
az provider register --namespace Microsoft.App

# Create environment
az containerapp env create \
  --name myenv \
  --resource-group myRG \
  --location eastus

# Create a container app from ACR image
az containerapp create \
  --name api \
  --resource-group myRG \
  --environment myenv \
  --image myacr.azurecr.io/api:v1.0 \
  --registry-server myacr.azurecr.io \
  --registry-identity system \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --env-vars "ASPNETCORE_ENVIRONMENT=Production" \
  --secrets "db-conn=<connection-string>" \
  --query properties.configuration.ingress.fqdn -o tsv

# Create from a public image (quick start)
az containerapp create \
  --name web \
  --resource-group myRG \
  --environment myenv \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 5
```

---

## 5. Ingress & Traffic Splitting {#5-ingress}

### Ingress Types

```
External Ingress: https://api.happyocean-abc123.eastus.azurecontainerapps.io
  → Accessible from internet + within environment

Internal Ingress: https://api.internal.happyocean-abc123.eastus.azurecontainerapps.io
  → Accessible only within the environment
```

### Traffic Splitting (Blue/Green & Canary)

```bash
# Deploy new revision
az containerapp update \
  --name api \
  --resource-group myRG \
  --image myacr.azurecr.io/api:v1.1

# Split traffic: 80% old, 20% new (canary)
az containerapp ingress traffic set \
  --name api \
  --resource-group myRG \
  --revision-weight api--rev1=80 api--rev2=20

# Promote new revision to 100%
az containerapp ingress traffic set \
  --name api \
  --resource-group myRG \
  --revision-weight api--rev2=100

# Rollback to previous revision
az containerapp ingress traffic set \
  --name api \
  --resource-group myRG \
  --revision-weight api--rev1=100
```

### Custom Domains

```bash
# Add custom domain with managed certificate
az containerapp hostname add \
  --name api \
  --resource-group myRG \
  --hostname api.contoso.com

az containerapp hostname bind \
  --name api \
  --resource-group myRG \
  --hostname api.contoso.com \
  --environment myenv \
  --validation-method CNAME
```

---

## 6. KEDA Auto-Scaling {#6-keda}

Container Apps uses KEDA (Kubernetes Event-Driven Autoscaling) to scale based on various event sources.

### HTTP Scaling (Default)

```bash
# Scale based on concurrent HTTP requests
az containerapp create \
  --name api \
  --resource-group myRG \
  --environment myenv \
  --image myacr.azurecr.io/api:latest \
  --min-replicas 0 \
  --max-replicas 30 \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50
# → Scale up when > 50 concurrent requests per replica
# → Scale to zero when no traffic
```

### Queue-Based Scaling (Service Bus)

```bash
# Scale based on Service Bus queue depth
az containerapp create \
  --name order-processor \
  --resource-group myRG \
  --environment myenv \
  --image myacr.azurecr.io/processor:latest \
  --min-replicas 0 \
  --max-replicas 20 \
  --scale-rule-name queue-rule \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata \
    queueName=orders \
    namespace=myservicebus \
    messageCount=10 \
  --scale-rule-auth \
    connection=servicebus-connection-secret
# → 1 replica per 10 messages in queue
# → Scale to zero when queue is empty
```

### Custom Scaling Rules

```yaml
# YAML definition for scale rules
properties:
  template:
    scale:
      minReplicas: 0
      maxReplicas: 30
      rules:
        - name: http-rule
          http:
            metadata:
              concurrentRequests: "50"
        - name: cpu-rule
          custom:
            type: cpu
            metadata:
              type: Utilization
              value: "70"
        - name: cron-rule
          custom:
            type: cron
            metadata:
              timezone: "America/New_York"
              start: "0 8 * * 1-5"    # 8 AM weekdays
              end: "0 18 * * 1-5"     # 6 PM weekdays
              desiredReplicas: "5"
```

---

## 7. Dapr Integration {#7-dapr}

Dapr (Distributed Application Runtime) provides building blocks for microservices, injected as a sidecar.

```
┌─────────────── Container App ──────────────────┐
│  ┌──────────────┐    ┌──────────────────────┐  │
│  │ Your App     │◄──►│ Dapr Sidecar         │  │
│  │ (port 8080)  │    │ (localhost:3500)      │  │
│  │              │    │                      │  │
│  │ HTTP/gRPC    │    │ • Service Invocation  │  │
│  │ calls to     │    │ • State Management    │  │
│  │ localhost     │    │ • Pub/Sub Messaging   │  │
│  │              │    │ • Bindings            │  │
│  └──────────────┘    │ • Secrets             │  │
│                      └──────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Enable Dapr

```bash
# Enable Dapr on a container app
az containerapp create \
  --name api \
  --resource-group myRG \
  --environment myenv \
  --image myacr.azurecr.io/api:latest \
  --enable-dapr true \
  --dapr-app-id api \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

### Dapr Service Invocation

```csharp
// Call another service through Dapr sidecar
// No need to know the target's URL, port, or IP
var httpClient = new HttpClient();
var response = await httpClient.GetAsync(
    "http://localhost:3500/v1.0/invoke/order-service/method/orders/123"
);
// Dapr routes to "order-service" container app automatically
```

### Dapr State Store Component

```bash
# Create Dapr state store component (using Azure Blob Storage)
az containerapp env dapr-component set \
  --name myenv \
  --resource-group myRG \
  --dapr-component-name statestore \
  --yaml statestore.yaml
```

```yaml
# statestore.yaml
componentType: state.azure.blobstorage
version: v1
metadata:
  - name: accountName
    value: mystorageaccount
  - name: containerName
    value: dapr-state
  - name: azureClientId
    value: "<managed-identity-client-id>"
scopes:
  - api
  - order-service
```

### Dapr Pub/Sub Component

```yaml
# pubsub.yaml (using Azure Service Bus)
componentType: pubsub.azure.servicebus.topics
version: v1
metadata:
  - name: namespaceName
    value: myservicebus.servicebus.windows.net
  - name: consumerID
    value: "order-processor"
scopes:
  - api
  - order-processor
```

```csharp
// Publish event through Dapr
await httpClient.PostAsJsonAsync(
    "http://localhost:3500/v1.0/publish/pubsub/orders",
    new { orderId = 123, total = 99.99 }
);

// Subscribe to events (ASP.NET endpoint)
[HttpPost("/orders")]
public async Task<IActionResult> HandleOrder([FromBody] OrderEvent order)
{
    // Process the order event
    return Ok();
}
```

---

## 8. Managed Identity & Secrets {#8-security}

```bash
# Enable system-assigned managed identity
az containerapp identity assign \
  --name api \
  --resource-group myRG \
  --system-assigned

# Add secrets
az containerapp secret set \
  --name api \
  --resource-group myRG \
  --secrets "db-conn=Server=tcp:mydb.database.windows.net;Database=mydb"

# Reference secret as environment variable
az containerapp update \
  --name api \
  --resource-group myRG \
  --set-env-vars "ConnectionStrings__DefaultConnection=secretref:db-conn"

# Reference Key Vault secret (with managed identity)
az containerapp secret set \
  --name api \
  --resource-group myRG \
  --secrets "api-key=keyvaultref:https://myvault.vault.azure.net/secrets/api-key,identityref:system"
```

---

## 9. CI/CD Deployment {#9-cicd}

### GitHub Actions

```yaml
name: Deploy to Container Apps
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Build and deploy
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: myRG
          containerAppName: api
          imageToDeploy: myacr.azurecr.io/api:${{ github.sha }}
```

---

## 10. Jobs (Background Tasks) {#10-jobs}

Container Apps Jobs run containers to completion (not long-running services).

```bash
# Manual trigger job
az containerapp job create \
  --name data-migration \
  --resource-group myRG \
  --environment myenv \
  --trigger-type Manual \
  --image myacr.azurecr.io/migrator:latest \
  --cpu 2.0 \
  --memory 4.0Gi \
  --replica-timeout 3600

# Scheduled job (cron)
az containerapp job create \
  --name nightly-report \
  --resource-group myRG \
  --environment myenv \
  --trigger-type Schedule \
  --cron-expression "0 2 * * *" \
  --image myacr.azurecr.io/reporter:latest \
  --cpu 0.5 \
  --memory 1.0Gi

# Event-driven job (scale with queue)
az containerapp job create \
  --name queue-processor \
  --resource-group myRG \
  --environment myenv \
  --trigger-type Event \
  --min-executions 0 \
  --max-executions 10 \
  --image myacr.azurecr.io/processor:latest \
  --scale-rule-name queue \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata queueName=jobs namespace=myservicebus messageCount=1
```

---

## 11. Observability {#11-observability}

```bash
# Container Apps automatically sends logs to Log Analytics
# Query logs with KQL

# Application console logs
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "api"
| where Log_s contains "error"
| project TimeGenerated, Log_s
| order by TimeGenerated desc

# System logs (scaling events, restarts)
ContainerAppSystemLogs_CL
| where ContainerAppName_s == "api"
| where Reason_s == "ScalingUp" or Reason_s == "ScalingDown"
| project TimeGenerated, Reason_s, ReplicaCount_d
```

---

## 12. Best Practices {#12-best-practices}

```
✅ When to Use Container Apps
├─ Microservices that need scale-to-zero
├─ Event-driven processors (queue consumers)
├─ APIs with variable traffic patterns
├─ Background jobs and scheduled tasks
└─ Teams that don't want to manage Kubernetes

❌ When NOT to Use Container Apps
├─ Need full K8s API (custom CRDs, operators) → use AKS
├─ Need Windows containers → use AKS or ACI
├─ Need stateful workloads with persistent volumes → use AKS
├─ Need GPU-intensive ML training → use AKS
└─ Already have K8s expertise and need full control → use AKS

✅ Architecture
├─ Use one Environment per application boundary
├─ Use internal ingress for service-to-service communication
├─ Use Dapr for cross-service calls (avoid hardcoded URLs)
├─ Use traffic splitting for canary deployments
└─ Use Container Apps Jobs for batch/scheduled work

✅ Cost Optimization
├─ Set min-replicas to 0 for non-critical services
├─ Right-size CPU and memory (start small, monitor usage)
├─ Use Consumption plan (default) — pay per second
└─ Use Dedicated plan only for compliance/isolation needs
```
