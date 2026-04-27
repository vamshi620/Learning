# 20 – Azure Certifications Guide

---

## Table of Contents
1. [Certification Roadmap](#1-certification-roadmap)
2. [AZ-900: Azure Fundamentals](#2-az-900-azure-fundamentals)
3. [AZ-104: Azure Administrator](#3-az-104-azure-administrator)
4. [AZ-204: Azure Developer](#4-az-204-azure-developer)
5. [AZ-305: Azure Solutions Architect](#5-az-305-azure-solutions-architect)
6. [AZ-400: Azure DevOps Engineer](#6-az-400-azure-devops-engineer)
7. [AZ-500: Azure Security Engineer](#7-az-500-azure-security-engineer)
8. [Specialty Certifications](#8-specialty-certifications)
9. [Exam Tips & Strategies](#9-exam-tips--strategies)
10. [Study Resources](#10-study-resources)

---

## 1. Certification Roadmap

```
FUNDAMENTALS (no prereqs)
└── AZ-900: Azure Fundamentals              ─── start here

ASSOCIATE (recommended after AZ-900)
├── AZ-104: Azure Administrator
├── AZ-204: Azure Developer
├── AZ-400: Azure DevOps Engineer
├── AZ-500: Azure Security Engineer
├── AZ-700: Networking Engineer
├── AZ-800/801: Hybrid Administrator
└── DP-203: Data Engineer

EXPERT (requires Associate)
├── AZ-305: Solutions Architect Expert
│   └── prereq: AZ-104
└── AZ-400: DevOps Engineer Expert
    └── prereq: AZ-104 or AZ-204

SPECIALTY
├── AZ-140: Azure Virtual Desktop
├── AI-102: Azure AI Engineer
├── DP-100: Azure Data Scientist
└── DP-300: Azure Database Administrator
```

---

## 2. AZ-900: Azure Fundamentals

**Audience:** Beginners, non-technical stakeholders, IT professionals new to Azure  
**Duration:** 60 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Cloud concepts | 25–30% | IaaS/PaaS/SaaS, cloud models, CapEx vs OpEx, consumption model |
| Azure architecture | 15–20% | Regions, AZs, resource groups, subscriptions, management groups |
| Azure services | 30–35% | Compute, networking, storage, databases |
| Management & governance | 25–30% | Cost management, Azure Policy, RBAC, Tags, Blueprints, Compliance |

### Key Concepts to Know

**Cloud Service Models:**
```
IaaS – You manage: OS, runtime, apps, data
       Azure manages: hardware, virtualization, networking
       Examples: Azure VMs, Azure Virtual Network

PaaS – You manage: apps, data
       Azure manages: OS, runtime, middleware, infrastructure
       Examples: App Service, Azure SQL Database, Azure Functions

SaaS – Azure manages: everything
       You configure: settings, users
       Examples: Microsoft 365, Dynamics 365, Azure DevOps (as a service)
```

**Cloud Deployment Models:**

| Model | Description | Examples |
|---|---|---|
| Public Cloud | Resources on Azure's shared infrastructure | Default Azure |
| Private Cloud | Azure resources dedicated to one org | Azure Dedicated Host |
| Hybrid Cloud | On-premises + cloud connected | ExpressRoute, Azure Arc |
| Multi-Cloud | Multiple cloud providers | Azure + AWS + GCP |

**CapEx vs OpEx:**
- **CapEx (Capital Expenditure):** Upfront investment (buy servers). Traditional on-prem.
- **OpEx (Operational Expenditure):** Pay-as-you-go (Azure subscription). Cloud model.

### Common AZ-900 Exam Traps

- **Availability Zones** = physically separate datacenters within a region
- **Region Pairs** = separate regions paired for DR (not the same as AZs)
- **Resource Groups** = logical containers; resources can only be in one RG
- **Azure Subscriptions** trust one Entra ID tenant
- **SLA goes UP when you add redundancy** (parallel services)
- **SLA goes DOWN when services are in series** (composite SLA = multiplication)

---

## 3. AZ-104: Azure Administrator

**Audience:** Azure administrators managing day-to-day operations  
**Duration:** 120 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Manage Azure identities and governance | 20–25% | Entra ID, RBAC, Policy, subscriptions, resource groups |
| Implement and manage storage | 15–20% | Storage accounts, blob, file sync, storage tiers, SAS |
| Deploy and manage compute | 20–25% | VMs, VMSS, App Service, ACI, Azure Kubernetes Service (basic) |
| Implement and manage networking | 15–20% | VNet, NSG, routing, LB, App Gateway, DNS, VPN, peering |
| Monitor and maintain Azure resources | 10–15% | Monitor, Log Analytics, alerts, backup, Site Recovery |

### Must-Know CLI Commands

```bash
# --- Identity & Access ---
az ad user create / list / update / delete
az ad group create / member add / member list
az role assignment create / list / delete
az role definition create / list

# --- Resource Management ---
az group create / list / delete
az resource list / move / tag
az lock create / list / delete
az policy assignment create / list
az policy state list

# --- Compute ---
az vm create / start / stop / restart / delete / resize
az vm availability-set create
az vmss create / scale
az webapp create / deploy
az appservice plan create

# --- Storage ---
az storage account create / list / update
az storage container create / list
az storage blob upload / download / copy
az storage account keys list
az storage account generate-sas

# --- Networking ---
az network vnet create / subnet create
az network nsg create / rule create
az network public-ip create
az network nic create
az network lb create / rule create
az network application-gateway create
az network dns zone create / record-set a add-record
az network vnet-gateway create
az network vpn-connection create

# --- Monitor ---
az monitor metrics list
az monitor log-analytics workspace create
az monitor diagnostic-settings create
az monitor alert create
az backup vault create / protection enable-for-vm
```

### Common AZ-104 Topics Deep-Dive

**VM Sizing Naming Convention:**
```
Standard_D4s_v3
         │││  │
         │││  └─ v3 = generation
         ││└──── s = Premium SSD support
         │└───── 4 = number of vCPUs
         └────── D = general purpose (D=General, E=Memory, F=Compute, N=GPU, L=Storage)
```

**Storage Access Tiers:**
```
Hot   → frequently accessed data     → highest storage cost, lowest access cost
Cool  → infrequently accessed (30d)  → lower storage cost, higher access cost
Cold  → rarely accessed (90d)        → lower storage cost, higher access cost  
Archive → rarely accessed (180d)     → lowest storage cost, highest access cost (rehydrate first)
```

**NSG Rule Priority:** Lower number = higher priority. 100 overrides 200. Default rules are 65000+.

---

## 4. AZ-204: Azure Developer

**Audience:** Developers building apps on Azure  
**Duration:** 120 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Develop Azure compute solutions | 25–30% | VMs, App Service, Azure Functions, ACI, container workflows |
| Develop for Azure storage | 15–20% | Blob (SDK), Cosmos DB (SDK, partition key, consistency), Table, Queue |
| Implement Azure security | 20–25% | Key Vault, Managed Identity, App Registrations, MSAL, SAS |
| Monitor, troubleshoot, optimize | 15–20% | Application Insights, caching (Redis), CDN, message queues |
| Connect to and consume Azure services | 15–20% | API Management, Event Grid, Event Hub, Service Bus, Logic Apps |

### Must-Know SDK Code Patterns

```python
# Blob Storage SDK
from azure.storage.blob import BlobServiceClient, BlobClient
from azure.identity import DefaultAzureCredential

blob_service = BlobServiceClient(
    account_url="https://myaccount.blob.core.windows.net",
    credential=DefaultAzureCredential()
)
container_client = blob_service.get_container_client("mycontainer")
blob_client = container_client.get_blob_client("myblob.txt")

# Upload
blob_client.upload_blob(b"Hello World", overwrite=True)

# Download
data = blob_client.download_blob().readall()

# Generate SAS URL
from azure.storage.blob import generate_blob_sas, BlobSasPermissions
from datetime import datetime, timedelta, timezone

sas_token = generate_blob_sas(
    account_name="myaccount",
    container_name="mycontainer",
    blob_name="myblob.txt",
    account_key="<account-key>",
    permission=BlobSasPermissions(read=True),
    expiry=datetime.now(timezone.utc) + timedelta(hours=1)
)
sas_url = f"https://myaccount.blob.core.windows.net/mycontainer/myblob.txt?{sas_token}"
```

```python
# Cosmos DB SDK
from azure.cosmos import CosmosClient, PartitionKey

client = CosmosClient(url="https://myaccount.documents.azure.com:443/", credential=DefaultAzureCredential())
db = client.get_database_client("mydb")
container = db.get_container_client("mycontainer")

# Create item
container.upsert_item({"id": "001", "category": "electronics", "name": "Laptop", "price": 1200})

# Query
items = container.query_items(
    query="SELECT * FROM c WHERE c.category = @cat",
    parameters=[{"name": "@cat", "value": "electronics"}],
    enable_cross_partition_query=True
)

# Point read (fastest – uses partition key + id)
item = container.read_item(item="001", partition_key="electronics")
```

### Azure Functions Triggers

| Trigger | Description |
|---|---|
| HTTP | REST API endpoint |
| Timer | Cron schedule |
| Blob | New/updated blob in container |
| Queue | Message in Azure Queue |
| Service Bus | Message in Service Bus queue/topic |
| Event Grid | Event from Event Grid |
| Event Hub | Stream of events |
| Cosmos DB | Change feed |

### AZ-204 Key Concepts

**Cosmos DB Consistency Levels (strong → weak):**
```
Strong         → read always sees latest write (highest consistency, highest latency)
Bounded staleness → reads lag by K operations or T time
Session        → consistency within a session (most popular)
Consistent prefix → reads see operations in order, but may lag
Eventual       → weakest, highest availability, lowest latency
```

**App Service Deployment Slots:**
- Slots are separate running instances of the app
- Swap slots for zero-downtime deployments
- Only Standard, Premium, Isolated tiers support slots
- Settings can be "slot-specific" (not swapped) or swapped with the slot

---

## 5. AZ-305: Azure Solutions Architect

**Audience:** Senior architects designing Azure solutions  
**Prereq:** AZ-104 recommended  
**Duration:** 120 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Design identity, governance, and monitoring | 25–30% | Entra ID, PIM, RBAC, Policy, Monitor, Log Analytics, Cost |
| Design data storage solutions | 20–25% | SQL, Cosmos DB, ADLS, Blob, Storage lifecycle, data redundancy |
| Design infrastructure solutions | 30–35% | VMs, AKS, App Service, networking, HA/DR, migration |
| Design business continuity solutions | 15–20% | Backup, ASR, RTO/RPO, failover, geo-redundancy |

### Key Architect Decisions

**VM vs App Service vs AKS vs Functions:**
```
Scenario                        Best Choice
────────────────────────────    ─────────────────────
Need OS-level control           Azure VM
Lift-and-shift web app          App Service
Microservices with containers   AKS
Event-driven, short tasks       Azure Functions
Serverless container (no K8s)   Azure Container Apps
Single container, short-lived   Azure Container Instances
```

**Storage Choice Decision Tree:**
```
Structured + relational + ACID?       → Azure SQL Database
Semi-structured + massive scale?      → Cosmos DB (choose API)
Analytical + big data?                → Azure Synapse / ADLS Gen2
Unstructured files (blobs)?           → Azure Blob Storage
SMB/NFS file shares?                  → Azure Files
Message queue?                        → Storage Queue (simple) / Service Bus (enterprise)
Cache?                                → Azure Cache for Redis
```

---

## 6. AZ-400: Azure DevOps Engineer

**Audience:** DevOps engineers implementing CI/CD, IaC, and DevSecOps  
**Prereq:** AZ-104 or AZ-204  
**Duration:** 120 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Configure processes and communications | 10–15% | Azure Boards, dashboards, notifications, work item linking |
| Design and implement source control | 15–20% | Git strategies (GitFlow, trunk-based), PR policies, branch protection |
| Design and implement build and release pipelines | 40–45% | YAML pipelines, stages/jobs/steps, templates, approvals, environments |
| Develop security and compliance plan | 10–15% | SAST, DAST, SCA, secret scanning, dependency scanning, SBOM |
| Implement instrumentation strategy | 10–15% | App Insights, feature flags, canary releases, deployment rings |

### Key Pipeline Concepts

**YAML Pipeline Structure:**
```yaml
trigger: [main]

variables:
  buildConfiguration: Release

stages:
- stage: Build
  jobs:
  - job: BuildApp
    pool: {vmImage: ubuntu-latest}
    steps:
    - task: DotNetCoreCLI@2
      inputs: {command: build, arguments: --configuration $(buildConfiguration)}

- stage: Deploy
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployProd
    environment: production      # approval gate here
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo deploying
```

**DevSecOps Tools on Azure:**
| Stage | Tool | Purpose |
|---|---|---|
| Commit | git-secrets, gitleaks | Secret scanning |
| Build | SonarCloud, Checkmarx | SAST |
| Package | WhiteSource Bolt, OWASP Dependency Check | SCA |
| Deploy | Checkov, tfsec | IaC scanning |
| Runtime | Defender for Containers | Runtime threat detection |
| Post-deploy | OWASP ZAP, Burp Suite | DAST |

---

## 7. AZ-500: Azure Security Engineer

**Audience:** Security engineers implementing Azure security controls  
**Duration:** 120 minutes | **Questions:** 40–60 | **Passing score:** 700/1000  
**Cost:** $165 USD

### Exam Domains

| Domain | Weight | Key Topics |
|---|---|---|
| Manage identity and access | 25–30% | Entra ID, MFA, Conditional Access, PIM, App Registrations, Managed Identity |
| Secure networking | 20–25% | NSG, Firewall, DDoS, Private Endpoints, VPN, Bastion |
| Secure compute, storage, and databases | 20–25% | Disk encryption, storage encryption, SQL TDE, AKS security |
| Manage security operations | 25–30% | Defender for Cloud, Sentinel, Key Vault, Monitor, Policy |

---

## 8. Specialty Certifications

| Cert | Role | Key Areas |
|---|---|---|
| **AI-102** | Azure AI Engineer | Azure OpenAI, Cognitive Services, Azure ML, Bot Service |
| **DP-100** | Azure Data Scientist | Azure ML (experiments, pipelines, endpoints, AutoML) |
| **DP-203** | Azure Data Engineer | Synapse Analytics, Data Factory, Databricks, ADLS, Event Hub |
| **DP-300** | Azure DBA | Azure SQL, Cosmos DB, PostgreSQL, performance tuning, HA |
| **AZ-140** | Azure Virtual Desktop | AVD setup, host pools, FSLogix, monitoring |
| **AZ-700** | Azure Network Engineer | VNet, ExpressRoute, VPN, Front Door, Load Balancer, DNS |
| **SC-900** | Security Fundamentals | Zero Trust, compliance, Purview, Defender, Entra ID basics |
| **SC-300** | Identity Administrator | Entra ID, B2B, B2C, PIM, Conditional Access, Entitlement Mgmt |

---

## 9. Exam Tips & Strategies

### Before the Exam

```
✅ Read each question carefully – Azure exams use specific wording
✅ Practice in the Azure Portal and Azure CLI for hands-on questions
✅ Know the difference between similar services (e.g., Load Balancer vs App Gateway)
✅ Do at least 3 full practice exams (MeasureUp, Whizlabs, ExamTopics)
✅ Review Microsoft Learn modules for each domain
✅ Book the exam before you feel 100% ready (creates urgency)
```

### During the Exam

```
✅ Answer all questions – no penalty for wrong answers
✅ Mark uncertain questions and review at the end
✅ For scenario questions: identify the constraints first (cost, performance, security)
✅ For "MOST cost-effective" – usually the simplest/smallest solution
✅ For "highest availability" – usually zone-redundant + multiple instances
✅ Watch out for distractor answers that sound right but violate a stated constraint
✅ Case studies: read the requirements section first, then the scenario
```

### Common Trick Questions

| Question Pattern | Watch Out For |
|---|---|
| "You need to ensure ONLY…" | Policy with **Deny** effect, not Audit |
| "With the LEAST administrative effort" | Use built-in roles/policies, not custom |
| "Minimize cost" | Use the correct tier (Standard vs Premium), consider reservations |
| "Zero downtime deployment" | Deployment slots, canary, blue-green |
| "Data must not leave the region" | LRS or ZRS (not GRS which goes to paired region) |
| "Audit compliance without blocking" | Audit effect (not Deny) |
| "Automatically remediate" | DeployIfNotExists or Modify policy effect |

---

## 10. Study Resources

### Free Resources

| Resource | URL | Best For |
|---|---|---|
| Microsoft Learn | learn.microsoft.com | Official study paths, free sandbox |
| Azure Documentation | docs.microsoft.com/azure | Deep-dive reference |
| Azure Free Account | azure.microsoft.com/free | Hands-on practice ($200 credits) |
| John Savill's YouTube | youtube.com/@NTFAQGuy | AZ-104, AZ-305, AZ-400 masterclasses |
| Azure Charts | azurecharts.com | Visual service comparison |
| Azure Architecture Center | learn.microsoft.com/azure/architecture | Reference architectures |
| Microsoft Azure Blog | azure.microsoft.com/blog | Latest announcements |

### Paid Resources

| Resource | Best For |
|---|---|
| A Cloud Guru / Pluralsight | Video courses for all Azure certs |
| Udemy (Scott Duffy, Alan Rodrigues) | Affordable video courses |
| MeasureUp Practice Tests | Official Microsoft partner, high-quality practice exams |
| Whizlabs | Practice exams with detailed explanations |
| ExamTopics | Community practice questions (verify answers – some are wrong) |

### Study Plan Template (AZ-104 – 8 weeks)

| Week | Topics | Activities |
|---|---|---|
| 1 | Entra ID, RBAC, Policy, Subscriptions | Microsoft Learn path + portal practice |
| 2 | Storage accounts, Blob, File Sync, Tiers | Create storage account, upload blobs with CLI |
| 3 | VMs, Availability Sets, VMSS, Backups | Deploy VM, configure backup, scale VMSS |
| 4 | Networking: VNet, NSG, LB, App Gateway | Build hub-spoke network, configure NSG rules |
| 5 | Networking: DNS, VPN, Peering, Bastion | Configure VNet peering, Azure Bastion |
| 6 | App Service, ACI, AKS basics | Deploy web app, set up deployment slot |
| 7 | Monitor, Log Analytics, Alerts | Create dashboard, write KQL query, set up alert |
| 8 | Practice exams + review weak areas | 3 full practice exams, review incorrect answers |

### Key Azure Service Comparison Cards

**Event Hub vs Service Bus vs Storage Queue:**

| Feature | Storage Queue | Service Bus Queue | Event Hub |
|---|---|---|---|
| Max message size | 64 KB | 256 KB – 100 MB | 1 MB |
| Max retention | 7 days | 14 days | 7–90 days |
| Ordering | FIFO (best effort) | FIFO (guaranteed) | Per partition |
| Throughput | Unlimited | Up to 1M msg/sec | Millions/sec |
| Use case | Simple decoupling | Enterprise messaging | Event streaming, analytics |
| Dead letter queue | ❌ | ✅ | ❌ |
| Sessions (grouping) | ❌ | ✅ | Via partition key |

**Load Balancer vs Application Gateway vs Front Door vs Traffic Manager:**

| Feature | Load Balancer | App Gateway | Front Door | Traffic Manager |
|---|---|---|---|---|
| Layer | L4 (TCP/UDP) | L7 (HTTP/S) | L7 + CDN | DNS only |
| Scope | Regional | Regional | Global | Global |
| SSL termination | ❌ | ✅ | ✅ | ❌ |
| WAF | ❌ | ✅ | ✅ | ❌ |
| Path-based routing | ❌ | ✅ | ✅ | ❌ |
| Caching/CDN | ❌ | ❌ | ✅ | ❌ |
| Health probes | ✅ | ✅ | ✅ | ✅ |
| Use case | VM load balancing | AKS ingress, web tier | Global web apps | DR routing |
