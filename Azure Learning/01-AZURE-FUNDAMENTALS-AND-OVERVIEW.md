# Azure Fundamentals and Complete Architecture Guide
## Document 1: Azure Concepts, Services, and Project Overview

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Audience:** Complete beginners to advanced users  
**Time to Read:** 45-60 minutes

---

## TABLE OF CONTENTS

1. [What is Azure and Why Use It?](#what-is-azure)
2. [Cloud Computing Models](#cloud-computing-models)
3. [Azure Core Concepts](#azure-core-concepts)
4. [Our Project Architecture Overview](#project-architecture)
5. [Visual Architecture Diagrams](#visual-diagrams)
6. [Resource Hierarchy](#resource-hierarchy)

---

## What is Azure and Why Use It? {#what-is-azure}

### Definition
**Microsoft Azure** is a cloud computing platform that provides on-demand computing resources, storage, networking, and databases. Instead of buying and maintaining physical servers, you rent computing power from Microsoft's data centers globally.

### Why Use Azure?

| Benefit | Explanation |
|---------|-------------|
| **Cost Efficiency** | Pay only for what you use. No expensive hardware to maintain. Scales up/down automatically. |
| **Global Presence** | 60+ data centers worldwide. Deploy apps close to users for better performance. |
| **Security** | Enterprise-grade security, compliance certifications (HIPAA, PCI-DSS, ISO 27001), encrypted by default. |
| **Scalability** | Automatically handle traffic spikes. Grow from 10 to 1 million users without infrastructure changes. |
| **Reliability** | 99.99% uptime SLA. Multi-region redundancy. Automatic failover. |
| **Integration** | Works seamlessly with Microsoft products (Office, Teams, Windows). Third-party tool integrations. |
| **Innovation** | Access to AI, ML, IoT, blockchain services without building from scratch. |

### Real-World Example: Our Project
**Without Azure (On-Premises):**
- Buy 5 expensive servers (~$50,000)
- Hire IT team to maintain them (~$200,000/year)
- Physical space with cooling, electricity (~$10,000/year)
- Hard to scale during traffic spikes
- Disaster recovery is expensive and complex

**With Azure:**
- Pay ~$100-500/month based on usage
- Microsoft manages maintenance, updates, security
- Automatic scaling during traffic peaks
- Built-in disaster recovery
- Scale globally in minutes

---

## Cloud Computing Models {#cloud-computing-models}

### Three Main Models

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR RESPONSIBILITY                     │
├─────────────────────────────────────────────────────────────┤
│ On-Premises: [Apps][Data][Runtime][OS][Server][Network]    │
│ IaaS:        [Apps][Data][Runtime][OS] ← Azure manages      │
│ PaaS:        [Apps][Data]              ← Azure manages      │
│ SaaS:        [   ]                      ← Azure manages      │
└─────────────────────────────────────────────────────────────┘
```

### 1. **Infrastructure as a Service (IaaS)**
You manage: Applications, Data, Runtime, OS  
Azure manages: Servers, Networking, Storage

**Examples:** Azure VMs, Azure Container Instances  
**Use Case:** Full control needed, legacy applications, complex networking

### 2. **Platform as a Service (PaaS)**
You manage: Applications, Data  
Azure manages: Runtime, OS, Servers, Networking

**Examples:** Azure App Service, Azure SQL Database, Azure Kubernetes Service  
**Use Case:** Focus on code, not infrastructure  
**Our Project Uses:** PaaS (Cosmos DB, App Service, AKS)

### 3. **Software as a Service (SaaS)**
You manage: Only your data  
Azure manages: Everything else

**Examples:** Microsoft 365, Dynamics 365, Salesforce  
**Use Case:** Use ready-made applications

---

## Azure Core Concepts {#azure-core-concepts}

### 1. **Subscription**
- Entry point to Azure
- Billing unit (monthly bill per subscription)
- Controls what resources can be created
- Our project: Subscription ID = `c40e63f8-3058-4a39-81a9-d3e58648a4b4`

### 2. **Resource Group**
- Logical container for related resources
- All resources must belong to a resource group
- Shared billing, access control, deployment together
- Our project: Resource Group = `azure-learn-rg-dev`

**Why Separate Resource Groups?**
```
azure-learn-rg-dev    (Development)
  ├── AKS Cluster
  ├── Cosmos DB
  ├── Key Vault
  └── Container Registry

azure-learn-rg-prod   (Production)
  ├── AKS Cluster
  ├── Cosmos DB
  ├── Key Vault
  └── Container Registry
```

### 3. **Region/Location**
- Azure data center location
- Physical place where resources live
- Different regions = different latencies, pricing, services
- Our project: Location = `westus` (West US)

**Why Choose a Region?**
- **Latency:** Deploy close to users
- **Compliance:** Some countries require data to stay locally
- **Cost:** Different regions have different pricing
- **Service Availability:** Not all services available in all regions

### 4. **Availability Zone**
- Separate data centers within a region
- Protects against data center failures
- High availability for critical apps
- Our project: Not explicitly set (simplified for learning)

---

## Our Project Architecture {#project-architecture}

### What We're Building

**Azure Learning App** - A product management REST API deployed on Kubernetes with:
- ✅ Scalable API backend (.NET 8)
- ✅ NoSQL database (Cosmos DB)
- ✅ Secrets management (Key Vault)
- ✅ Container orchestration (AKS)
- ✅ CI/CD pipeline (Azure DevOps)
- ✅ Private image registry (ACR)

### High-Level Data Flow

```
End User (Browser/API Client)
        ↓
    Load Balancer
    (External IP: 20.245.92.146)
        ↓
┌─────────────────────────────────┐
│   Azure Kubernetes Service      │
│   (AKS Cluster)                 │
│  ┌─────────────────────────┐   │
│  │ Pod 1: App Container    │   │
│  │ - Port: 8080            │   │
│  │ - Replicas: 2-5 (auto)  │   │
│  └─────────────────────────┘   │
│  ┌─────────────────────────┐   │
│  │ Pod 2: App Container    │   │
│  │ - Port: 8080            │   │
│  │ - Replicas: 2-5 (auto)  │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘
        ↓
    Managed Identity
    (Authentication)
        ↓
┌─────────────────────────────────┐
│   Azure Cosmos DB               │
│   (NoSQL Database)              │
│   - Collections: Products       │
│   - Throughput: Serverless      │
└─────────────────────────────────┘

Side Services:
- Key Vault (Stores connection strings)
- Container Registry (Stores Docker images)
- Azure DevOps (CI/CD automation)
```

---

## Visual Architecture Diagrams {#visual-diagrams}

### Complete Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER                                       │
│                  (Local Machine / VS Code)                             │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 1. Write Code (.NET C#)                                          │ │
│  │ 2. Commit to Git                                                 │ │
│  │ 3. Push to Azure DevOps                                          │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│                    AZURE DEVOPS (CI/CD)                               │
│  Pipeline Stages:                                                      │
│  1. Build      → Compile code, run tests, build Docker image          │
│  2. Validate   → Check infrastructure templates                       │
│  3. Deploy-Dev → Deploy to development                                │
│  4. Deploy-Prod→ Deploy to production (with approval)                 │
└───────────────────────┬────────────────────────┬─────────────────────┘
                        │                        │
          ┌─────────────┘                        └──────────────┐
          ↓                                                     ↓
┌──────────────────────────────────┐        ┌──────────────────────────┐
│   DEVELOPMENT ENVIRONMENT        │        │  PRODUCTION ENVIRONMENT  │
│                                  │        │                          │
│ Resource Group: rg-dev           │        │ Resource Group: rg-prod  │
│                                  │        │                          │
│ ┌────────────────────────────┐  │        │ ┌────────────────────┐   │
│ │  Azure Container Registry  │  │        │ │  Azure Container   │   │
│ │  (Private Docker images)   │  │        │ │  Registry (Images) │   │
│ │  - azurelearn:v3.0         │  │        │ │  - azurelearn:v3.0 │   │
│ │  - azurelearn:latest       │  │        │ │  - azurelearn:prod │   │
│ └────────────────────────────┘  │        │ └────────────────────┘   │
│              ↓                   │        │           ↓              │
│ ┌────────────────────────────┐  │        │ ┌────────────────────┐   │
│ │    AKS Cluster             │  │        │ │   AKS Cluster      │   │
│ │  (Kubernetes)              │  │        │ │  (Kubernetes)      │   │
│ │                            │  │        │ │                    │   │
│ │  Nodes: 2 (auto-scale)     │  │        │ │  Nodes: 3+ (prod)  │   │
│ │  ┌──────────┐ ┌──────────┐ │  │        │ │  ┌──────────────┐  │   │
│ │  │ Pod 1    │ │ Pod 2    │ │  │        │ │  │ Pod 1        │  │   │
│ │  │ App v3.0 │ │ App v3.0 │ │  │        │ │  │ App (latest) │  │   │
│ │  └──────────┘ └──────────┘ │  │        │ │  └──────────────┘  │   │
│ │                            │  │        │ │                    │   │
│ │  Health Checks: ✓ Live     │  │        │ │  Health Checks: ✓  │   │
│ │                  ✓ Ready   │  │        │ │                    │   │
│ └────────────────────────────┘  │        │ └────────────────────┘   │
│              ↓                   │        │           ↓              │
│ ┌────────────────────────────┐  │        │ ┌────────────────────┐   │
│ │    Load Balancer           │  │        │ │  Load Balancer     │   │
│ │  IP: 20.245.92.146         │  │        │ │  IP: [prod-ip]     │   │
│ │  Port: 80 → 8080           │  │        │ │  Port: 80 → 8080   │   │
│ └────────────────────────────┘  │        │ └────────────────────┘   │
│              ↑                   │        │           ↑              │
│         External API            │        │    Prod External API    │
│                                  │        │                          │
│ ┌────────────────────────────┐  │        │ ┌────────────────────┐   │
│ │  Cosmos DB (Database)      │  │        │ │  Cosmos DB (Data)  │   │
│ │  Collection: Products      │  │        │ │  Collection: Prod  │   │
│ │  Serverless pricing        │  │        │ │  Provisioned       │   │
│ └────────────────────────────┘  │        │ └────────────────────┘   │
│                                  │        │                          │
│ ┌────────────────────────────┐  │        │ ┌────────────────────┐   │
│ │  Key Vault (Secrets)       │  │        │ │  Key Vault         │   │
│ │  - Cosmos Connection       │  │        │ │  - Secrets         │   │
│ │  - API Keys                │  │        │ │                    │   │
│ └────────────────────────────┘  │        │ └────────────────────┘   │
│                                  │        │                          │
└──────────────────────────────────┘        └──────────────────────────┘
```

### Resource Dependencies

```
Subscription (Billing Unit)
│
└── Resource Group: azure-learn-rg-dev
    │
    ├── Azure Container Registry (ACR)
    │   └─ Stores Docker images
    │      └─ Used by: AKS nodes
    │
    ├── Azure Kubernetes Service (AKS)
    │   ├─ 2 nodes
    │   ├─ Pod 1 (App)
    │   ├─ Pod 2 (App)
    │   ├─ Service (LoadBalancer)
    │   ├─ ConfigMap (Config)
    │   └─ Secrets (Credentials)
    │      └─ Authenticated by: Managed Identity
    │
    ├── Cosmos DB Account
    │   ├─ Database: AzureLearnDb
    │   └─ Container: Products
    │      ↑─ Accessed by: AKS Pods
    │
    ├── Key Vault
    │   ├─ Secret: CosmosDbConnectionString
    │   ├─ Secret: ApiKeys
    │   └─ Policy: Allows Managed Identity access
    │      ↑─ Accessed by: AKS Pods via Managed Identity
    │
    ├── Virtual Network
    │   ├─ Subnet for AKS nodes
    │   └─ Network Policy (optional)
    │
    └── Managed Identity
        └─ Authenticates pods to: Key Vault, Cosmos DB
```

---

## Resource Hierarchy {#resource-hierarchy}

### Conceptual Model

```
HIERARCHY LEVELS:

Level 1: Subscription (Global)
  ↓
Level 2: Resource Group (Logical Container)
  ↓
Level 3: Resources (Actual Services)
  ├─ Compute Resources (AKS, VMs)
  ├─ Storage Resources (Cosmos DB, Storage Accounts)
  ├─ Network Resources (Virtual Network, Load Balancer)
  ├─ Security Resources (Key Vault, Managed Identity)
  └─ Integration Resources (Container Registry)
  ↓
Level 4: Resource Details (Configuration)
  ├─ Node Pools (in AKS)
  ├─ Collections (in Cosmos DB)
  ├─ Subnets (in Virtual Network)
  └─ Vaults (in Key Vault)
```

### Our Project's Hierarchy

```
Subscription: c40e63f8-3058-4a39-81a9-d3e58648a4b4
│
├── Resource Group: azure-learn-rg-dev (Region: westus)
│   ├── Container Registry: azurelearnacrhof7rpcc
│   │   └─ Repository: azurelearn
│   │      ├─ Tag: latest
│   │      ├─ Tag: v1.0
│   │      ├─ Tag: v2.0
│   │      └─ Tag: v3.0
│   │
│   ├── AKS Cluster: azurelearn-dev-aks
│   │   ├─ Node Pool: system (2 nodes)
│   │   │  ├─ VM Type: Standard_D2s_v3
│   │   │  ├─ OS: Linux (Ubuntu)
│   │   │  └─ K8s Version: 1.34.4
│   │   │
│   │   └─ Namespace: azure-learn-app
│   │      ├─ Deployment: azure-learn-app
│   │      │  ├─ Replicas: 2 (min: 2, max: 5)
│   │      │  ├─ Strategy: RollingUpdate
│   │      │  └─ Pod Template
│   │      │     └─ Container: app
│   │      │        ├─ Image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
│   │      │        ├─ Port: 8080
│   │      │        └─ Health Probes: Liveness, Readiness
│   │      │
│   │      ├─ Service: azure-learn-app-service
│   │      │  ├─ Type: LoadBalancer
│   │      │  ├─ External IP: 20.245.92.146
│   │      │  └─ Port Mapping: 80:30131
│   │      │
│   │      ├─ HorizontalPodAutoscaler: azure-learn-app-hpa
│   │      │  ├─ Min Replicas: 2
│   │      │  ├─ Max Replicas: 5
│   │      │  └─ Target CPU: 70%
│   │      │
│   │      ├─ ConfigMap: app-config
│   │      │  ├─ KeyVault__Url: https://azurelearnkvhof7rpcc.vault.azure.net/
│   │      │  ├─ CosmosDb__DatabaseName: AzureLearnDb
│   │      │  ├─ CosmosDb__ContainerName: Products
│   │      │  └─ ASPNETCORE_ENVIRONMENT: Production
│   │      │
│   │      ├─ Secret: cosmos-db-secret
│   │      │  └─ connectionString: [encrypted]
│   │      │
│   │      ├─ Secret: acr-secret
│   │      │  ├─ username: [encrypted]
│   │      │  ├─ password: [encrypted]
│   │      │  └─ server: azurelearnacrhof7rpcc.azurecr.io
│   │      │
│   │      └─ ServiceAccount: azure-learn-app-sa
│   │         └─ Workload Identity: [client-id]
│   │
│   ├── Cosmos DB Account: azurelearndbhof7rpcc
│   │   ├─ Kind: GlobalDocumentDB
│   │   ├─ API: SQL (Core)
│   │   ├─ Consistency: Session
│   │   ├─ Replication: Single region (westus)
│   │   ├─ Backup: Periodic (every 4 hours)
│   │   └─ Database: AzureLearnDb
│   │      └─ Container: Products
│   │         ├─ Partition Key: /id
│   │         ├─ Throughput: Serverless
│   │         └─ TTL: -1 (no expiration)
│   │
│   ├── Key Vault: azurelearnkvhof7rpcc
│   │   ├─ SKU: Standard
│   │   ├─ Purge Protection: Enabled
│   │   ├─ Access Policies
│   │   │  ├─ User: [your-account]
│   │   │  │  └─ Permissions: Get, List, Set, Delete
│   │   │  └─ Managed Identity: azure-learn-app-pod-identity
│   │   │     └─ Permissions: Get
│   │   │
│   │   └─ Secrets
│   │      ├─ CosmosDbConnectionString: [encrypted]
│   │      └─ [other secrets]
│   │
│   ├── Virtual Network: azurelearn-dev-vnet
│   │   ├─ Address Space: 10.0.0.0/8
│   │   ├─ Subnet: aks-subnet
│   │   │  └─ CIDR: 10.240.0.0/16
│   │   └─ Network Policy: Enabled
│   │
│   ├── Managed Identity: azure-learn-app-pod-identity
│   │   ├─ Type: User-Assigned
│   │   ├─ Client ID: bf9ea53d-6351-42bf-aa45-3328a0bd296f
│   │   └─ Principal ID: 4741bbdd-46a2-40c2-95de-a4f29c3cc305
│   │
│   └── Role Assignments
│       ├─ Cosmos DB Data Contributor (Managed ID → Cosmos DB)
│       ├─ Key Vault Secrets User (Managed ID → Key Vault)
│       └─ AcrPull (AKS nodes → Container Registry)
│
└── Resource Group: azure-learn-rg-prod (Similar structure for production)
```

---

## Key Takeaways

✅ Azure is a cloud provider offering IaaS, PaaS, and SaaS  
✅ Resources organized in subscriptions and resource groups  
✅ Our project uses PaaS services (AKS, Cosmos DB, Key Vault)  
✅ Multi-region deployment for high availability  
✅ Managed identities provide secure authentication  
✅ Separation of concerns: compute, storage, security, networking  

**Next Document:** Document 2 will explain each Azure service in detail and why we created them.
