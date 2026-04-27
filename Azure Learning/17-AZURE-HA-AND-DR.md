# 17 – Azure High Availability & Disaster Recovery

---

## Table of Contents
1. [Foundational Concepts](#1-foundational-concepts)
2. [High Availability Patterns](#2-high-availability-patterns)
3. [Azure Backup](#3-azure-backup)
4. [Azure Site Recovery](#4-azure-site-recovery)
5. [Geo-Redundancy & Multi-Region Architecture](#5-geo-redundancy--multi-region-architecture)
6. [Chaos Engineering with Azure Chaos Studio](#6-chaos-engineering-with-azure-chaos-studio)
7. [Disaster Recovery Runbooks](#7-disaster-recovery-runbooks)
8. [Resilience Checklist](#8-resilience-checklist)

---

## 1. Foundational Concepts

### SLA, SLO, SLI

| Term | Definition | Example |
|---|---|---|
| **SLA** (Service Level Agreement) | Contract between provider and customer | Microsoft guarantees 99.99% uptime for Azure SQL DB |
| **SLO** (Service Level Objective) | Internal target you set for your system | Your team targets 99.95% uptime for your app |
| **SLI** (Service Level Indicator) | Actual measured metric | % of successful requests in last 30 days |
| **Error Budget** | SLO – actual availability | If SLO=99.9%, error budget = 0.1% = 43.8 min/month |

### Availability % ↔ Downtime/Year

| Availability | Downtime per Year | Downtime per Month | Downtime per Week |
|---|---|---|---|
| 99% | 87 hours 36 min | 7 hours 18 min | 1 hour 41 min |
| 99.5% | 43 hours 48 min | 3 hours 39 min | 50 min |
| 99.9% | 8 hours 46 min | 43.8 min | 10 min |
| 99.95% | 4 hours 23 min | 21.9 min | 5 min |
| 99.99% | 52 min 35 sec | 4.38 min | 1 min |
| 99.999% | 5 min 15 sec | 26 sec | 6 sec |

### RTO and RPO

```
Timeline of an Outage:
──────────────────────────────────────────────────────────────
Last backup     Disaster occurs      Service restored
     │                │                      │
     ├──── RPO ───────┤                      │
     │                ├──────── RTO ─────────┤
     │                │                      │
  Time T0         Time T1                Time T2

RPO (Recovery Point Objective) = T1 - T0
  → Maximum acceptable data loss
  → "How old can the recovered data be?"
  → Drives backup frequency

RTO (Recovery Time Objective) = T2 - T1
  → Maximum acceptable downtime
  → "How fast must we recover?"
  → Drives recovery automation
```

### Composite SLA

When services are **in series** (all must work):
```
Composite SLA = SLA_A × SLA_B × SLA_C
Example: 99.9% × 99.9% × 99.9% = 99.7%
```

When services are **in parallel** (any one working is enough):
```
Composite SLA = 1 - ((1 - SLA_A) × (1 - SLA_B))
Example: 1 - (0.001 × 0.001) = 1 - 0.000001 = 99.9999%
```

### Availability Zones

Azure Availability Zones are physically separate datacenters within a region:
- Each zone has independent power, cooling, and networking
- Regions with AZs: East US, West US 2, West Europe, North Europe, UK South, Australia East, and 30+ more
- Latency between zones: < 2ms

### Region Pairs

| Primary Region | Paired Region | Geography |
|---|---|---|
| East US | West US | United States |
| East US 2 | Central US | United States |
| North Europe | West Europe | Europe |
| UK South | UK West | United Kingdom |
| Australia East | Australia Southeast | Australia |
| Southeast Asia | East Asia | Asia Pacific |

Benefits of region pairs: sequential platform updates, data residency, prioritized recovery.

---

## 2. High Availability Patterns

### Compute HA

**Availability Set:**
```
VMs spread across Fault Domains (separate racks) and Update Domains (staged maintenance)

FD = 2-3 (separate physical hardware)
UD = 5-20 (sequential OS/host updates)

SLA: 99.95% with 2+ VMs in same Availability Set
```

**Availability Zones:**
```
VM1 ──── Zone 1 (Datacenter A)
VM2 ──── Zone 2 (Datacenter B)
VM3 ──── Zone 3 (Datacenter C)

SLA: 99.99% with 2+ VMs in different zones
```

**VMSS (Virtual Machine Scale Sets):**
```bash
# Create zone-redundant VMSS
az vmss create \
  --resource-group myrg \
  --name myapp-vmss \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v3 \
  --instance-count 3 \
  --zones 1 2 3 \
  --load-balancer myapp-lb \
  --upgrade-policy-mode Automatic \
  --admin-username azureuser \
  --generate-ssh-keys
```

### Storage Replication Options

| Replication | Copies | Scope | Failover | Best For |
|---|---|---|---|---|
| **LRS** (Locally Redundant) | 3 | Single datacenter | Manual | Dev/test, cheap |
| **ZRS** (Zone Redundant) | 3 | 3 zones in 1 region | Automatic | Zone failure protection |
| **GRS** (Geo Redundant) | 6 | Primary + paired region | Microsoft-initiated | DR, compliance |
| **GZRS** (Geo+Zone Redundant) | 6 | 3 zones + paired region | Microsoft-initiated | Mission critical |
| **RA-GRS** (Read-Access GRS) | 6 | Primary + paired region | Manual customer-initiated | Read from secondary |
| **RA-GZRS** | 6 | 3 zones + paired region | Manual customer-initiated | Highest durability |

```
LRS:
  Primary Region
  ┌─────────────────────────────┐
  │  DC-1: Copy 1, Copy 2, Copy 3 │
  └─────────────────────────────┘

ZRS:
  Primary Region
  ┌──────┐  ┌──────┐  ┌──────┐
  │Zone 1│  │Zone 2│  │Zone 3│
  │Copy 1│  │Copy 2│  │Copy 3│
  └──────┘  └──────┘  └──────┘

GRS:
  Primary Region          Paired Region
  ┌──────────┐           ┌──────────┐
  │ LRS (3x) │──────────▶│ LRS (3x) │
  └──────────┘           └──────────┘
```

### AKS High Availability

```yaml
# Zone-redundant AKS node pool
az aks create \
  --resource-group myrg \
  --name mycluster \
  --node-count 3 \
  --zones 1 2 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --network-plugin azure

# PodDisruptionBudget - minimum available pods during voluntary disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

```yaml
# Pod Topology Spread Constraints - spread pods across zones
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

---

## 3. Azure Backup

### Recovery Services Vault vs Backup Vault

| Feature | Recovery Services Vault | Backup Vault |
|---|---|---|
| Azure VMs | ✅ | ❌ |
| SQL in Azure VM | ✅ | ❌ |
| SAP HANA | ✅ | ❌ |
| Azure Files | ✅ | ❌ |
| Azure Disks | ❌ | ✅ |
| Azure Blobs | ❌ | ✅ |
| AKS | ❌ | ✅ |
| PostgreSQL | ❌ | ✅ |

### Backup Policy (Azure VM)

| Schedule | Retention |
|---|---|
| Daily (10 PM UTC) | 30 days |
| Weekly (Sunday) | 12 weeks |
| Monthly (1st Sunday) | 12 months |
| Yearly (January 1st) | 5 years |

### CLI Commands

```bash
# Create Recovery Services Vault
az backup vault create \
  --name myvault \
  --resource-group myrg \
  --location eastus \
  --storage-redundancy GeoRedundant

# Enable backup for Azure VM
az backup protection enable-for-vm \
  --vault-name myvault \
  --resource-group myrg \
  --vm myvm \
  --policy-name DefaultPolicy

# List backup jobs
az backup job list \
  --vault-name myvault \
  --resource-group myrg \
  --output table

# Trigger on-demand backup
az backup protection backup-now \
  --vault-name myvault \
  --resource-group myrg \
  --container-name myvm \
  --item-name myvm \
  --backup-management-type AzureIaasVM \
  --retain-until 2024-12-31

# List restore points
az backup recoverypoint list \
  --vault-name myvault \
  --resource-group myrg \
  --container-name myvm \
  --item-name myvm \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --output table

# Enable soft delete
az backup vault backup-properties set \
  --vault-name myvault \
  --resource-group myrg \
  --soft-delete-feature-state Enable
```

---

## 4. Azure Site Recovery

ASR provides continuous replication and orchestrated failover for disaster recovery.

### ASR vs Azure Backup

| Aspect | Azure Backup | Azure Site Recovery |
|---|---|---|
| Purpose | Data backup and restore | DR – business continuity |
| RPO | Hours (scheduled backups) | Seconds to minutes (continuous) |
| RTO | Hours (restore from backup) | Minutes to 1-2 hours |
| Orchestration | Manual restore | Automated recovery plans |
| Cost model | Per instance + storage | Per protected instance |

### ASR Replication Flow

```
Source Region                       Target Region
────────────────                    ─────────────────
VM (running)                        VM (standby – off)
  │                                     │
  ├── Azure Site Recovery Agent         │
  │   (installed automatically)         │
  │                                     │
  ▼                                     ▼
Cache Storage Account ──────────▶ Replica Managed Disk
(staging area)                   (synced every 5-10 min)
                                  Target VNet (pre-created)
                                  Recovery Point: RPO 30 sec
```

### CLI Commands

```bash
# Create Recovery Services Vault for ASR
az recovery-services vault create \
  --name dr-vault \
  --resource-group dr-rg \
  --location westus  # target region

# Enable replication for a VM (done via Portal or PowerShell due to complexity)
# PowerShell example:
New-AzRecoveryServicesAsrReplicationProtectedItem `
  -AzureToAzure `
  -AzureVmId "/subscriptions/<sub>/resourceGroups/prod-rg/providers/Microsoft.Compute/virtualMachines/myvm" `
  -Name "myvm-replication" `
  -RecoveryResourceGroupId "/subscriptions/<sub>/resourceGroups/dr-rg" `
  -RecoveryAzureStorageAccountId "<storage-account-id>" `
  -RecoveryReplicaDiskAccountType "Premium_LRS" `
  -RecoveryVirtualNetworkId "<target-vnet-id>"
```

### Failover Types

| Type | Description | Impact |
|---|---|---|
| **Test Failover** | Spins up replica VM in isolated VNet | Non-disruptive, test connectivity |
| **Planned Failover** | Graceful failover (stop source, start replica) | Planned maintenance, ~0 data loss |
| **Unplanned Failover** | Immediate failover, source may be down | Some data loss possible |

---

## 5. Geo-Redundancy & Multi-Region Architecture

### Active-Passive vs Active-Active

```
Active-Passive:
  Region A (Primary)  ──── Traffic Manager ◀── All traffic
  Region B (Standby)  ──── Traffic Manager ✗ (promoted on failure)
  
  RPO: minutes, RTO: 5-30 minutes, Cost: lower

Active-Active:
  Region A ──────────── Traffic Manager ─────── Users
  Region B ──────────── Traffic Manager ─────── Users
  (both regions serve traffic simultaneously)
  
  RPO: near-zero, RTO: near-zero, Cost: 2x, Complexity: high
```

### Azure Traffic Manager Routing Methods

| Method | How It Routes | Use Case |
|---|---|---|
| **Priority** | Route to primary; failover to secondary | Active-passive DR |
| **Weighted** | Split traffic by percentage (e.g., 80/20) | Gradual migration, canary |
| **Performance** | Route to lowest-latency endpoint | Global user base |
| **Geographic** | Route based on user's country/region | Data residency compliance |
| **Multivalue** | Return multiple healthy endpoints | DNS-based load balancing |
| **Subnet** | Route based on source IP range | A/B testing by IP range |

### SQL Auto-Failover Groups

```bash
# Create failover group for Azure SQL
az sql failover-group create \
  --name my-failover-group \
  --partner-server dr-sql-server \
  --resource-group prod-rg \
  --server prod-sql-server \
  --add-db mydb \
  --failover-policy Automatic \
  --grace-period 1  # hours before automatic failover

# Check failover group status
az sql failover-group show \
  --name my-failover-group \
  --resource-group prod-rg \
  --server prod-sql-server

# Manual failover (e.g., for planned DR drill)
az sql failover-group set-primary \
  --name my-failover-group \
  --resource-group prod-rg \
  --server dr-sql-server
```

---

## 6. Chaos Engineering with Azure Chaos Studio

Chaos Studio validates that your system can withstand failures.

### Supported Fault Types

| Target | Fault | Description |
|---|---|---|
| Azure VM | CPU pressure | Spike CPU to 95% |
| Azure VM | Memory pressure | Consume 90% of RAM |
| Azure VM | Kill process | Kill a named process |
| AKS | Pod kill | Delete pods matching selector |
| AKS | Node drain | Cordon and drain a node |
| Cosmos DB | Failover | Force regional failover |
| Azure Cache (Redis) | Reboot | Reboot primary/secondary |
| Virtual Network | Network disconnect | Block outbound traffic |

```bash
# Enable Chaos Studio on a VM target
az rest --method put \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.Compute/virtualMachines/myvm/providers/Microsoft.Chaos/targets/Microsoft-VirtualMachine?api-version=2023-11-01" \
  --body '{"properties": {}}'

# Create and run a chaos experiment via Portal or CLI
az chaos experiment create \
  --name "vm-cpu-experiment" \
  --resource-group myrg \
  --experiment-file experiment.json

# Start experiment
az chaos experiment start \
  --name "vm-cpu-experiment" \
  --resource-group myrg
```

---

## 7. Disaster Recovery Runbooks

### BIA (Business Impact Analysis) Template

| Application | Tier | RTO Target | RPO Target | Revenue Impact/Hour | DR Strategy |
|---|---|---|---|---|---|
| Payment API | Tier 1 | 15 min | 1 min | $50,000 | Active-Active |
| Customer Portal | Tier 1 | 1 hour | 15 min | $10,000 | Active-Passive |
| Reporting Service | Tier 2 | 4 hours | 1 hour | $500 | Backup + Restore |
| Internal Admin | Tier 3 | 24 hours | 24 hours | $0 | Manual restore |

### AKS Application Failover Runbook

```bash
# Step 1: Confirm primary region is degraded
kubectl get nodes --context=aks-eastus
# Expected: nodes NotReady or unreachable

# Step 2: Verify database replication is current
az sql failover-group show --name prod-fg --server prod-sql

# Step 3: Update Traffic Manager to point to DR region
az network traffic-manager endpoint update \
  --name primary-endpoint \
  --profile-name prod-tm \
  --resource-group prod-rg \
  --type externalEndpoints \
  --endpoint-status Disabled

az network traffic-manager endpoint update \
  --name dr-endpoint \
  --profile-name prod-tm \
  --resource-group prod-rg \
  --type externalEndpoints \
  --endpoint-status Enabled

# Step 4: Initiate SQL failover to DR region
az sql failover-group set-primary \
  --name prod-fg \
  --resource-group dr-rg \
  --server dr-sql-server

# Step 5: Scale up AKS in DR region
az aks scale \
  --name dr-aks \
  --resource-group dr-rg \
  --node-count 10

# Step 6: Verify app health in DR region
kubectl get pods --context=aks-westus --namespace prod
curl https://dr.myapp.com/health

# Step 7: Send all-clear notification with RTO achieved
# Notify: incident channel, status page, executive team
```

---

## 8. Resilience Checklist

### Per-Service Checklist

| Service | HA Checklist |
|---|---|
| **VMs** | ✅ Availability Zones or Availability Set, ✅ Load Balancer with health probes, ✅ Backup enabled, ✅ ASR replication configured |
| **AKS** | ✅ Multi-zone node pool, ✅ PodDisruptionBudget, ✅ TopologySpreadConstraints, ✅ Multiple replicas, ✅ Resource limits set, ✅ Liveness + readiness probes |
| **Azure SQL** | ✅ Zone-redundant (Business Critical tier), ✅ Auto-failover group to DR region, ✅ Point-in-time restore enabled, ✅ Long-term retention configured |
| **Cosmos DB** | ✅ Multi-region reads enabled, ✅ Automatic failover enabled, ✅ Consistency level chosen appropriately |
| **App Service** | ✅ Multiple instances (≥3), ✅ Zone redundancy enabled, ✅ Deployment slots for zero-downtime deployments, ✅ Health check endpoint configured |
| **Storage** | ✅ ZRS or GZRS replication, ✅ Soft delete enabled, ✅ Versioning enabled for Blob |
| **Key Vault** | ✅ Soft delete enabled, ✅ Purge protection enabled, ✅ Private endpoint deployed |

### Architecture Review Questions

```
1. What is the RTO and RPO for each tier-1 service?
2. Is there a single point of failure anywhere in the critical path?
3. Are all stateful services replicated to a secondary region?
4. Is the failover process documented, automated, and tested?
5. When was the last DR drill? What was the outcome?
6. Are health checks configured at every layer (app, LB, Traffic Manager)?
7. Is the infrastructure deployable via IaC (Terraform/Bicep) in a new region in < 1 hour?
8. Are runbooks stored in a location accessible during an Azure outage?
9. Are communication channels defined for an incident affecting multiple teams?
10. Is Chaos Studio or equivalent testing scheduled quarterly?
```
