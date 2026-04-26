# Azure Services Deep Dive
## Document 2: Every Service Explained - Why, What, and How

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Complete explanation of each Azure service in our project

---

## TABLE OF CONTENTS

1. [Azure Kubernetes Service (AKS)](#aks)
2. [Azure Container Registry (ACR)](#acr)
3. [Azure Cosmos DB](#cosmos-db)
4. [Azure Key Vault](#key-vault)
5. [Azure Virtual Network](#vnet)
6. [Managed Identity](#managed-identity)
7. [Load Balancer](#load-balancer)
8. [Comparison Table](#comparison)

---

## Azure Kubernetes Service (AKS) {#aks}

### What is AKS?

**AKS** = Managed Kubernetes Service from Microsoft  
**Kubernetes** = Container orchestration platform that automates deployment, scaling, and management of containerized applications

Think of Kubernetes like an **intelligent warehouse manager**:
- You tell it what work to do (Docker container)
- It decides how many workers needed (replicas)
- It spreads work across multiple buildings (nodes)
- If a worker is sick, it replaces them (auto-healing)
- If orders spike, it hires more workers (auto-scaling)

### Why We Use AKS Instead of Running Servers Directly

**Without AKS (Manual approach):**
```
You manage:
- Buy 5 servers
- Install Docker
- Pull images manually
- Monitor each server
- Deploy updates manually
- Handle failures manually
- Scale by buying more hardware
Result: Time-consuming, error-prone, expensive
```

**With AKS (Managed approach):**
```
AKS manages:
- Infrastructure provisioning
- Docker runtime
- Container scheduling
- Auto-healing failed containers
- Auto-scaling based on load
- Rolling updates without downtime
You focus on: Application code only
Result: Faster, more reliable, cheaper
```

### How AKS Works in Our Project

```
┌──────────────────────────────────────────────────────┐
│         Azure Kubernetes Service (AKS)               │
│                                                      │
│  ┌────────────────────────────────────────────┐    │
│  │           Control Plane (Managed)           │    │
│  │  - API Server                              │    │
│  │  - Scheduler                               │    │
│  │  - etcd (database)                         │    │
│  │  Microsoft manages this completely         │    │
│  └────────────────────────────────────────────┘    │
│                      ↓                             │
│  ┌────────────────────────────────────────────┐    │
│  │        Worker Nodes (Linux VMs)            │    │
│  │                                            │    │
│  │  Node 1                    Node 2          │    │
│  │  ┌──────────────────┐  ┌──────────────────┐│    │
│  │  │ Kubelet          │  │ Kubelet          ││    │
│  │  │ Container Runtime│  │ Container Runtime││    │
│  │  │ (Docker/containerd)  │ (Docker/contain)││    │
│  │  │                  │  │                  ││    │
│  │  │  Pod 1: App 1   │  │  Pod 2: App 2   ││    │
│  │  │  ┌──────────┐   │  │  ┌──────────┐   ││    │
│  │  │  │Container │   │  │  │Container │   ││    │
│  │  │  └──────────┘   │  │  └──────────┘   ││    │
│  │  │                  │  │                  ││    │
│  │  └──────────────────┘  └──────────────────┘│    │
│  │                                            │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
│  Network:                                           │
│  - Service (LoadBalancer) → Routes traffic         │
│  - Network Policy → Controls communication         │
│  - Ingress → Advanced routing                      │
└──────────────────────────────────────────────────────┘
```

### Our AKS Configuration

**Cluster Name:** azurelearn-dev-aks  
**Kubernetes Version:** 1.34.4  
**Node Count:** 2 (can auto-scale)  
**Node Size:** Standard_D2s_v3 (2 CPU, 8 GB RAM per node)  
**Region:** westus

**What Gets Created?**

| Component | Purpose | Count | Details |
|-----------|---------|-------|---------|
| **Node Pool** | Group of identical VMs | 1 | 2 nodes, auto-scale enabled |
| **Nodes (VMs)** | Actual compute machines | 2 | Linux Ubuntu, Docker runtime installed |
| **System Namespace** | AKS system components | 1 | Contains DNS, metrics, core services |
| **Virtual Network** | Internal networking | 1 | 10.0.0.0/8 address space |
| **Subnet** | Node placement | 1 | 10.240.0.0/16 |
| **Network Security Group** | Firewall rules | 1 | Controls inbound/outbound traffic |

### Deployment in Our AKS

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-learn-app
spec:
  replicas: 2              # Start with 2 containers
  strategy:
    type: RollingUpdate    # Update one at a time (no downtime)
  template:
    spec:
      containers:
      - name: app
        image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
        ports:
        - containerPort: 8080
        
        # Health checks for Kubernetes
        livenessProbe:      # Is app still running?
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        
        readinessProbe:     # Is app ready for traffic?
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

### Auto-Scaling Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: azure-learn-app-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: azure-learn-app
  minReplicas: 2          # Always at least 2 pods
  maxReplicas: 5          # Never more than 5 pods
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**How It Works:**
- CPU usage below 70% → Scale down (minimum 2 pods)
- CPU usage above 70% → Add more pods (maximum 5)
- Each pod gets ~12.5% CPU threshold before scaling

### Service (Load Balancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: azure-learn-app-service
spec:
  type: LoadBalancer      # External IP assigned
  selector:
    app: azure-learn-app  # Route to pods with this label
  ports:
  - protocol: TCP
    port: 80              # External port (what users access)
    targetPort: 8080      # Pod port (app listens here)
```

**Result:**
- External IP: 20.245.92.146 (created automatically)
- User accesses: `http://20.245.92.146/api/products`
- Gets routed to: pod on port 8080
- Azure Load Balancer distributes across all pods

### Cost Implications

| Item | Cost | Notes |
|------|------|-------|
| **AKS Cluster Management** | FREE | Microsoft manages control plane |
| **Worker Nodes (D2s_v3)** | ~$100/node/month | Pay for the VMs, not AKS |
| **Data Transfer (egress)** | ~$0.10/GB | Traffic leaving Azure |
| **Load Balancer** | ~$15/month | Static IP charge |
| **Total (2 nodes)** | ~$250/month | Similar to small dedicated server |

---

## Azure Container Registry (ACR) {#acr}

### What is ACR?

**ACR** = Private Docker image repository  
Like a library of books (Docker images) that only you can access

### Why We Need ACR

**Without ACR:**
```
You: Docker image → Docker Hub (public)
     ↓
Anyone can: Download your image
           See your code
           Run your containers
Problem: Security risk! Your code is public!
```

**With ACR:**
```
You: Docker image → Azure Container Registry (private)
     ↓
Only authenticated users can: Download
                              Access
                              Manage
Security: Your code stays private!
```

### ACR in Our Project

**Registry Name:** azurelearnacrhof7rpcc  
**Tier:** Standard (affordable, good performance)  
**SKU:** 100 GB storage included  
**Region:** westus (same as AKS for better performance)

### Images in Our Registry

```
Registry: azurelearnacrhof7rpcc.azurecr.io

Repository: azurelearn
├─ Image:Tag latest        ← Points to latest build
├─ Image:Tag v1.0          ← Version 1.0
├─ Image:Tag v2.0          ← Version 2.0 (bug fix)
├─ Image:Tag v3.0          ← Version 3.0 (Swagger enabled)
└─ Image:Tag prod          ← Production version
```

### How AKS Pulls Images from ACR

```
1. Pod creation request in AKS
   ↓
2. AKS looks for image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
   ↓
3. Authentication:
   - Uses docker-registry secret (credentials)
   - OR uses Managed Identity (recommended)
   ↓
4. Image pull:
   - Downloads image from ACR
   - Caches locally on node
   ↓
5. Container starts from image
   ↓
6. Pod runs
```

### Docker Image Details

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app
COPY src/*.csproj ./
RUN dotnet restore           # Download dependencies
COPY src/ ./
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
USER appuser (non-root)
HEALTHCHECK --interval=30s ...
ENTRYPOINT ["dotnet", "AzureLearnApp.dll"]
```

**Image Layers:**
- Layer 1: Base image (aspnet:8.0, 127 MB)
- Layer 2: Application DLLs (~5 MB)
- Layer 3: Configuration (~1 MB)
- Total: ~133 MB

### Versioning Strategy

| Version | When Used | Example |
|---------|-----------|---------|
| `latest` | Most recent | CI/CD pipeline uses |
| `v1.0`, `v2.0`, etc | Release versions | Production tags |
| `dev` | Development builds | Testing branch |
| `staging` | Pre-production | Staging environment |

**Why Multiple Tags?**
- Easy rollback: Deploy v2.0 if v3.0 has issues
- A/B testing: Run two versions simultaneously
- Version tracking: Know what's in production

### ACR Pricing

| Tier | Cost | Storage | Features |
|------|------|---------|----------|
| Basic | $5/month | 10 GB | Good for learning |
| Standard | $30/month | 100 GB | Good for projects |
| Premium | $250/month | 500 GB | Enterprise features |

Our project: **Standard** (~$30/month)

---

## Azure Cosmos DB {#cosmos-db}

### What is Cosmos DB?

**Cosmos DB** = Fully managed NoSQL database  
Stores data without rigid table structures (unlike SQL databases)

### Why NoSQL Instead of SQL?

**SQL Database (Traditional):**
```
Table: Products
┌────┬──────────┬───────┐
│ ID │ Name     │ Price │
├────┼──────────┼───────┤
│ 1  │ Product1 │ 99.99 │
│ 2  │ Product2 │ 49.99 │
└────┴──────────┴───────┘

Fixed structure - must follow schema
```

**NoSQL Database (Cosmos DB):**
```json
Collection: Products
Document 1:
{
  "id": "1",
  "name": "Product1",
  "price": 99.99,
  "category": "Electronics",
  "inStock": true
}

Document 2:
{
  "id": "2",
  "name": "Product2",
  "price": 49.99,
  "tags": ["home", "garden"],
  "supplier": {"name": "Acme"}
}

Flexible structure - each document can vary
```

### Why Cosmos DB for Our Project?

| Reason | Benefit |
|--------|---------|
| **Global Distribution** | Low latency worldwide |
| **Serverless** | Pay only for requests, not idle servers |
| **Automatic Scaling** | Handles traffic spikes |
| **High Availability** | 99.99% SLA built-in |
| **Multi-region** | Replicate data automatically |
| **Flexible Schema** | Easy to evolve data model |

### Our Cosmos DB Setup

**Account Name:** azurelearndbhof7rpcc  
**Kind:** GlobalDocumentDB  
**API:** SQL (like SQL queries but on JSON documents)  
**Consistency:** Session (balanced approach)  
**Replication:** Single region (westus) - can change later  
**Backup:** Automatic every 4 hours

```
Cosmos DB Account: azurelearndbhof7rpcc
│
└─ Database: AzureLearnDb
   │
   └─ Container: Products
      │
      ├─ Partition Key: /id
      │  (Data distributed by this field)
      │
      └─ Documents (JSON)
         ├─ {"id": "1", "name": "..."}
         ├─ {"id": "2", "name": "..."}
         └─ {"id": "3", "name": "..."}
```

### Pricing Model

**Serverless (What We Use):**
- No upfront cost
- Pay per request
- Free tier: 1M free requests/month
- Cost: ~$0.25 per million requests

**Provisioned (Alternative for high volume):**
- Reserved throughput: 400 RU/s minimum
- Cost: ~$25/month + usage
- Better for predictable workloads

**When to Use Each:**
- **Serverless:** Dev/test, variable traffic, learning (like us)
- **Provisioned:** Production, consistent load, cost predictable

### Data Model

```json
Product Document Structure:
{
  "id": "unique-identifier",
  "name": "Product Name",
  "description": "What is this?",
  "price": 99.99,
  "stock": 50,
  "category": "Electronics",
  "createdAt": "2026-04-26T00:00:00Z",
  "updatedAt": "2026-04-26T00:00:00Z",
  "tags": ["new", "featured"],
  "supplier": {
    "name": "Acme Corp",
    "country": "USA"
  }
}
```

### Queries in Cosmos DB

```sql
-- Get all products
SELECT * FROM Products c

-- Find by category
SELECT * FROM Products c WHERE c.category = 'Electronics'

-- Price range
SELECT * FROM Products c WHERE c.price BETWEEN 10 AND 100

-- With ordering
SELECT * FROM Products c ORDER BY c.price DESC

-- Aggregation
SELECT COUNT(1) as count FROM Products c
```

### Connection from Our App

```csharp
// Program.cs
var cosmosClient = new CosmosClient(
    cosmosConnectionString,
    new CosmosClientOptions
    {
        ConnectionMode = ConnectionMode.Gateway,
        MaxRetryAttemptsOnRateLimitedRequests = 9,
        MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30)
    }
);

// Usage
var database = cosmosClient.GetDatabase("AzureLearnDb");
var container = database.GetContainer("Products");

// Create
await container.CreateItemAsync(product);

// Read
var item = await container.ReadItemAsync<Product>("id-value");

// Query
var query = container.GetItemQueryIterator<Product>("SELECT * FROM c");

// Update
await container.ReplaceItemAsync(product, "id-value");

// Delete
await container.DeleteItemAsync<Product>("id-value");
```

---

## Azure Key Vault {#key-vault}

### What is Key Vault?

**Key Vault** = Secure storage for secrets, keys, and certificates  
Like a safe that only authorized apps can open

### What Secrets Are We Storing?

```
Key Vault: azurelearnkvhof7rpcc

├─ Secret: CosmosDbConnectionString
│  Value: "AccountEndpoint=https://...;AccountKey=..."
│  Why: Don't hardcode in code!
│  Who needs: App running in AKS
│
├─ Secret: ApiKeys
│  Value: "sk_live_xxxxx"
│  Why: Third-party API access
│  Who needs: App
│
└─ Certificate: SSL/TLS (future)
   Why: HTTPS encryption
```

### Why We Need Key Vault

**❌ WRONG (Don't do this):**
```csharp
// In Program.cs or appsettings.json
string connectionString = 
    "AccountEndpoint=https://...;AccountKey=sk_live_xxxxx;";
// Problem: Secret in code → Git repo → Anyone with access sees it!
```

**✅ CORRECT (What we do):**
```csharp
// In Azure Key Vault
// Retrieved at runtime via Managed Identity
var secretClient = new SecretClient(new Uri(keyVaultUrl), credential);
var secret = await secretClient.GetSecretAsync("CosmosDbConnectionString");
string connectionString = secret.Value.Value;
// Secret never in code, only retrieved when needed
```

### Key Vault Access Control

```
┌──────────────────────────────────────────────┐
│       Azure Key Vault: azurelearnkvhof7rpcc  │
│                                              │
│  Access Policy 1:                            │
│  ├─ Principal: Your User Account             │
│  ├─ Permissions: Get, List, Set, Delete      │
│  └─ Purpose: Manage secrets                  │
│                                              │
│  Access Policy 2:                            │
│  ├─ Principal: Managed Identity              │
│  │  (azure-learn-app-pod-identity)           │
│  ├─ Permissions: Get (only read)             │
│  └─ Purpose: Pods read secrets at runtime    │
│                                              │
│  Network Security:                           │
│  ├─ Firewall: Deny by default                │
│  ├─ Whitelist: Azure services                │
│  └─ Private Link: Optional VPN access        │
└──────────────────────────────────────────────┘
```

### Managed Identity for Access

**Problem:** How does app in AKS authenticate to Key Vault?  
**Solution:** Managed Identity (handled by Azure automatically)

```
App (AKS Pod)
    ↓
"I need secret CosmosDbConnectionString"
    ↓
Managed Identity
    ↓
"I am azure-learn-app-pod-identity"
"Client ID: bf9ea53d-6351-42bf-aa45-3328a0bd296f"
    ↓
Azure AD (validates identity)
    ↓
"Yes, you're allowed"
    ↓
Key Vault
    ↓
Returns: CosmosDbConnectionString
    ↓
App has secret!
```

**Benefits:**
- No passwords to manage
- No keys to hardcode
- Automatic token refresh
- Audit trail in Azure

### Key Vault Pricing

| Feature | Cost |
|---------|------|
| Vault creation | FREE |
| Operations (Get, List, Set) | $0.03 per 10,000 ops |
| Certificate operations | $1 per operation |
| Total for typical app | ~$5-10/month |

---

## Azure Virtual Network {#vnet}

### What is Virtual Network?

**VNet** = Private network within Azure  
Like your home WiFi - only your devices can talk to each other

### Network Topology

```
                 Internet
                    ↓
            Azure Load Balancer
                    ↓
        ┌──────────────────────┐
        │   Public Subnet      │
        │  (Internet-facing)   │
        │  10.0.0.0/24        │
        │                      │
        │  Load Balancer IP    │
        │  20.245.92.146       │
        └──────────────────────┘
                    ↓
        ┌──────────────────────┐
        │  Virtual Network     │
        │  10.0.0.0/8         │
        │                      │
        │  AKS Subnet:         │
        │  10.240.0.0/16      │
        │  - Nodes: 2         │
        │  - Pods: Running    │
        │                      │
        └──────────────────────┘
                    ↓
        ┌──────────────────────┐
        │  Internal Services   │
        │  (Not accessible)    │
        │  - Cosmos DB        │
        │  - Key Vault        │
        │  - Storage          │
        └──────────────────────┘
```

### Our VNet Configuration

**Network:** azurelearn-dev-vnet  
**Address Space:** 10.0.0.0/8 (16 million IPs available)  
**AKS Subnet:** 10.240.0.0/16 (65k IPs for pods/nodes)

### Network Security Group (NSG)

Acts like a firewall with rules:

```
Inbound Rules:
├─ SSH (22): Deny (pods don't use SSH)
├─ HTTP (80): Allow (Load Balancer)
├─ HTTPS (443): Allow (future)
├─ Kubernetes API (6443): Deny (internal only)
└─ Other: Deny

Outbound Rules:
├─ All: Allow (pods need to download images, reach DB)
```

### Service-to-Service Communication

```
┌─────────────────────────────────────────┐
│          Virtual Network                │
│          10.0.0.0/8                     │
│                                         │
│  ┌──────────────────┐                  │
│  │   AKS Pod        │                  │
│  │  IP: 10.224.0.19 │                  │
│  │                  │                  │
│  │  var client =    │                  │
│  │    new Cosmos    │                  │
│  │    Client(uri)   │                  │
│  └────────┬─────────┘                  │
│           ↓                            │
│  ┌──────────────────────────────┐     │
│  │   Cosmos DB (Private)        │     │
│  │   10.x.x.x (internal IP)     │     │
│  │   Accessible only inside VNet│     │
│  └──────────────────────────────┘     │
│                                         │
│  ┌──────────────────────────────┐     │
│  │   Key Vault (Private)        │     │
│  │   10.x.x.x (internal IP)     │     │
│  │   Accessible only inside VNet│     │
│  └──────────────────────────────┘     │
└─────────────────────────────────────────┘
```

**Key Point:** Cosmos DB and Key Vault accessible only within VNet = Enhanced security!

---

## Managed Identity {#managed-identity}

### What is Managed Identity?

**Managed Identity** = Azure-managed service account  
Like a virtual "user" that Azure creates and manages for you

### Types of Managed Identity

```
┌────────────────────────────────────────────────────┐
│            Managed Identity Types                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  System-Assigned:                                  │
│  ├─ One per resource                              │
│  ├─ Lifecycle tied to resource                    │
│  ├─ Automatically deleted when resource deleted  │
│  └─ Use when: One app needs access                │
│                                                    │
│  User-Assigned (What we use):                     │
│  ├─ Created separately                            │
│  ├─ Can assign to multiple resources              │
│  ├─ Lifecycle independent                        │
│  └─ Use when: Multiple apps need same access      │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Our Managed Identity

**Name:** azure-learn-app-pod-identity  
**Type:** User-Assigned  
**Client ID:** bf9ea53d-6351-42bf-aa45-3328a0bd296f  
**Principal ID:** 4741bbdd-46a2-40c2-95de-a4f29c3cc305

### Permission Flow

```
Step 1: Managed Identity Created
   ├─ Client ID assigned
   ├─ Principal ID assigned
   └─ Registered with Azure AD

Step 2: Grant Permissions
   ├─ Key Vault: Grant "Secrets User" role
   ├─ Cosmos DB: Grant "Data Contributor" role
   └─ ACR: Grant "AcrPull" role

Step 3: Pod Lifecycle
   ├─ Pod starts in AKS
   ├─ Pod has annotation: azure.workload.identity/use: true
   └─ Workload Identity enabled

Step 4: Pod Requests Secret
   ├─ Pod: "I need CosmosDbConnectionString"
   ├─ Workload Identity Provider: "Are you azure-learn-app?"
   ├─ Pod: "Yes, my service account says so"
   ├─ Azure AD: "Verified! Token issued"
   ├─ Key Vault: "Token valid for azure-learn-app, secret returned"
   └─ Pod: "Secret received!"

Step 5: Pod Uses Secret
   └─ Connect to Cosmos DB
```

### Why Managed Identity is Better Than Passwords

| Aspect | Passwords | Managed Identity |
|--------|-----------|------------------|
| **Storage** | Hardcoded in config | Azure manages |
| **Rotation** | Manual process | Automatic |
| **Expiration** | Can expire | Tokens auto-refresh |
| **Audit** | Limited logging | Full Azure AD audit |
| **Risk** | High - exposed in code | Low - never exposed |

---

## Load Balancer {#load-balancer}

### What is Load Balancer?

**Load Balancer** = Distributes incoming traffic across multiple pods

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│         External Users (Internet)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  User 1     │  │  User 2     │  │  User 3     │    │
│  └────────┬────┘  └────────┬────┘  └────────┬────┘    │
│           │                 │                 │         │
│           └─────────────────┼─────────────────┘        │
│                             │                          │
│                     ┌───────▼────────┐               │
│                     │  Azure Load    │               │
│                     │  Balancer      │               │
│                     │  External IP:  │               │
│                     │  20.245.92.146 │               │
│                     │  Port: 80      │               │
│                     └───────┬────────┘               │
│                             │                         │
│           ┌─────────────────┼─────────────────┐      │
│           ↓                 ↓                 ↓      │
│      ┌──────────┐   ┌──────────┐      ┌──────────┐  │
│      │  Pod 1   │   │  Pod 2   │      │ Pod N    │  │
│      │ Port 8080│   │ Port 8080│      │Port 8080 │  │
│      └──────────┘   └──────────┘      └──────────┘  │
│                                                       │
│  Kubernetes Service (Type: LoadBalancer)             │
│  - Selects pods with label: app=azure-learn-app     │
│  - Distributes traffic: Round-robin by default       │
│  - Session affinity: Optional (sticky sessions)      │
└─────────────────────────────────────────────────────────┘
```

### Load Balancing Algorithms

**Round-Robin (Default):**
```
Request 1 → Pod 1
Request 2 → Pod 2
Request 3 → Pod 1
Request 4 → Pod 2
(Cycles through pods equally)
```

**Least Connections:**
```
Pod 1: 5 connections
Pod 2: 2 connections
Pod 3: 8 connections

Next request → Pod 2 (has least)
```

### Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: azure-learn-app-service
spec:
  type: LoadBalancer              # Assigns external IP
  sessionAffinity: None           # No sticky sessions
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800       # 3 hours
  selector:
    app: azure-learn-app          # Route to these pods
  ports:
  - protocol: TCP
    port: 80                       # External port
    targetPort: 8080               # Pod port
    nodePort: 30131                # Alternative access
```

### Accessing Service

```
Method 1: External IP (LoadBalancer)
  curl http://20.245.92.146/api/products
  (Works from anywhere on internet)

Method 2: ClusterIP (Internal only)
  curl http://10.0.160.208/api/products
  (Only from within K8s cluster)

Method 3: NodePort (Node IP + Port)
  curl http://[node-ip]:30131/api/products
  (Any node in cluster)
```

### Cost

| Item | Monthly Cost |
|------|--------------|
| Standard Load Balancer | ~$15-20 |
| Data processed | ~$0.006 per GB |
| Typical app | ~$20-30 |

---

## Services Comparison {#comparison}

### Decision Matrix: Which Service When?

```
┌─────────────────────────────────────────────────────────┐
│     SERVICE SELECTION GUIDE                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ For Containerized Applications:                        │
│ ├─ Few containers (< 10 for learning): AKS             │
│ ├─ Serverless containers: Azure Container Instances   │
│ └─ Full control: Virtual Machines                     │
│                                                         │
│ For Data Storage:                                      │
│ ├─ NoSQL, flexible schema: Cosmos DB                  │
│ ├─ Relational data: Azure SQL Database                │
│ ├─ Key-value cache: Azure Cache for Redis             │
│ └─ Blobs (files): Azure Storage                       │
│                                                         │
│ For Security:                                          │
│ ├─ Secrets/Keys: Key Vault                            │
│ ├─ Network isolation: Virtual Networks                │
│ ├─ Identity management: Managed Identities             │
│ └─ Advanced: Azure AD Premium                         │
│                                                         │
│ For DevOps:                                            │
│ ├─ Image registry: Container Registry (ACR)           │
│ ├─ CI/CD pipelines: Azure DevOps, GitHub Actions     │
│ └─ Infrastructure as Code: Bicep, Terraform           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Cost Comparison

| Service | Monthly Cost (Our Setup) | Scaling Unit |
|---------|----------------------------|--------------|
| **AKS** | ~$250 (2 nodes) | Per node |
| **ACR** | ~$30 | Per registry |
| **Cosmos DB** | ~$10 (serverless) | Per request |
| **Key Vault** | ~$5 | Per operation |
| **VNet** | FREE | Per network |
| **Managed Identity** | FREE | Per identity |
| **Load Balancer** | ~$20 | Per service |
| **Total Monthly** | ~$315 | All included |

**Scale by 10x users?**
- AKS auto-scales: +$250/node
- Cosmos DB: +$x (proportional to requests)
- Others: Mostly fixed costs

---

## Key Takeaways

✅ **AKS** = Container orchestration + auto-scaling + high availability  
✅ **ACR** = Private, secure Docker image storage  
✅ **Cosmos DB** = Flexible NoSQL database, automatic scaling  
✅ **Key Vault** = Secure secrets management, zero-knowledge approach  
✅ **VNet** = Network isolation, private communication  
✅ **Managed Identity** = Passwordless authentication, automatic token refresh  
✅ **Load Balancer** = Distribute traffic, automatic failover  

**Next Document:** Document 3 will cover application code architecture and best practices.
