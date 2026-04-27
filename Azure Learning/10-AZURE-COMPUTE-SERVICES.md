# Azure Compute Services — Complete Reference Guide
## Document 10: Zero-to-Hero Compute Handbook

**Last Updated:** 2026  
**Document Version:** 1.0  
**Focus:** Every Azure Compute service — VMs, App Service, Functions, ACI, Container Apps, AKS deep-dive, Batch, VMSS

---

## TABLE OF CONTENTS

1. [Azure Virtual Machines (VMs)](#1-azure-virtual-machines)
2. [Azure App Service](#2-azure-app-service)
3. [Azure Functions (Serverless)](#3-azure-functions-serverless)
4. [Azure Container Instances (ACI)](#4-azure-container-instances-aci)
5. [Azure Container Apps](#5-azure-container-apps)
6. [Azure Kubernetes Service (AKS) — Advanced Detail](#6-azure-kubernetes-service-aks)
7. [Azure Batch](#7-azure-batch)
8. [Azure Virtual Machine Scale Sets (VMSS)](#8-virtual-machine-scale-sets-vmss)
9. [Compute Service Decision Guide](#9-compute-decision-guide)

---

## Compute Services Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AZURE COMPUTE LANDSCAPE                             │
│                                                                         │
│   CONTROL ──────────────────────────────────────────► ABSTRACTION      │
│                                                                         │
│   VMs          VMSS        App Service    Functions    Container Apps   │
│  (IaaS)       (IaaS)         (PaaS)      (Serverless)   (Serverless)   │
│   Full         Auto-         Managed       Event-        Microservice   │
│  Control      Scaled         Runtime       Driven        Platform       │
│                                                                         │
│          AKS              ACI              Batch                        │
│      (Containers)     (Single Run)       (HPC/Jobs)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

| Service | IaaS/PaaS | OS Control | Auto-Scale | Best For |
|---|---|---|---|---|
| Virtual Machines | IaaS | Full | Manual/VMSS | Lift-and-shift, custom OS |
| VMSS | IaaS | Full | Yes | Auto-scaled VM fleets |
| App Service | PaaS | No | Yes | Web apps, APIs |
| Azure Functions | Serverless | No | Yes (auto) | Event-driven tasks |
| ACI | PaaS | No | No | Single container runs |
| Container Apps | Serverless | No | Yes (KEDA) | Microservices containers |
| AKS | PaaS/IaaS | Partial | Yes | Full Kubernetes |
| Azure Batch | PaaS | Partial | Yes | HPC workloads |

---

## 1. Azure Virtual Machines

### What is a VM and When to Use It

An Azure Virtual Machine (VM) is an Infrastructure-as-a-Service (IaaS) offering that gives you a full operating system running in Microsoft's datacenter. You control everything: OS, middleware, runtime, application. Azure manages only the physical hardware, virtualization layer, and datacenter infrastructure.

**Use a VM when:**
- You need full OS-level control (custom kernel, specific OS version, registry edits)
- You're lifting and shifting an on-premises workload
- Your software requires specific drivers, COM objects, or native system libraries
- You need persistent stateful compute that runs 24/7
- You're running software that cannot be containerized easily (legacy apps, licensed software)
- You need Remote Desktop (RDP) or SSH access to the machine

**Do NOT use a VM when:**
- You're deploying a web app or API (use App Service instead)
- You're running event-driven functions (use Azure Functions)
- You need container orchestration (use AKS)
- Cost efficiency is critical and workload is bursty (use Functions or Container Apps)

```
┌─────────────────────────────────────────────────────┐
│              AZURE VIRTUAL MACHINE                  │
│                                                     │
│  Physical Server (Azure Datacenter)                 │
│  ┌─────────────────────────────────────────────┐   │
│  │  Hypervisor (Azure Fabric Controller)       │   │
│  │  ┌────────────────────────────────────┐    │   │
│  │  │ Your VM                            │    │   │
│  │  │  ┌────────────┐ ┌──────────────┐  │    │   │
│  │  │  │ OS (Win/   │ │  Your App    │  │    │   │
│  │  │  │  Linux)    │ │  Code        │  │    │   │
│  │  │  └────────────┘ └──────────────┘  │    │   │
│  │  │  ┌────────────────────────────┐   │    │   │
│  │  │  │ Managed Disk (vHD)         │   │    │   │
│  │  │  └────────────────────────────┘   │    │   │
│  │  │  ┌────────────────────────────┐   │    │   │
│  │  │  │ NIC → VNet → Internet      │   │    │   │
│  │  │  └────────────────────────────┘   │    │   │
│  │  └────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

### VM Sizes and Families

Azure VM sizes follow this naming convention:

```
[Family][Sub-family][# of vCPUs][Constrained vCPUs][Add. features][Accelerator Type]_v[Version]

Example: Standard_D4s_v5
  Standard   = Tier
  D          = General Purpose family
  4          = 4 vCPUs
  s          = Premium SSD support
  v5         = Version 5 (latest hardware gen)
```

#### B-Series — Burstable (Cost-Efficient for Low Avg CPU)

B-series VMs are ideal for workloads that don't need full CPU performance continuously. They accumulate CPU credits during idle periods and spend them during bursts.

| Size | vCPUs | RAM | Base CPU % | Max CPU % | Use Case |
|---|---|---|---|---|---|
| Standard_B1s | 1 | 1 GB | 10% | 100% | Dev/test, micro services |
| Standard_B2s | 2 | 4 GB | 40% | 200% | Small web apps |
| Standard_B4ms | 4 | 16 GB | 90% | 400% | Medium apps with spikes |
| Standard_B8ms | 8 | 32 GB | 135% | 800% | CI/CD agents |

**When to use B-series:** Dev/test environments, small web apps, light databases, CI/CD build agents. Cheapest option for workloads with low average CPU.

**When NOT to use B-series:** Production databases, sustained heavy computation — you'll run out of CPU credits and throttle.

#### D-Series — General Purpose

D-series offers a balanced CPU-to-memory ratio (1:4). Most commonly used for web servers, application servers, and moderate databases.

| Size | vCPUs | RAM | Temp Storage | Network |
|---|---|---|---|---|
| Standard_D2s_v5 | 2 | 8 GB | Remote only | 12.5 Gbps |
| Standard_D4s_v5 | 4 | 16 GB | Remote only | 12.5 Gbps |
| Standard_D8s_v5 | 8 | 32 GB | Remote only | 12.5 Gbps |
| Standard_D16s_v5 | 16 | 64 GB | Remote only | 12.5 Gbps |
| Standard_D32s_v5 | 32 | 128 GB | Remote only | 16 Gbps |
| Standard_D64s_v5 | 64 | 256 GB | Remote only | 16 Gbps |

**When to use D-series:** General-purpose workloads, web servers, application tiers, light databases, development servers.

#### E-Series — Memory Optimized

E-series has a high memory-to-CPU ratio (1:8), designed for in-memory databases, large caches, and analytics.

| Size | vCPUs | RAM | Use Case |
|---|---|---|---|
| Standard_E2s_v5 | 2 | 16 GB | Small in-memory DB |
| Standard_E4s_v5 | 4 | 32 GB | Redis, MongoDB |
| Standard_E8s_v5 | 8 | 64 GB | SQL Server |
| Standard_E16s_v5 | 16 | 128 GB | SAP HANA small |
| Standard_E64s_v5 | 64 | 512 GB | Large in-memory analytics |
| Standard_E96s_v5 | 96 | 672 GB | SAP HANA large |

**When to use E-series:** SQL Server with large buffer pools, SAP HANA, in-memory caches (Redis), analytics workloads.

#### F-Series — Compute Optimized

F-series has a high CPU-to-memory ratio (1:2). Designed for CPU-intensive workloads.

| Size | vCPUs | RAM | Use Case |
|---|---|---|---|
| Standard_F2s_v2 | 2 | 4 GB | Gaming servers |
| Standard_F4s_v2 | 4 | 8 GB | Web front ends |
| Standard_F8s_v2 | 8 | 16 GB | Batch processing |
| Standard_F32s_v2 | 32 | 64 GB | High-throughput compute |

**When to use F-series:** Web front-ends, batch processing, analytics, gaming servers, application servers where CPU is the bottleneck.

#### N-Series — GPU-Enabled

N-series VMs include NVIDIA GPUs for graphics rendering, machine learning, and parallel computing.

| Sub-Family | GPU Type | Use Case |
|---|---|---|
| NC-series (NCv3) | NVIDIA Tesla V100 | Machine learning training |
| ND-series | NVIDIA A100 | Deep learning, AI |
| NV-series | NVIDIA Tesla M60 | Remote visualization, VDI |
| NCas T4_v3 | NVIDIA T4 | Inference, moderate ML |

```bash
# Example: NC6s_v3 = 6 vCPUs, 112 GB RAM, 1x V100 GPU (16 GB)
# Example: ND96asr_v4 = 96 vCPUs, 900 GB RAM, 8x A100 GPUs
```

**When to use N-series:** Machine learning training, AI inference, video transcoding, 3D rendering, scientific simulation.

#### H-Series — High Performance Computing (HPC)

H-series VMs are purpose-built for compute-intensive HPC workloads with InfiniBand networking for ultra-low latency inter-node communication.

| Size | vCPUs | RAM | Network | Use Case |
|---|---|---|---|---|
| Standard_HB120rs_v3 | 120 | 448 GB | 200 Gb InfiniBand | CFD, molecular dynamics |
| Standard_HC44rs | 44 | 352 GB | 100 Gb InfiniBand | Financial modeling |

**When to use H-series:** MPI-based HPC workloads, computational fluid dynamics (CFD), weather modeling, seismic processing, molecular simulation.

---

### VM Images

An image is the OS template used to create a VM's OS disk.

#### Azure Marketplace Images (Most Common)

```bash
# List all Windows Server images
az vm image list --publisher MicrosoftWindowsServer --offer WindowsServer --sku 2022-Datacenter --all --output table

# List Ubuntu images
az vm image list --publisher Canonical --offer 0001-com-ubuntu-server-jammy --all --output table

# List Red Hat Enterprise Linux
az vm image list --publisher RedHat --offer RHEL --all --output table

# List popular images (cached, fast)
az vm image list --output table
```

#### Common Image URNs

| OS | URN |
|---|---|
| Windows Server 2022 Datacenter | `MicrosoftWindowsServer:WindowsServer:2022-Datacenter:latest` |
| Windows Server 2019 Datacenter | `MicrosoftWindowsServer:WindowsServer:2019-Datacenter:latest` |
| Ubuntu 22.04 LTS | `Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest` |
| Ubuntu 20.04 LTS | `Canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest` |
| RHEL 9.x | `RedHat:RHEL:9_2:latest` |
| CentOS 8.x | `OpenLogic:CentOS:8_5-gen2:latest` |
| Debian 12 | `Debian:debian-12:12:latest` |
| SLES 15 SP5 | `SUSE:sles-15-sp5:gen2:latest` |

#### Custom Images and Shared Image Gallery

```bash
# Capture a VM as a managed image
az vm deallocate --resource-group myRG --name myVM
az vm generalize --resource-group myRG --name myVM
az image create \
  --resource-group myRG \
  --name myCustomImage \
  --source myVM

# Create a Shared Image Gallery
az sig create \
  --resource-group myRG \
  --gallery-name myGallery \
  --location eastus

# Create image definition in gallery
az sig image-definition create \
  --resource-group myRG \
  --gallery-name myGallery \
  --gallery-image-definition myImageDef \
  --publisher myCompany \
  --offer myApp \
  --sku v1 \
  --os-type Linux \
  --os-state generalized

# Create image version from managed image
az sig image-version create \
  --resource-group myRG \
  --gallery-name myGallery \
  --gallery-image-definition myImageDef \
  --gallery-image-version 1.0.0 \
  --managed-image /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Compute/images/myCustomImage \
  --target-regions eastus=1 westus=1
```

---

### Managed Disks

Managed Disks are the recommended persistent storage for Azure VMs. Azure manages the storage account backing — you just pick a disk type and size.

#### Disk Types Comparison

| Disk Type | Max IOPS | Max Throughput | Latency | Use Case | Price/GB |
|---|---|---|---|---|---|
| Ultra Disk | 160,000 | 2,000 MB/s | <1 ms | Mission-critical DBs | High |
| Premium SSD v2 | 80,000 | 1,200 MB/s | ~1 ms | Production DBs | Medium-High |
| Premium SSD | 20,000 | 900 MB/s | ~5 ms | Production workloads | Medium |
| Standard SSD | 6,000 | 750 MB/s | ~10 ms | Web servers, dev | Low-Medium |
| Standard HDD | 2,000 | 500 MB/s | ~20 ms | Backups, cold data | Low |

#### Choosing the Right Disk

```
Decision Tree:
├── Need <1ms latency + high IOPS?
│   └── Ultra Disk (only available in specific regions, zones)
├── Production database (SQL, Oracle)?
│   ├── Need flexible IOPS/throughput sizing?
│   │   └── Premium SSD v2
│   └── Standard production use?
│       └── Premium SSD
├── Web server, development?
│   └── Standard SSD
└── Backups, archive, infrequent access?
    └── Standard HDD
```

#### OS Disk vs Data Disk vs Temp Disk

```
VM Disk Layout:
┌─────────────────────────────────────────────┐
│  OS Disk (Managed Disk)                     │
│  - Contains OS, pagefile                    │
│  - Persists across reboots                  │
│  - Default: 128 GB (Windows), 30 GB (Linux) │
├─────────────────────────────────────────────┤
│  Data Disks (Managed Disks, 0-64 attached)  │
│  - Your application data                    │
│  - Persists across reboots                  │
│  - Max: 32 TB each                          │
├─────────────────────────────────────────────┤
│  Temp Disk (Local SSD, NOT persistent!)     │
│  - D: drive (Windows) or /dev/sdb (Linux)   │
│  - LOST on VM stop/deallocate/resize        │
│  - Use only for swap/scratch                │
└─────────────────────────────────────────────┘
```

**CRITICAL:** Never store important data on the temp disk. It is erased when the VM is deallocated or moved to a different host.

---

### Availability Sets vs Availability Zones vs Scale Sets

#### Availability Sets

An Availability Set ensures your VMs are distributed across multiple **fault domains** (separate physical racks with own power/network) and **update domains** (groups that are updated one at a time during planned maintenance).

```
Availability Set (2 Fault Domains, 5 Update Domains):

Fault Domain 0          Fault Domain 1
(Rack A)                (Rack B)
┌─────────────┐         ┌─────────────┐
│ VM1 (UD0)   │         │ VM2 (UD1)   │
│ VM5 (UD4)   │         │ VM3 (UD2)   │
│             │         │ VM4 (UD3)   │
└─────────────┘         └─────────────┘
SLA: 99.95% uptime
```

**Use Availability Sets when:**
- Deploying to regions without Availability Zones
- Running legacy workloads that require same-region placement
- You need SLA of 99.95%

#### Availability Zones

Availability Zones are physically separate datacenters within a region, each with independent power, cooling, and networking. Distance between zones is typically 1-10 km.

```
Azure Region (e.g., East US 2)
┌─────────────────────────────────────────────────┐
│                                                 │
│  Zone 1            Zone 2            Zone 3     │
│ ┌──────────┐      ┌──────────┐      ┌────────┐ │
│ │Datacenter│      │Datacenter│      │Datacent│ │
│ │    A     │      │    B     │      │   C    │ │
│ │ VM1      │      │ VM2      │      │ VM3    │ │
│ └──────────┘      └──────────┘      └────────┘ │
│                                                 │
│  Connected via high-speed private fiber         │
└─────────────────────────────────────────────────┘
SLA: 99.99% uptime (when deployed across 2+ zones)
```

**Use Availability Zones when:**
- You need highest resiliency (99.99% SLA)
- Protecting against entire datacenter failure
- Running production workloads in zone-enabled regions

#### Comparison: Sets vs Zones

| Feature | Availability Set | Availability Zone |
|---|---|---|
| Protects against | Rack failure, planned maintenance | Datacenter failure |
| SLA | 99.95% | 99.99% |
| Cost | No extra cost for set | Egress charges between zones |
| Region support | All regions | ~30+ regions |
| Disk type | Managed disks | Zone-redundant storage |
| Best for | Single-datacenter HA | Full zone-level resilience |

#### Scale Sets (VMSS)

Scale Sets are covered in [Section 8](#8-virtual-machine-scale-sets-vmss). In short: they deploy multiple identical VMs from a single configuration and auto-scale based on metrics.

---

### VM Networking

Every VM needs at minimum a **Network Interface Card (NIC)** attached to a subnet within a VNet.

```
Internet
    │
    ▼
Public IP Address (Optional)
    │
    ▼
Network Security Group (NSG) - Subnet level
    │
    ▼
Virtual Network (VNet: 10.0.0.0/16)
    │
    ├── Subnet (10.0.1.0/24)
    │       │
    │       ▼
    │   Network Security Group (NSG) - NIC level
    │       │
    │       ▼
    │   NIC (Primary) ──► VM
    │       └── Private IP: 10.0.1.4 (auto or static)
    │           Public IP: 20.55.x.x (optional)
    │
    └── Subnet (10.0.2.0/24) — other VMs/services
```

#### Network Security Group (NSG) Rules

NSGs are stateful packet filters. Rules are evaluated by priority (lower number = higher priority).

```bash
# Create NSG
az network nsg create \
  --resource-group myRG \
  --name myNSG

# Allow SSH (port 22) inbound
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowSSH \
  --priority 100 \
  --protocol Tcp \
  --destination-port-range 22 \
  --access Allow \
  --direction Inbound

# Allow HTTP inbound
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowHTTP \
  --priority 200 \
  --protocol Tcp \
  --destination-port-range 80 \
  --access Allow \
  --direction Inbound

# Allow HTTPS inbound
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowHTTPS \
  --priority 210 \
  --protocol Tcp \
  --destination-port-range 443 \
  --access Allow \
  --direction Inbound

# Deny everything else (default deny is built-in at priority 65500)
```

---

### VM Extensions

VM Extensions are small applications that run post-deployment configuration and automation on your VM.

| Extension | Purpose |
|---|---|
| CustomScriptExtension | Run PowerShell/Bash scripts on deploy |
| Azure Monitor Agent | Send metrics/logs to Azure Monitor |
| Microsoft Antimalware | Install and configure antimalware |
| Key Vault VM Extension | Auto-refresh certificates from Key Vault |
| Azure Disk Encryption | Enable BitLocker/DM-Crypt full disk encryption |
| Diagnostics Extension | Collect boot diagnostics and performance counters |
| Chef/Puppet/DSC | Configuration management |
| Azure AD Login | Enable Azure AD authentication for VM |

```bash
# Install Custom Script Extension on Linux VM (run a bash script)
az vm extension set \
  --resource-group myRG \
  --vm-name myVM \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"fileUris": ["https://mystorageaccount.blob.core.windows.net/scripts/install.sh"], "commandToExecute": "bash install.sh"}'

# Install Azure Monitor Agent
az vm extension set \
  --resource-group myRG \
  --vm-name myVM \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0

# List extensions on a VM
az vm extension list --resource-group myRG --vm-name myVM --output table

# Remove an extension
az vm extension delete \
  --resource-group myRG \
  --vm-name myVM \
  --name CustomScript
```

---

### Azure Spot VMs and Reserved Instances

#### Spot VMs

Spot VMs use Azure's unused capacity at discounts of up to 90% off pay-as-you-go pricing. Azure can evict them with 30 seconds notice when capacity is needed.

```bash
# Create a Spot VM
az vm create \
  --resource-group myRG \
  --name mySpotVM \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \   # or Delete
  --max-price 0.05 \               # max you'll pay per hour (-1 = up to on-demand price)
  --admin-username azureuser \
  --generate-ssh-keys
```

**Eviction policies:**
- `Deallocate`: VM is stopped and deallocated (disks preserved, no compute charge)
- `Delete`: VM and disks are deleted (saves disk storage cost)

**Use Spot VMs for:**
- Batch processing and analytics
- Development and testing
- Stateless microservices
- Machine learning training jobs
- CI/CD build agents

**Do NOT use Spot VMs for:**
- Production web servers
- Databases
- Any workload that cannot tolerate sudden interruption

#### Reserved Instances

Reserved Instances (RIs) give you 1-year or 3-year commitments in exchange for significant discounts.

| Commitment | Discount vs Pay-As-You-Go |
|---|---|
| 1-year Reserved | ~30-40% |
| 3-year Reserved | ~55-65% |
| Spot VM (no commitment) | ~60-90% (but interruptible) |

**Reserved Instance flexibility:**
- **Instance size flexibility:** A reservation for `Standard_D4s_v5` can cover `Standard_D2s_v5` × 2 (same family, different size)
- **Scope:** Single subscription, shared across management group, or resource group
- **Exchange/Refund:** You can exchange or refund RIs (subject to limits)

---

### Complete Azure CLI Commands for VMs

```bash
# ─────────────────────────────────────────────────────────
# PREREQUISITES
# ─────────────────────────────────────────────────────────

# Login to Azure
az login

# Set default subscription
az account set --subscription "My Subscription"

# Create resource group
az group create \
  --name myRG \
  --location eastus

# ─────────────────────────────────────────────────────────
# CREATE A LINUX VM (QUICK)
# ─────────────────────────────────────────────────────────

az vm create \
  --resource-group myRG \
  --name myLinuxVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --output json

# ─────────────────────────────────────────────────────────
# CREATE A WINDOWS VM
# ─────────────────────────────────────────────────────────

az vm create \
  --resource-group myRG \
  --name myWinVM \
  --image Win2022Datacenter \
  --size Standard_D4s_v5 \
  --admin-username azureuser \
  --admin-password "SecureP@ssword123!" \
  --public-ip-sku Standard

# ─────────────────────────────────────────────────────────
# CREATE VM WITH FULL NETWORK CONFIGURATION
# ─────────────────────────────────────────────────────────

# Create VNet and subnet first
az network vnet create \
  --resource-group myRG \
  --name myVNet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name mySubnet \
  --subnet-prefix 10.0.1.0/24

# Create NSG
az network nsg create --resource-group myRG --name myNSG

# Allow SSH
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowSSH \
  --priority 100 \
  --protocol Tcp \
  --destination-port-range 22 \
  --access Allow

# Create public IP (static)
az network public-ip create \
  --resource-group myRG \
  --name myPublicIP \
  --sku Standard \
  --allocation-method Static

# Create NIC
az network nic create \
  --resource-group myRG \
  --name myNIC \
  --vnet-name myVNet \
  --subnet mySubnet \
  --public-ip-address myPublicIP \
  --network-security-group myNSG

# Create VM with custom NIC
az vm create \
  --resource-group myRG \
  --name myVM \
  --nics myNIC \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --os-disk-size-gb 64 \
  --os-disk-caching ReadWrite \
  --storage-sku Premium_LRS

# ─────────────────────────────────────────────────────────
# ADD A DATA DISK
# ─────────────────────────────────────────────────────────

# Attach a new empty managed disk
az vm disk attach \
  --resource-group myRG \
  --vm-name myVM \
  --name myDataDisk \
  --size-gb 128 \
  --sku Premium_LRS \
  --new

# List disks attached to VM
az vm show \
  --resource-group myRG \
  --name myVM \
  --query storageProfile.dataDisks \
  --output table

# Detach a disk
az vm disk detach \
  --resource-group myRG \
  --vm-name myVM \
  --name myDataDisk

# ─────────────────────────────────────────────────────────
# VM LIFECYCLE OPERATIONS
# ─────────────────────────────────────────────────────────

# Start a VM
az vm start --resource-group myRG --name myVM

# Stop (OS shutdown, still allocated — you still pay!)
az vm stop --resource-group myRG --name myVM

# Deallocate (stop billing for compute — disk charges remain)
az vm deallocate --resource-group myRG --name myVM

# Restart
az vm restart --resource-group myRG --name myVM

# Redeploy (move to new Azure host — troubleshoot connectivity issues)
az vm redeploy --resource-group myRG --name myVM

# ─────────────────────────────────────────────────────────
# VM INFORMATION
# ─────────────────────────────────────────────────────────

# List all VMs in a resource group
az vm list --resource-group myRG --output table

# Show VM details
az vm show --resource-group myRG --name myVM --output json

# Show VM status (running, stopped, etc.)
az vm get-instance-view \
  --resource-group myRG \
  --name myVM \
  --query instanceView.statuses[1].displayStatus \
  --output tsv

# Get VM public IP
az vm show \
  --resource-group myRG \
  --name myVM \
  --show-details \
  --query publicIps \
  --output tsv

# ─────────────────────────────────────────────────────────
# RESIZE A VM
# ─────────────────────────────────────────────────────────

# List available sizes in region
az vm list-vm-resize-options \
  --resource-group myRG \
  --name myVM \
  --output table

# Resize (requires VM to be deallocated if changing family)
az vm resize \
  --resource-group myRG \
  --name myVM \
  --size Standard_D4s_v5

# ─────────────────────────────────────────────────────────
# CREATE SNAPSHOT
# ─────────────────────────────────────────────────────────

# Get OS disk ID
DISK_ID=$(az vm show \
  --resource-group myRG \
  --name myVM \
  --query storageProfile.osDisk.managedDisk.id \
  --output tsv)

# Create snapshot
az snapshot create \
  --resource-group myRG \
  --name mySnapshot \
  --source "$DISK_ID" \
  --incremental true

# ─────────────────────────────────────────────────────────
# DELETE A VM
# ─────────────────────────────────────────────────────────

# Delete VM only (disk and NIC remain)
az vm delete --resource-group myRG --name myVM --yes

# Delete VM and all associated resources
az group delete --name myRG --yes --no-wait
```

---

### Bicep Template Example for VM

```bicep
// vm.bicep — Deploys a Linux VM with VNet, NSG, and Public IP

param location string = resourceGroup().location
param vmName string = 'myLinuxVM'
param adminUsername string = 'azureuser'
param vmSize string = 'Standard_D2s_v5'

@secure()
param adminPublicKey string

var vnetName = '${vmName}-vnet'
var subnetName = 'default'
var nsgName = '${vmName}-nsg'
var nicName = '${vmName}-nic'
var publicIpName = '${vmName}-pip'
var osDiskName = '${vmName}-osdisk'

// Public IP
resource publicIp 'Microsoft.Network/publicIPAddresses@2023-04-01' = {
  name: publicIpName
  location: location
  sku: { name: 'Standard' }
  properties: {
    publicIPAllocationMethod: 'Static'
    dnsSettings: {
      domainNameLabel: toLower(vmName)
    }
  }
}

// Network Security Group
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-04-01' = {
  name: nsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowSSH'
        properties: {
          priority: 100
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '22'
        }
      }
    ]
  }
}

// VNet
resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
          networkSecurityGroup: { id: nsg.id }
        }
      }
    ]
  }
}

// NIC
resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = {
  name: nicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: { id: vnet.properties.subnets[0].id }
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: { id: publicIp.id }
        }
      }
    ]
  }
}

// Virtual Machine
resource vm 'Microsoft.Compute/virtualMachines@2023-07-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: { vmSize: vmSize }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      linuxConfiguration: {
        disablePasswordAuthentication: true
        ssh: {
          publicKeys: [
            {
              path: '/home/${adminUsername}/.ssh/authorized_keys'
              keyData: adminPublicKey
            }
          ]
        }
      }
    }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer: '0001-com-ubuntu-server-jammy'
        sku: '22_04-lts-gen2'
        version: 'latest'
      }
      osDisk: {
        name: osDiskName
        createOption: 'FromImage'
        managedDisk: { storageAccountType: 'Premium_LRS' }
        diskSizeGB: 64
      }
    }
    networkProfile: {
      networkInterfaces: [{ id: nic.id }]
    }
    diagnosticsProfile: {
      bootDiagnostics: { enabled: true }
    }
  }
}

output vmPublicIp string = publicIp.properties.ipAddress
output vmFqdn string = publicIp.properties.dnsSettings.fqdn
```

```bash
# Deploy the Bicep template
az deployment group create \
  --resource-group myRG \
  --template-file vm.bicep \
  --parameters adminPublicKey="$(cat ~/.ssh/id_rsa.pub)" vmName=myLinuxVM
```

---

### VM Cost Optimization Tips

1. **Right-size your VMs.** Use Azure Advisor recommendations. A 50% average CPU VM should be downsized.
2. **Use B-series for dev/test** — they're 30-50% cheaper than D-series for bursty workloads.
3. **Deallocate (not just stop) VMs** outside business hours. Stopped VMs still incur compute charges.
4. **Use Spot VMs** for stateless, fault-tolerant workloads (batch, ML training, CI/CD).
5. **Buy Reserved Instances** for stable production workloads — up to 65% savings over 3 years.
6. **Use Azure Hybrid Benefit** if you have Windows Server or SQL Server on-premises licenses.
7. **Auto-shutdown** dev VMs at 6pm daily using Azure DevTest Labs or scheduled shutdown feature.
8. **Use Managed Disks** (not unmanaged) — they're simpler and cost-optimized.
9. **Delete orphaned resources**: Disks, NICs, and public IPs continue billing after VM deletion.
10. **Enable Accelerated Networking** on D-series and later — same cost, up to 30 Gbps throughput.

```bash
# Enable auto-shutdown on a VM (shut down at 7pm UTC daily)
az vm auto-shutdown \
  --resource-group myRG \
  --name myVM \
  --time 1900 \
  --email "you@example.com"

# Enable Azure Hybrid Benefit for Windows
az vm update \
  --resource-group myRG \
  --name myWinVM \
  --license-type Windows_Server

# Enable Accelerated Networking on NIC
az network nic update \
  --resource-group myRG \
  --name myNIC \
  --accelerated-networking true
```

### VM Best Practices

| Area | Best Practice |
|---|---|
| **Security** | Disable password auth on Linux; use SSH keys |
| **Security** | Never expose RDP/SSH directly to internet; use Azure Bastion |
| **Security** | Apply NSGs at subnet level (not just NIC level) |
| **Security** | Enable Microsoft Defender for Cloud on all VMs |
| **Storage** | Always use Managed Disks (not storage account-backed) |
| **Storage** | Enable disk encryption (Azure Disk Encryption or SSE with CMK) |
| **Networking** | Assign static private IP for long-lived VMs |
| **Availability** | Place production VMs in Availability Zones |
| **Availability** | Use multiple VMs behind a load balancer |
| **Monitoring** | Install Azure Monitor Agent; send to Log Analytics |
| **Patching** | Enable Azure Update Manager for automated patching |
| **Backup** | Enable Azure Backup for VM protection |
| **Cost** | Use Auto-shutdown for non-production VMs |

---

## 2. Azure App Service

### What is App Service?

Azure App Service is a fully managed PaaS platform for hosting web applications, REST APIs, and mobile back-ends. You bring your code; Azure manages the underlying OS, runtime patching, scaling infrastructure, and load balancing.

**Use App Service when:**
- Deploying a web application or REST API
- You want managed hosting without managing VMs
- You need deployment slots (blue/green, staging)
- You want CI/CD integration with GitHub, Azure DevOps, Bitbucket
- You need auto-scale based on HTTP requests

**Do NOT use App Service when:**
- You need non-HTTP protocols (use VMs or AKS)
- Your app requires specific OS-level customization
- You need GPU compute
- You're running containerized microservices at scale (use AKS or Container Apps)

```
App Service Architecture:
┌─────────────────────────────────────────────────────────┐
│                  App Service Plan                       │
│  (Defines compute: vCPUs, RAM, instances, features)    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │             App Service (Web App)                │  │
│  │                                                  │  │
│  │  ┌────────────┐  ┌────────────┐                 │  │
│  │  │ Production │  │  Staging   │ ← Deployment     │  │
│  │  │   Slot     │  │   Slot     │   Slots          │  │
│  │  └────────────┘  └────────────┘                 │  │
│  │                                                  │  │
│  │  Runtime: .NET, Node, Python, Java, PHP, Ruby   │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Scale out: 1 → N instances (auto or manual)           │
└─────────────────────────────────────────────────────────┘
```

---

### App Service Plans and Tiers

An **App Service Plan** defines the compute resources that your app runs on. Multiple apps can share the same plan (saving cost), but they all share the same compute resources.

| Tier | Plan | vCPUs | RAM | Instances | Custom Domain | SSL | Slots | Auto-Scale |
|---|---|---|---|---|---|---|---|---|
| **Free** | F1 | Shared | 1 GB | 1 | No | No | 0 | No |
| **Shared** | D1 | Shared | 1 GB | 1 | Yes | No | 0 | No |
| **Basic** | B1 | 1 | 1.75 GB | 1-3 | Yes | Yes | 0 | No |
| **Basic** | B2 | 2 | 3.5 GB | 1-3 | Yes | Yes | 0 | No |
| **Basic** | B3 | 4 | 7 GB | 1-3 | Yes | Yes | 0 | No |
| **Standard** | S1 | 1 | 1.75 GB | 1-10 | Yes | Yes | 5 | Yes |
| **Standard** | S2 | 2 | 3.5 GB | 1-10 | Yes | Yes | 5 | Yes |
| **Standard** | S3 | 4 | 7 GB | 1-10 | Yes | Yes | 5 | Yes |
| **Premium v3** | P1v3 | 2 | 8 GB | 1-30 | Yes | Yes | 20 | Yes |
| **Premium v3** | P2v3 | 4 | 16 GB | 1-30 | Yes | Yes | 20 | Yes |
| **Premium v3** | P3v3 | 8 | 32 GB | 1-30 | Yes | Yes | 20 | Yes |
| **Isolated v2** | I1v2 | 2 | 8 GB | 1-100 | Yes | Yes | 20 | Yes |

**Free/Shared:** Runs on shared infra, no SLA, limited CPU minutes. Good for testing only.  
**Basic:** Dedicated VMs, manual scale only, no deployment slots.  
**Standard:** Production-ready, auto-scale, deployment slots, custom domains with SNI SSL.  
**Premium v3:** Enhanced performance, VNet Integration, Private Endpoints, more slots.  
**Isolated v2:** Runs in your own App Service Environment (ASE) within your VNet. Maximum isolation, compliance.

**Pricing guidance:**
- Free → $0 (60 CPU min/day limit)
- B1 → ~$13/month
- S1 → ~$70/month
- P1v3 → ~$125/month
- I1v2 → ~$400/month

---

### Deployment Slots

Deployment slots are live environments with their own URLs (e.g., `myapp-staging.azurewebsites.net`). You deploy to staging first, validate, then **swap** to production — zero downtime.

```
Deployment Flow:
                     ┌──────────────────────┐
Developer → Deploy → │  Staging Slot        │
                     │  myapp-staging.azurewebsites.net │
                     │  v2.0 code           │
                     └──────────┬───────────┘
                                │  Swap (zero downtime)
                     ┌──────────▼───────────┐
                     │  Production Slot     │
                     │  myapp.azurewebsites.net        │
                     │  v2.0 code (swapped) │
                     └──────────────────────┘

Swap behavior:
- Production gets staging's code
- Staging gets production's code (easy rollback!)
- "Sticky settings" (marked slot-specific) do NOT swap
- Warm-up requests are sent before traffic moves
```

```bash
# Create a deployment slot
az webapp deployment slot create \
  --resource-group myRG \
  --name myApp \
  --slot staging

# Deploy to staging slot
az webapp deployment source config-zip \
  --resource-group myRG \
  --name myApp \
  --slot staging \
  --src ./app.zip

# Swap staging to production
az webapp deployment slot swap \
  --resource-group myRG \
  --name myApp \
  --slot staging \
  --target-slot production

# List slots
az webapp deployment slot list \
  --resource-group myRG \
  --name myApp \
  --output table

# Delete a slot
az webapp deployment slot delete \
  --resource-group myRG \
  --name myApp \
  --slot staging
```

---

### Complete CLI Commands for App Service

```bash
# ─────────────────────────────────────────────────────────
# CREATE APP SERVICE PLAN AND WEB APP
# ─────────────────────────────────────────────────────────

# Create App Service Plan (Linux, Standard S1)
az appservice plan create \
  --resource-group myRG \
  --name myPlan \
  --location eastus \
  --sku S1 \
  --is-linux

# Create Web App (.NET 8)
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myUniqueAppName \
  --runtime "DOTNETCORE:8.0"

# Create Web App (Node.js 20)
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myNodeApp \
  --runtime "NODE:20-lts"

# Create Web App (Python 3.11)
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myPythonApp \
  --runtime "PYTHON:3.11"

# Create Web App from Docker container
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myContainerApp \
  --deployment-container-image-name nginx:latest

# ─────────────────────────────────────────────────────────
# CONFIGURE APP SETTINGS
# ─────────────────────────────────────────────────────────

# Set application settings (environment variables)
az webapp config appsettings set \
  --resource-group myRG \
  --name myApp \
  --settings \
    DATABASE_URL="Server=myserver.database.windows.net;..." \
    REDIS_HOST="myredis.redis.cache.windows.net" \
    APP_ENV="production"

# List app settings
az webapp config appsettings list \
  --resource-group myRG \
  --name myApp \
  --output table

# Delete an app setting
az webapp config appsettings delete \
  --resource-group myRG \
  --name myApp \
  --setting-names DATABASE_URL

# Set connection strings
az webapp config connection-string set \
  --resource-group myRG \
  --name myApp \
  --name SQLConnectionString \
  --type SQLAzure \
  --connection-string "Server=tcp:..."

# ─────────────────────────────────────────────────────────
# DEPLOYMENT
# ─────────────────────────────────────────────────────────

# Deploy from ZIP file
az webapp deploy \
  --resource-group myRG \
  --name myApp \
  --src-path ./publish.zip \
  --type zip

# Configure Git deployment
az webapp deployment source config \
  --resource-group myRG \
  --name myApp \
  --repo-url https://github.com/myorg/myrepo \
  --branch main \
  --manual-integration

# Enable local Git deployment
az webapp deployment source config-local-git \
  --resource-group myRG \
  --name myApp

# Get deployment credentials
az webapp deployment list-publishing-credentials \
  --resource-group myRG \
  --name myApp \
  --output json

# ─────────────────────────────────────────────────────────
# SCALING
# ─────────────────────────────────────────────────────────

# Scale up (change plan SKU)
az appservice plan update \
  --resource-group myRG \
  --name myPlan \
  --sku P2v3

# Scale out manually (add instances)
az appservice plan update \
  --resource-group myRG \
  --name myPlan \
  --number-of-workers 3

# Configure autoscale
az monitor autoscale create \
  --resource-group myRG \
  --name myAutoscale \
  --resource "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/serverfarms/myPlan" \
  --min-count 1 \
  --max-count 10 \
  --count 2

# Add autoscale rule (scale out when CPU > 70%)
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 2

# Add autoscale rule (scale in when CPU < 30%)
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1

# ─────────────────────────────────────────────────────────
# CUSTOM DOMAIN AND SSL
# ─────────────────────────────────────────────────────────

# Add custom domain
az webapp config hostname add \
  --resource-group myRG \
  --webapp-name myApp \
  --hostname www.mysite.com

# Create managed certificate (free, auto-renew)
az webapp config ssl create \
  --resource-group myRG \
  --name myApp \
  --hostname www.mysite.com

# Bind certificate to hostname
az webapp config ssl bind \
  --resource-group myRG \
  --name myApp \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI

# ─────────────────────────────────────────────────────────
# MONITORING AND LOGS
# ─────────────────────────────────────────────────────────

# Enable application logging
az webapp log config \
  --resource-group myRG \
  --name myApp \
  --application-logging filesystem \
  --level information \
  --web-server-logging filesystem

# Stream live logs
az webapp log tail \
  --resource-group myRG \
  --name myApp

# Download log files
az webapp log download \
  --resource-group myRG \
  --name myApp \
  --log-file logs.zip

# Show recent logs
az webapp log show \
  --resource-group myRG \
  --name myApp

# ─────────────────────────────────────────────────────────
# VNet Integration
# ─────────────────────────────────────────────────────────

# Enable regional VNet integration
az webapp vnet-integration add \
  --resource-group myRG \
  --name myApp \
  --vnet myVNet \
  --subnet mySubnet

# List VNet integrations
az webapp vnet-integration list \
  --resource-group myRG \
  --name myApp
```

---

### App Service Environment (ASE)

App Service Environment (ASE) is a premium, fully isolated deployment of App Service that runs within your own VNet. It supports the Isolated pricing tier.

```
ASE Architecture:
┌──────────────────────────────────────────────────────┐
│  Your Azure VNet (e.g., 10.0.0.0/16)                │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  ASE Subnet (needs /24 or larger)              │ │
│  │                                                │ │
│  │  ┌──────────────────────────────────────────┐ │ │
│  │  │  App Service Environment v3              │ │ │
│  │  │  - Internal Load Balancer (ILB ASE)      │ │ │
│  │  │    or External (public VIP)              │ │ │
│  │  │                                          │ │ │
│  │  │  Apps: myapp.myase.appserviceenvironment │ │ │
│  │  │  Plan: Isolated v2 (I1v2, I2v2, I3v2)   │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**Use ASE when:**
- Apps must be fully network-isolated (compliance, PCI-DSS, HIPAA)
- Need to access private resources (on-prem via ExpressRoute, private endpoints)
- Need very high scale (100+ instances on single plan)
- Need dedicated outbound IP addresses

### App Service Best Practices

| Area | Best Practice |
|---|---|
| **Performance** | Use Premium v3 plans for production (optimized hardware) |
| **Scaling** | Configure auto-scale rules; don't rely on manual scaling |
| **Deployment** | Always use deployment slots; never deploy directly to production |
| **Security** | Use Managed Identity instead of connection strings with passwords |
| **Security** | Enable Always-On to prevent app pool recycling |
| **Security** | Disable FTP; use ZIP deploy or CI/CD pipelines |
| **Config** | Store secrets in Key Vault; reference with Key Vault references |
| **Monitoring** | Enable Application Insights for performance monitoring |
| **Network** | Use VNet Integration for outbound traffic to private resources |
| **Network** | Use Private Endpoint to restrict inbound traffic |
| **Cost** | Share App Service Plans across dev/test apps |

---

## 3. Azure Functions (Serverless)

### What are Azure Functions?

Azure Functions is an event-driven, serverless compute platform. You write a function (a small unit of code), configure a trigger (what starts it), and Azure handles all infrastructure: servers, OS, runtime, scaling, and billing.

You pay only for what you use — billed per execution and execution time, not for idle capacity.

**Use Azure Functions when:**
- Processing events (queue messages, blob uploads, HTTP requests)
- Scheduled tasks (run every 5 minutes)
- Building APIs without managing servers
- Fan-out processing (process 1000 items in parallel)
- Glue code (connecting services together)
- Real-time data stream processing

```
Azure Functions Event Flow:

Event Source              Functions Runtime        Your Code
(Trigger)                                          (Function)
┌─────────────┐           ┌──────────────┐        ┌──────────────┐
│ HTTP Request │──────────►│              │        │  function.py │
│ Queue Msg    │──────────►│  Trigger     │───────►│  function.cs │
│ Blob Upload  │──────────►│  System      │        │  index.js    │
│ Timer (cron) │──────────►│              │◄───────│              │
│ Event Hub    │──────────►│  Auto-scale  │        │  Output      │
│ Service Bus  │──────────►│  0→N         │───────►│  Bindings    │
└─────────────┘           └──────────────┘        └──────────────┘
                                                         │
                                                   Output Targets:
                                                   - Storage Blob
                                                   - Queue
                                                   - CosmosDB
                                                   - Service Bus
                                                   - SignalR
```

---

### Hosting Plans

| Plan | Pricing | Scale | Cold Start | Max Execution | Best For |
|---|---|---|---|---|---|
| **Consumption** | Per execution + GB-s | 0 to 200 instances | Yes (~1-3s) | 10 minutes | Infrequent, bursty |
| **Flex Consumption** | Per execution (newer) | 0 to 1000 instances | Reduced | Configurable | High-scale bursty |
| **Premium** | Fixed + per execution | Pre-warmed instances | No | Unlimited | Always-on, VNet |
| **Dedicated (App Service)** | Per App Service Plan | Manual/auto-scale | No | Unlimited | Predictable load |
| **Container Apps** | Per vCPU/memory | KEDA-based | Minimal | Unlimited | Full containers |

#### Consumption Plan Details
- **Bill:** First 1 million executions free, then $0.20/million. First 400,000 GB-s free.
- **Timeout:** Default 5 min, max 10 min.
- **Scale:** Up to 200 instances per region (increases on request).
- **Cold Start:** First invocation after idle period takes 1-5 seconds to spin up.

#### Premium Plan Details
- **Bill:** At least 1 pre-warmed instance always running (hourly charge) + burst instances.
- **Benefits:** No cold start, VNet Integration, longer timeout, BYOC (custom containers).
- **Timeout:** Unlimited.
- **Use when:** You need always-on, VNet access, or long-running functions.

#### Dedicated Plan Details
- **Bill:** App Service Plan charges regardless of executions.
- **Use when:** You already have an under-utilized App Service Plan and want to add functions.

---

### Triggers Deep Dive

#### HTTP Trigger

```python
# Python HTTP Trigger
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.route(route="hello/{name}", methods=["GET", "POST"])
def http_trigger(req: func.HttpRequest) -> func.HttpResponse:
    name = req.route_params.get('name', 'World')
    
    # Read query param
    greeting = req.params.get('greeting', 'Hello')
    
    logging.info(f'Processing request for {name}')
    
    return func.HttpResponse(
        f"{greeting}, {name}!",
        status_code=200,
        headers={"Content-Type": "text/plain"}
    )
```

```csharp
// C# HTTP Trigger (.NET 8 isolated worker)
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;

public class HttpTriggerFunction
{
    private readonly ILogger<HttpTriggerFunction> _logger;

    public HttpTriggerFunction(ILogger<HttpTriggerFunction> logger)
    {
        _logger = logger;
    }

    [Function("HttpTrigger")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = "hello/{name}")] 
        HttpRequestData req,
        string name)
    {
        _logger.LogInformation($"Processing request for {name}");

        var response = req.CreateResponse(HttpStatusCode.OK);
        response.Headers.Add("Content-Type", "text/plain");
        await response.WriteStringAsync($"Hello, {name}!");
        return response;
    }
}
```

#### Timer Trigger (Scheduled)

```javascript
// JavaScript Timer Trigger — runs every 5 minutes
const { app } = require('@azure/functions');

app.timer('scheduledTask', {
    schedule: '0 */5 * * * *',   // CRON: every 5 minutes
    handler: async (myTimer, context) => {
        const timeStamp = new Date().toISOString();
        context.log(`Timer fired at: ${timeStamp}`);
        
        if (myTimer.isPastDue) {
            context.log('Timer function is running late!');
        }
        
        // Your work here
        await processRecords();
    }
});

// Common CRON expressions:
// '0 0 * * * *'      — every hour at :00
// '0 0 9 * * 1-5'    — weekdays at 9am
// '0 */30 * * * *'   — every 30 minutes
// '0 0 0 * * *'      — daily at midnight
// '0 0 0 1 * *'      — first day of every month
```

#### Blob Storage Trigger

```python
# Python Blob Trigger — fires when new blob is created in container
import azure.functions as func
import logging

app = func.FunctionApp()

@app.blob_trigger(
    arg_name="myblob",
    path="uploads/{name}",    # 'uploads' container, {name} = filename
    connection="AzureWebJobsStorage"
)
def blob_trigger(myblob: func.InputStream):
    logging.info(f"Blob trigger: {myblob.name}, size: {myblob.length} bytes")
    
    # Read content
    content = myblob.read()
    logging.info(f"First 100 bytes: {content[:100]}")
    
    # Process the blob (e.g., resize image, parse CSV, etc.)
    process_file(content, myblob.name)
```

#### Queue Storage Trigger

```python
# Python Queue Trigger — processes messages from Azure Queue Storage
import azure.functions as func
import json
import logging

app = func.FunctionApp()

@app.queue_trigger(
    arg_name="msg",
    queue_name="my-queue",
    connection="AzureWebJobsStorage"
)
def queue_trigger(msg: func.QueueMessage):
    message_body = msg.get_json()
    logging.info(f"Processing message: {message_body}")
    
    # Dequeue count — how many times this has been dequeued
    dequeue_count = msg.dequeue_count
    
    if dequeue_count > 5:
        logging.error(f"Message failed 5 times, moving to poison queue")
        # Poison messages automatically go to {queue-name}-poison queue
        return
    
    process_order(message_body['orderId'])
```

#### Event Hub Trigger (Batch Processing)

```csharp
// C# Event Hub Trigger — processes batches of events
[Function("EventHubTrigger")]
public void Run(
    [EventHubTrigger("my-hub", Connection = "EventHubConnectionString", 
                     ConsumerGroup = "$Default")] 
    string[] events,
    FunctionContext context)
{
    var logger = context.GetLogger("EventHubTrigger");
    logger.LogInformation($"Processing batch of {events.Length} events");
    
    foreach (var eventData in events)
    {
        var data = JsonSerializer.Deserialize<MyEvent>(eventData);
        logger.LogInformation($"Event: {data.Id} at {data.Timestamp}");
        ProcessEvent(data);
    }
}
```

---

### Input and Output Bindings

Bindings let your function read from or write to external services **without writing connection code**. Azure handles authentication and connection management.

```
Function with Bindings:

Input Binding            Your Function            Output Binding
(reads data)                                      (writes data)
┌──────────────┐         ┌────────────────┐       ┌──────────────┐
│ CosmosDB     │────────►│                │──────►│ Queue Storage│
│ (read record)│         │  function code │       │ (send msg)   │
└──────────────┘         │                │       └──────────────┘
                         │                │       ┌──────────────┐
HTTP Trigger ───────────►│                │──────►│ Blob Storage │
                         │                │       │ (write file) │
                         └────────────────┘       └──────────────┘
```

```python
# Python function with CosmosDB input binding + Queue output binding
import azure.functions as func
import logging

app = func.FunctionApp()

@app.route(route="order/{orderId}", methods=["GET"])
@app.cosmos_db_input(
    arg_name="order",
    database_name="mydb",
    container_name="orders",
    connection="CosmosDBConnection",
    id="{orderId}",
    partition_key="{orderId}"
)
@app.queue_output(
    arg_name="outputQueue",
    queue_name="processed-orders",
    connection="AzureWebJobsStorage"
)
def get_and_queue_order(
    req: func.HttpRequest,
    order: func.DocumentList,
    outputQueue: func.Out[str]
) -> func.HttpResponse:
    
    if not order:
        return func.HttpResponse("Order not found", status_code=404)
    
    order_data = order[0]
    
    # Queue the order for processing
    outputQueue.set(str(order_data))
    
    return func.HttpResponse(f"Order {order_data['id']} queued for processing")
```

---

### Durable Functions

Durable Functions is an extension for stateful, long-running function workflows. It adds orchestrators, activities, and entities.

```
Durable Functions Patterns:

1. FUNCTION CHAINING (sequential steps)
   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
   │Step1│───►│Step2│───►│Step3│───►│Step4│
   └─────┘    └─────┘    └─────┘    └─────┘
   Output of each step feeds next step

2. FAN-OUT / FAN-IN (parallel work)
   ┌────────────────────┐
   │    Orchestrator    │
   └───┬────┬────┬──────┘
       │    │    │       Parallel execution
   ┌───▼┐ ┌▼──┐ ┌▼──┐
   │Act1│ │Act2│ │Act3│
   └───┬┘ └┬──┘ └┬──┘
       │    │    │
   ┌───▼────▼────▼──────┐
   │   Aggregate Result │
   └────────────────────┘

3. ASYNC HTTP API (long polling)
   Client ──POST──► HTTP Start ──► Orchestrator
   Client ◄─202─── (returns status URL)
   Client ──GET──► Status URL ──► (running/completed)
   Client ◄─200─── (result when done)

4. MONITOR (recurring check with dynamic intervals)
   Start ──► Check Status ──► Done? ──Yes──► End
                  ▲                 │
                  └──── Wait ───────No
```

```python
# Python Durable Functions — Fan-out/Fan-in pattern
import azure.durable_functions as df
import azure.functions as func

app = df.DFApp(http_auth_level=func.AuthLevel.ANONYMOUS)

# HTTP starter — creates a new orchestration instance
@app.route(route="orchestrators/{functionName}")
@app.durable_client_input(client_name="client")
async def http_start(req: func.HttpRequest, client):
    instance_id = await client.start_new(
        req.route_params["functionName"],
        client_input=req.get_json()
    )
    return client.create_check_status_response(req, instance_id)

# Orchestrator — coordinates the workflow
@app.orchestration_trigger(context_name="context")
def process_items_orchestrator(context: df.DurableOrchestrationContext):
    # Get input
    items = context.get_input()
    
    # Fan-out: process all items in parallel
    tasks = [context.call_activity("process_single_item", item) for item in items]
    
    # Fan-in: wait for all to complete
    results = yield context.task_all(tasks)
    
    # Aggregate results
    total = sum(results)
    return {"processed": len(results), "total": total}

# Activity — does the actual work
@app.activity_trigger(input_name="item")
def process_single_item(item: dict) -> int:
    # Process the item (call API, write to DB, etc.)
    value = item.get("value", 0)
    return value * 2
```

---

### Cold Start Problem and Mitigation

**Cold Start** occurs when a function app has been idle and Azure needs to spin up a new instance from scratch. During cold start, the runtime loads: OS → Function runtime → Your dependencies → Your code.

```
Cold Start Timeline (Consumption Plan, Python):
0ms ─────────────────────────────────────────► 3000ms
│                                                 │
│  OS Init  Runtime Init  Deps Load  Code Load    │
│  ~200ms   ~300ms        ~1000ms    ~500ms        │
│                                              Ready│
```

**Mitigation strategies:**

| Strategy | How | Plan Required |
|---|---|---|
| **Premium Plan** | Pre-warmed instances | Premium |
| **Always Ready** | Minimum instances (Flex Consumption) | Flex Consumption |
| **Keep-alive ping** | Timer function pings HTTP trigger every 5 min | Any |
| **Reduce dependencies** | Minimize imports, use lighter packages | Any |
| **Async startup** | Lazy-load heavy dependencies | Any |
| **.NET ahead-of-time** | Use .NET AOT compilation | .NET only |

```bash
# Set minimum instance count on Premium plan (eliminates cold start)
az functionapp config appsettings set \
  --resource-group myRG \
  --name myFuncApp \
  --settings WEBSITE_RUN_FROM_PACKAGE=1 \
             FUNCTIONS_WORKER_RUNTIME=python \
             WEBSITE_ENABLE_SYNC_UPDATE_SITE=true

# For Premium plan — set pre-warmed instance count
az resource update \
  --resource-group myRG \
  --name myFuncApp \
  --resource-type "Microsoft.Web/sites" \
  --set properties.siteConfig.preWarmedInstanceCount=2
```

---

### Complete CLI Commands for Functions

```bash
# ─────────────────────────────────────────────────────────
# CREATE FUNCTION APP
# ─────────────────────────────────────────────────────────

# Create storage account (required for Functions)
az storage account create \
  --name myfuncsa$(date +%s) \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

# Create Function App (Consumption plan, Python 3.11)
az functionapp create \
  --resource-group myRG \
  --name myFuncApp \
  --storage-account myfuncsa \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux

# Create Function App (Consumption plan, .NET 8)
az functionapp create \
  --resource-group myRG \
  --name myDotNetFuncApp \
  --storage-account myfuncsa \
  --consumption-plan-location eastus \
  --runtime dotnet-isolated \
  --runtime-version 8 \
  --functions-version 4

# Create Function App (Premium plan, Node.js)
az functionapp plan create \
  --resource-group myRG \
  --name myPremiumPlan \
  --location eastus \
  --sku EP1 \           # EP1, EP2, EP3
  --is-linux

az functionapp create \
  --resource-group myRG \
  --name myNodeFuncApp \
  --plan myPremiumPlan \
  --storage-account myfuncsa \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4

# ─────────────────────────────────────────────────────────
# DEPLOY FUNCTION CODE
# ─────────────────────────────────────────────────────────

# Deploy via ZIP (most common for CI/CD)
func azure functionapp publish myFuncApp --build remote

# Deploy ZIP directly via CLI
az functionapp deploy \
  --resource-group myRG \
  --name myFuncApp \
  --src-path ./function.zip \
  --type zip

# ─────────────────────────────────────────────────────────
# CONFIGURE AND MANAGE
# ─────────────────────────────────────────────────────────

# Set app settings
az functionapp config appsettings set \
  --resource-group myRG \
  --name myFuncApp \
  --settings \
    MY_CONNECTION_STRING="Server=..." \
    FEATURE_FLAG_ENABLED=true

# List app settings
az functionapp config appsettings list \
  --resource-group myRG \
  --name myFuncApp

# Enable application insights
az monitor app-insights component create \
  --app myFuncAppInsights \
  --location eastus \
  --resource-group myRG \
  --kind web

INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app myFuncAppInsights \
  --resource-group myRG \
  --query instrumentationKey --output tsv)

az functionapp config appsettings set \
  --resource-group myRG \
  --name myFuncApp \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY=$INSTRUMENTATION_KEY

# Restart function app
az functionapp restart --resource-group myRG --name myFuncApp

# List functions in the app
az functionapp function list \
  --resource-group myRG \
  --name myFuncApp \
  --output table

# Get function URL
az functionapp function show \
  --resource-group myRG \
  --name myFuncApp \
  --function-name HttpTrigger \
  --query invokeUrlTemplate

# Stream live logs
az webapp log tail \
  --resource-group myRG \
  --name myFuncApp
```

### Functions Best Practices

| Area | Best Practice |
|---|---|
| **Design** | Keep functions small and single-purpose |
| **Design** | Use output bindings for writing to external services |
| **Reliability** | Make functions idempotent — safe to run multiple times |
| **Error Handling** | Always handle exceptions; failed queue messages go to poison queue |
| **Security** | Use Managed Identity for all service connections |
| **Security** | Use Function-level auth keys minimum; prefer AAD for HTTP |
| **Performance** | Initialize expensive resources outside the function handler |
| **Monitoring** | Always connect Application Insights |
| **Durable** | Use Durable Functions for workflows >10 min or complex state |
| **Cold Start** | Use Premium plan if latency is critical |

---

## 4. Azure Container Instances (ACI)

### What is ACI?

Azure Container Instances (ACI) is the simplest way to run a container in Azure. You specify a container image, CPU, memory, and network settings — Azure runs the container within seconds without any cluster management.

**ACI vs AKS vs Container Apps:**

| Feature | ACI | AKS | Container Apps |
|---|---|---|---|
| Startup time | Seconds | Minutes (cluster) | Seconds |
| Orchestration | None | Full Kubernetes | Serverless (KEDA) |
| Scaling | Manual | Full auto-scale | Auto-scale (KEDA) |
| Microservices | Poor | Excellent | Excellent |
| Pricing | Per second | Per node | Per vCPU/s |
| Management | Zero | High | Low |
| Persistent storage | Azure Files | Any PV | Azure Files |
| Best for | One-off tasks | Complex workloads | Microservices |

**Use ACI when:**
- Running batch jobs or data processing tasks
- Dev/test containers without a full cluster
- Event-driven container jobs (triggered by queue, timer, etc.)
- Running CI/CD build jobs
- Isolated containers that need to run once and exit
- Burstable workloads via Virtual Nodes (AKS + ACI integration)

---

### Container Groups

A **Container Group** in ACI is similar to a Kubernetes Pod — multiple containers that share a lifecycle, networking, and storage, and are scheduled on the same host.

```
Container Group: my-job
┌──────────────────────────────────────────────┐
│  Container Group (single host)               │
│  Public IP: 52.x.x.x                         │
│                                              │
│  ┌────────────────────┐  ┌────────────────┐  │
│  │  app-container     │  │  sidecar       │  │
│  │  nginx:latest      │  │  fluentd:latest│  │
│  │  Port 80           │  │  (log shipper) │  │
│  │  CPU: 1, RAM: 1GB  │  │  CPU: 0.5      │  │
│  └────────────────────┘  └────────────────┘  │
│                                              │
│  Shared volume: /app/data (Azure Files)      │
└──────────────────────────────────────────────┘
```

Multiple containers in a group:
- Share the same IP address and port namespace
- Can communicate via `localhost`
- Share volumes
- Start and stop together

---

### Complete CLI Commands for ACI

```bash
# ─────────────────────────────────────────────────────────
# CREATE AND RUN A CONTAINER
# ─────────────────────────────────────────────────────────

# Run a simple container (public image)
az container create \
  --resource-group myRG \
  --name myContainer \
  --image nginx:latest \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 \
  --protocol TCP \
  --dns-name-label my-nginx-demo \
  --location eastus

# Run with environment variables
az container create \
  --resource-group myRG \
  --name myPythonJob \
  --image python:3.11-slim \
  --cpu 2 \
  --memory 4 \
  --environment-variables \
    BATCH_SIZE=100 \
    OUTPUT_PATH=/data/output \
  --secure-environment-variables \
    DB_PASSWORD=mysecretpassword \
  --command-line "python /app/process.py" \
  --restart-policy Never    # OnFailure, Always, Never

# Run from private Azure Container Registry
az container create \
  --resource-group myRG \
  --name myPrivateContainer \
  --image myregistry.azurecr.io/myapp:v1.0 \
  --cpu 1 \
  --memory 2 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username $(az acr credential show --name myregistry --query username --output tsv) \
  --registry-password $(az acr credential show --name myregistry --query passwords[0].value --output tsv)

# ─────────────────────────────────────────────────────────
# MANAGE CONTAINERS
# ─────────────────────────────────────────────────────────

# Show container status
az container show \
  --resource-group myRG \
  --name myContainer \
  --output json

# Get logs
az container logs \
  --resource-group myRG \
  --name myContainer

# Stream live logs
az container attach \
  --resource-group myRG \
  --name myContainer

# Execute a command inside a running container
az container exec \
  --resource-group myRG \
  --name myContainer \
  --exec-command "/bin/sh"

# Stop container
az container stop \
  --resource-group myRG \
  --name myContainer

# Start container
az container start \
  --resource-group myRG \
  --name myContainer

# Delete container
az container delete \
  --resource-group myRG \
  --name myContainer \
  --yes

# ─────────────────────────────────────────────────────────
# MOUNT AZURE FILES VOLUME
# ─────────────────────────────────────────────────────────

# Create storage account and file share
az storage account create \
  --name myacistore \
  --resource-group myRG \
  --sku Standard_LRS

az storage share create \
  --account-name myacistore \
  --name myshare

# Mount file share into container
STORAGE_KEY=$(az storage account keys list \
  --account-name myacistore \
  --resource-group myRG \
  --query [0].value --output tsv)

az container create \
  --resource-group myRG \
  --name myContainerWithStorage \
  --image nginx:latest \
  --cpu 1 \
  --memory 1 \
  --azure-file-volume-account-name myacistore \
  --azure-file-volume-account-key "$STORAGE_KEY" \
  --azure-file-volume-share-name myshare \
  --azure-file-volume-mount-path /data
```

---

### ACI YAML Deployment Example

YAML is ideal for multi-container groups.

```yaml
# container-group.yaml
apiVersion: 2021-09-01
location: eastus
name: my-container-group
type: Microsoft.ContainerInstance/containerGroups
properties:
  osType: Linux
  restartPolicy: OnFailure
  
  containers:
  - name: app
    properties:
      image: myregistry.azurecr.io/myapp:v2.0
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 2.0
      ports:
      - port: 8080
        protocol: TCP
      environmentVariables:
      - name: APP_ENV
        value: production
      - name: DB_PASSWORD
        secureValue: my-secret-password    # Not shown in ARM template
      volumeMounts:
      - name: data-volume
        mountPath: /app/data
        readOnly: false
  
  - name: log-shipper
    properties:
      image: fluent/fluentd:v1.16
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
      volumeMounts:
      - name: data-volume
        mountPath: /fluentd/log
        readOnly: true
  
  ipAddress:
    type: Public
    ports:
    - port: 8080
      protocol: TCP
    dnsNameLabel: my-app-demo
  
  volumes:
  - name: data-volume
    azureFile:
      shareName: myshare
      storageAccountName: myacistore
      storageAccountKey: <storage-key>
  
  imageRegistryCredentials:
  - server: myregistry.azurecr.io
    username: myregistry
    password: <acr-password>
```

```bash
# Deploy from YAML
az container create \
  --resource-group myRG \
  --file container-group.yaml
```

---

## 5. Azure Container Apps

### What is Azure Container Apps?

Azure Container Apps is a serverless container hosting platform built on top of Kubernetes and KEDA (Kubernetes Event-Driven Autoscaling). It abstracts away all Kubernetes complexity while delivering the benefits: microservices, auto-scaling from zero, Dapr integration, and traffic splitting.

```
Where Container Apps fits:

Full Control ◄─────────────────────────────────► Zero Ops
    VMs         AKS       Container Apps    Functions
    (IaaS)   (Managed K8s)  (Serverless     (Serverless
                              containers)     code)

Container Apps = "Serverless Kubernetes" for containers
```

**Use Container Apps when:**
- Building microservices with containers
- You want auto-scaling from zero to N based on HTTP or events
- You need Dapr for service-to-service communication
- You want traffic splitting for blue/green deployments
- Running background jobs that scale on queue depth

**ACI vs Container Apps vs AKS:**

| Scenario | Recommended |
|---|---|
| One-off batch job | ACI |
| Simple web API container | Container Apps |
| Complex microservices with full K8s control | AKS |
| Team needs kubectl, Helm, custom operators | AKS |
| Small team, fast iteration | Container Apps |
| Max 200 concurrent scale | Container Apps |
| 1000+ pods, complex networking | AKS |

---

### Environments and Apps

An **Environment** is a secure boundary that acts like a Kubernetes namespace with shared networking and logging.

```
Container Apps Environment: my-env
┌──────────────────────────────────────────────────────┐
│  Shared Log Analytics Workspace                      │
│  Shared VNet (optional)                              │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Container   │  │  Container   │  │ Container  │ │
│  │  App: api    │  │  App: worker │  │ App: web   │ │
│  │  myapp.env.  │  │  (no ingress)│  │ Port 80    │ │
│  │  region.azca │  │              │  │            │ │
│  │  .net        │  │              │  │            │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                      │
│  Internal: apps talk to each other on internal VNet  │
└──────────────────────────────────────────────────────┘
```

---

### KEDA-Based Scaling

KEDA (Kubernetes Event-Driven Autoscaling) scales containers based on external event sources — not just CPU/memory.

| Scale Rule Type | Scales Based On |
|---|---|
| HTTP | Number of concurrent HTTP requests |
| CPU | CPU utilization % |
| Memory | Memory utilization % |
| Azure Queue | Queue length |
| Service Bus | Active message count |
| Event Hubs | Event lag |
| Custom KEDA scaler | Any KEDA-supported source |

```
KEDA Scaling Behavior:
0 instances ──► (event arrives) ──► 1 instance ──► (load grows) ──► 10 instances
                                    (cold start)                       (max)
              ──► (idle) ──► scale to zero (no cost!)
```

---

### Complete CLI Commands for Container Apps

```bash
# ─────────────────────────────────────────────────────────
# PREREQUISITES
# ─────────────────────────────────────────────────────────

# Install Container Apps extension
az extension add --name containerapp --upgrade

# Register providers
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# ─────────────────────────────────────────────────────────
# CREATE ENVIRONMENT AND APP
# ─────────────────────────────────────────────────────────

# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group myRG \
  --workspace-name myLogWorkspace

LOG_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group myRG \
  --workspace-name myLogWorkspace \
  --query customerId --output tsv)

LOG_WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group myRG \
  --workspace-name myLogWorkspace \
  --query primarySharedKey --output tsv)

# Create Container Apps Environment
az containerapp env create \
  --name myEnv \
  --resource-group myRG \
  --location eastus \
  --logs-workspace-id "$LOG_WORKSPACE_ID" \
  --logs-workspace-key "$LOG_WORKSPACE_KEY"

# Create a Container App with HTTP ingress
az containerapp create \
  --name my-api \
  --resource-group myRG \
  --environment myEnv \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --cpu 0.5 \
  --memory 1Gi \
  --min-replicas 0 \
  --max-replicas 10 \
  --env-vars \
    APP_ENV=production \
    LOG_LEVEL=info

# Get the app URL
az containerapp show \
  --name my-api \
  --resource-group myRG \
  --query properties.configuration.ingress.fqdn \
  --output tsv

# ─────────────────────────────────────────────────────────
# SCALING RULES
# ─────────────────────────────────────────────────────────

# Scale on HTTP requests (built-in)
az containerapp update \
  --name my-api \
  --resource-group myRG \
  --min-replicas 1 \
  --max-replicas 20 \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50  # 1 instance per 50 concurrent requests

# Scale on Azure Storage Queue length
az containerapp update \
  --name my-worker \
  --resource-group myRG \
  --min-replicas 0 \
  --max-replicas 10 \
  --scale-rule-name queue-rule \
  --scale-rule-type azure-queue \
  --scale-rule-metadata "queueName=my-queue" "queueLength=10" "accountName=mysa" \
  --scale-rule-auth "connection=StorageConnectionString:mysa-conn"

# ─────────────────────────────────────────────────────────
# REVISIONS AND TRAFFIC SPLITTING
# ─────────────────────────────────────────────────────────

# Update app (creates new revision)
az containerapp update \
  --name my-api \
  --resource-group myRG \
  --image myregistry.azurecr.io/myapp:v2.0

# List revisions
az containerapp revision list \
  --name my-api \
  --resource-group myRG \
  --output table

# Split traffic 80/20 between two revisions
az containerapp ingress traffic set \
  --name my-api \
  --resource-group myRG \
  --revision-weight \
    my-api--revision-v1=80 \
    my-api--revision-v2=20

# Set full traffic to new revision (complete rollout)
az containerapp ingress traffic set \
  --name my-api \
  --resource-group myRG \
  --label-weight latest=100

# ─────────────────────────────────────────────────────────
# SECRETS AND ENVIRONMENT VARIABLES
# ─────────────────────────────────────────────────────────

# Set a secret
az containerapp secret set \
  --name my-api \
  --resource-group myRG \
  --secrets db-password=mysecretvalue

# Reference secret as environment variable
az containerapp update \
  --name my-api \
  --resource-group myRG \
  --set-env-vars "DB_PASSWORD=secretref:db-password"

# ─────────────────────────────────────────────────────────
# DAPR INTEGRATION
# ─────────────────────────────────────────────────────────

# Enable Dapr on a Container App
az containerapp dapr enable \
  --name my-api \
  --resource-group myRG \
  --dapr-app-id my-api \
  --dapr-app-port 80 \
  --dapr-app-protocol http

# Create Dapr component (e.g., state store using Redis)
az containerapp env dapr-component set \
  --name myEnv \
  --resource-group myRG \
  --dapr-component-name statestore \
  --yaml - <<'EOF'
componentType: state.redis
version: v1
metadata:
- name: redisHost
  value: myredis.redis.cache.windows.net:6380
- name: redisPassword
  secretRef: redis-password
- name: enableTLS
  value: "true"
scopes:
- my-api
- my-worker
EOF
```

### Container Apps Best Practices

| Area | Best Practice |
|---|---|
| **Scale** | Always set min-replicas=0 for background workers to save cost |
| **Scale** | Use HTTP concurrency-based scaling for APIs |
| **Security** | Use Managed Identity for ACR and Azure services |
| **Secrets** | Store secrets in Container Apps Secrets, not environment variables |
| **Networking** | Use internal ingress for service-to-service communication |
| **Revisions** | Use revision labels (latest, stable) for traffic management |
| **Dapr** | Use Dapr for service discovery, pub/sub, state management |
| **Monitoring** | Connect to Log Analytics; use Application Insights |

---

## 6. Azure Kubernetes Service (AKS)

> **Note:** AKS fundamentals are covered in [02-AZURE-SERVICES-DEEP-DIVE.md](02-AZURE-SERVICES-DEEP-DIVE.md). This section covers advanced topics.

### Node Pools

An AKS cluster has two types of node pools:

#### System Node Pool
- **Required:** Every cluster must have at least one system pool.
- **Purpose:** Runs critical system pods: `coredns`, `metrics-server`, `konnectivity-agent`, CSI drivers.
- **Taint:** Automatically tainted with `CriticalAddonsOnly=true:NoSchedule` — prevents your workloads from running here.
- **Recommended size:** Standard_D4s_v5 or larger.

#### User Node Pool
- **Purpose:** Runs your application workloads.
- **Benefits:** Can use different VM sizes, OS types (Windows/Linux), or spot instances.
- **Scale to zero:** Can be scaled down to 0 nodes when idle.

```bash
# Create AKS cluster with system node pool
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5 \
  --nodepool-name systempool \
  --nodepool-mode System \
  --zones 1 2 3 \                  # Spread across availability zones
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5 \
  --generate-ssh-keys

# Add a GPU user node pool for ML workloads
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name gpupool \
  --mode User \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints sku=gpu:NoSchedule \    # Taint to require GPU tolerations
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 4

# Add a Windows user node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name winpool \
  --mode User \
  --os-type Windows \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5

# Add a Spot user node pool (80-90% cost reduction)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name spotpool \
  --mode User \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \             # -1 = up to on-demand price
  --node-count 3 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 10
```

---

### AKS Networking Models

#### Kubenet (Basic Networking)

Nodes get IPs from the Azure VNet subnet. Pods get IPs from a **separate overlay network** inside the node. Inter-node pod traffic is routed via user-defined routes (UDRs).

```
Kubenet Network Layout:
Azure VNet: 10.0.0.0/16
  └── Node Subnet: 10.0.1.0/24
        ├── Node 1: 10.0.1.4
        │     └── Pod Network: 10.244.0.0/24 (overlay)
        │           ├── pod-a: 10.244.0.5
        │           └── pod-b: 10.244.0.6
        └── Node 2: 10.0.1.5
              └── Pod Network: 10.244.1.0/24 (overlay)
                    ├── pod-c: 10.244.1.5
                    └── pod-d: 10.244.1.6

Pod-to-pod (cross-node): routed via UDR on Azure
```

**Pros of Kubenet:** Uses fewer IPs from VNet; simpler VNet design.  
**Cons:** UDR limit (400 routes max); no direct IP for pods from VNet; some services (internal load balancers) don't support it.

#### Azure CNI (Advanced Networking)

Every pod gets an IP directly from the VNet subnet. No overlay. Pods are first-class VNet citizens.

```
Azure CNI Network Layout:
Azure VNet: 10.0.0.0/16
  └── Node + Pod Subnet: 10.0.0.0/22 (needs large subnet!)
        ├── Node 1: 10.0.0.4    Pod 1: 10.0.1.5   Pod 2: 10.0.1.6
        ├── Node 2: 10.0.0.5    Pod 3: 10.0.2.5   Pod 4: 10.0.2.6
        └── Node 3: 10.0.0.6    Pod 5: 10.0.3.5   Pod 6: 10.0.3.6

All IPs come from the same VNet — no overlay, no UDRs
```

**Pros of Azure CNI:** Full VNet visibility for pods; supports Windows nodes; Private Endpoints accessible directly; no UDR limits.  
**Cons:** Requires large subnet (IPs per node × max nodes); IP exhaustion risk.

#### Azure CNI Overlay (Best of Both Worlds)

Nodes get IPs from VNet; pods get IPs from a private CIDR that is overlaid and does not consume VNet IPs. No UDR needed. Newer option recommended for large clusters.

```bash
# Create cluster with Azure CNI Overlay (recommended for new clusters)
az aks create \
  --resource-group myRG \
  --name myAKS \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \       # Pod IPs (not in VNet)
  --service-cidr 10.100.0.0/16 \   # Service IPs (not in VNet)
  --dns-service-ip 10.100.0.10 \
  --vnet-subnet-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/aks-subnet
```

---

### AKS Add-ons

| Add-on | Name | Purpose |
|---|---|---|
| Azure Monitor | `monitoring` | Container Insights — metrics and logs |
| Azure Policy | `azure-policy` | Enforce policies on pods (no privileged, required labels) |
| KEDA | `keda` | Kubernetes Event-Driven Autoscaling |
| Application Gateway Ingress | `ingress-appgw` | AGIC — App Gateway as ingress controller |
| Azure Key Vault Provider | `azure-keyvault-secrets-provider` | Mount secrets as volumes |
| Open Service Mesh | `open-service-mesh` | Service mesh |
| Web Application Routing | `web_application_routing` | Simplified ingress with DNS/SSL |

```bash
# Enable Container Insights monitoring
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons monitoring \
  --workspace-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLogWorkspace

# Enable Azure Key Vault Secrets Provider
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m

# Enable KEDA
az aks update \
  --resource-group myRG \
  --name myAKS \
  --enable-keda

# Enable Application Gateway Ingress Controller
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons ingress-appgw \
  --appgw-name myAppGateway \
  --appgw-subnet-cidr "10.2.0.0/16"

# Enable Azure Policy
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons azure-policy
```

---

### AKS Upgrades and Maintenance Windows

```bash
# List available Kubernetes versions in a region
az aks get-versions \
  --location eastus \
  --output table

# Check the upgrade path for your cluster
az aks get-upgrades \
  --resource-group myRG \
  --name myAKS \
  --output table

# Upgrade the control plane only (first)
az aks upgrade \
  --resource-group myRG \
  --name myAKS \
  --kubernetes-version 1.29.2 \
  --control-plane-only

# Upgrade a specific node pool
az aks nodepool upgrade \
  --resource-group myRG \
  --cluster-name myAKS \
  --name nodepool1 \
  --kubernetes-version 1.29.2 \
  --node-image-upgrade-only    # Upgrade just node image, not K8s version

# Configure planned maintenance window (avoid disruption)
az aks maintenanceconfiguration add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name default \
  --weekday Sunday \
  --start-hour 2    # Maintenance at 2am on Sundays

# Auto-upgrade channel
az aks update \
  --resource-group myRG \
  --name myAKS \
  --auto-upgrade-channel patch    # stable, patch, rapid, node-image, none
```

**Upgrade strategy:**
1. Always upgrade control plane first, then node pools
2. Test in non-production first
3. Use `--node-surge` to control surge (extra nodes during upgrade)
4. Use maintenance windows to schedule upgrades
5. Never skip more than one minor version (e.g., 1.27 → 1.28 → 1.29, not 1.27 → 1.29)

---

### Virtual Nodes (ACI Integration)

Virtual Nodes allow AKS to burst workloads onto Azure Container Instances instantly, without waiting for new nodes to provision.

```
AKS Cluster with Virtual Nodes:
┌─────────────────────────────────────────────────┐
│  AKS Cluster                                    │
│                                                 │
│  Node 1 (VM)     Node 2 (VM)    Virtual Node   │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │ pod1 pod2  │  │ pod3 pod4  │  │ pod5      │ │
│  │            │  │            │  │ (ACI!)    │ │
│  └────────────┘  └────────────┘  └───────────┘ │
│                                    ↑            │
│                              Burst pods onto    │
│                              ACI — seconds,     │
│                              not minutes        │
└─────────────────────────────────────────────────┘
```

```bash
# Enable virtual nodes add-on
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons virtual-node \
  --subnet-name aci-subnet

# Deploy a pod to virtual node (use nodeSelector)
# In your Kubernetes deployment manifest:
# spec:
#   nodeSelector:
#     kubernetes.io/role: agent
#     beta.kubernetes.io/os: linux
#     type: virtual-kubelet
#   tolerations:
#   - key: virtual-kubelet.io/provider
#     operator: Exists
```

---

## 7. Azure Batch

### What is Azure Batch?

Azure Batch is a cloud-scale job scheduling service for running large-scale parallel and HPC (High Performance Computing) workloads. You define pools of compute nodes, jobs, and tasks. Batch schedules tasks across nodes automatically.

```
Azure Batch Architecture:
┌──────────────────────────────────────────────────────────┐
│  Batch Account                                           │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Pool (VM fleet)                                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │ │
│  │  │  Node 1  │  │  Node 2  │  │  Node 3  │  ...   │ │
│  │  │  D4s_v5  │  │  D4s_v5  │  │  D4s_v5  │        │ │
│  │  └──────────┘  └──────────┘  └──────────┘        │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Job (logical grouping of tasks)                   │ │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │ │
│  │  │Task 1│  │Task 2│  │Task 3│  │Task N│          │ │
│  │  │item1 │  │item2 │  │item3 │  │itemN │          │ │
│  │  └──────┘  └──────┘  └──────┘  └──────┘          │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  Batch Scheduler assigns tasks to available nodes       │
└──────────────────────────────────────────────────────────┘
```

**Use Azure Batch when:**
- Rendering 3D frames (each frame = one task)
- Processing millions of files (each file = one task)
- Running Monte Carlo simulations
- Genomics pipelines (each sample = one task)
- Financial modeling (each scenario = one task)
- Machine learning parameter sweeps

**Don't use Azure Batch when:**
- You need a web server or API (use App Service, AKS)
- You need real-time processing (use Functions, Event Hubs)
- Your job is a single long-running process (use a VM)

---

### Pools, Jobs, and Tasks

| Concept | Description | Analogy |
|---|---|---|
| **Batch Account** | Top-level resource | Factory |
| **Pool** | Set of compute nodes (VMs) | Factory floor with machines |
| **Job** | Logical work container | Production order |
| **Task** | Individual unit of work assigned to a node | Single manufacturing step |
| **Job Schedule** | Recurring job definition | Recurring production schedule |

---

### Complete CLI Commands for Batch

```bash
# ─────────────────────────────────────────────────────────
# CREATE BATCH ACCOUNT
# ─────────────────────────────────────────────────────────

# Create storage account for Batch output
az storage account create \
  --name mybatchsa \
  --resource-group myRG \
  --sku Standard_LRS \
  --location eastus

# Create Batch account
az batch account create \
  --name mybatchaccount \
  --resource-group myRG \
  --location eastus \
  --storage-account mybatchsa

# Log in to Batch account
az batch account login \
  --name mybatchaccount \
  --resource-group myRG \
  --shared-key-auth

# ─────────────────────────────────────────────────────────
# CREATE POOL
# ─────────────────────────────────────────────────────────

# Create a pool of Ubuntu 22.04 VMs with auto-scale
az batch pool create \
  --id mypool \
  --vm-size Standard_D4s_v5 \
  --target-dedicated-nodes 0 \
  --target-low-priority-nodes 5 \    # Spot VMs for 80-90% savings
  --image \
    publisher=Canonical \
    offer=0001-com-ubuntu-server-jammy \
    sku=22_04-lts-gen2 \
    version=latest \
  --node-agent-sku-id "batch.node.ubuntu 22.04" \
  --auto-scale-formula \
    '$TargetDedicatedNodes = 0; $TargetLowPriorityNodes = min(pendingTaskCount, 20);' \
  --auto-scale-evaluation-interval PT5M

# Show pool status
az batch pool show --pool-id mypool

# ─────────────────────────────────────────────────────────
# CREATE JOB AND TASKS
# ─────────────────────────────────────────────────────────

# Create a job
az batch job create \
  --id myjob \
  --pool-id mypool

# Add a single task to the job
az batch task create \
  --job-id myjob \
  --task-id task001 \
  --command-line "bash -c 'echo processing item_001 && sleep 10 && echo done'"

# Add multiple tasks from a JSON file
cat > tasks.json << 'EOF'
[
  {
    "id": "task001",
    "commandLine": "python /mnt/batch/tasks/workitems/myjob/job-1/task001/wd/process.py --input data001.csv"
  },
  {
    "id": "task002",
    "commandLine": "python /mnt/batch/tasks/workitems/myjob/job-1/task002/wd/process.py --input data002.csv"
  }
]
EOF

az batch task create \
  --job-id myjob \
  --json-file tasks.json

# ─────────────────────────────────────────────────────────
# MONITOR JOBS AND TASKS
# ─────────────────────────────────────────────────────────

# List jobs
az batch job list --output table

# Show job status
az batch job show --job-id myjob

# List tasks in a job
az batch task list --job-id myjob --output table

# Show task details (including exit code, stdout, stderr)
az batch task show --job-id myjob --task-id task001

# Get task stdout
az batch task file download \
  --job-id myjob \
  --task-id task001 \
  --file-path stdout.txt \
  --destination ./task001-stdout.txt

# ─────────────────────────────────────────────────────────
# CLEANUP
# ─────────────────────────────────────────────────────────

# Delete job (and all its tasks)
az batch job delete --job-id myjob --yes

# Delete pool
az batch pool delete --pool-id mypool --yes
```

---

## 8. Virtual Machine Scale Sets (VMSS)

### What is VMSS?

Virtual Machine Scale Sets let you deploy and manage a group of **identical, load-balanced VMs**. VMSS manages VM provisioning, configuration, and scaling automatically. All VMs in a scale set are created from the same base OS image and configuration.

```
VMSS Architecture:
┌─────────────────────────────────────────────────────────┐
│  VM Scale Set: my-vmss                                  │
│                                                         │
│  Load Balancer                                          │
│  ┌───────────────────────────────────────────────────┐ │
│  │  Instance 1  Instance 2  Instance 3  ...           │ │
│  │  10.0.1.4    10.0.1.5    10.0.1.6                 │ │
│  │  (same image, same config, same size)              │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  Auto-Scaler ──► (CPU > 70%) ──► Add 2 instances       │
│                  (CPU < 30%) ──► Remove 1 instance      │
└─────────────────────────────────────────────────────────┘
```

**Use VMSS when:**
- You need multiple identical VMs behind a load balancer
- You need auto-scaling based on metrics
- Running web tiers, application tiers, or compute farms
- You need OS-level control that App Service doesn't provide

---

### Uniform vs Flexible Orchestration

| Feature | Uniform VMSS | Flexible VMSS |
|---|---|---|
| VM management | All identical, API-managed | Individual VM management |
| Instance uniqueness | All same | Mix of VM types possible |
| Max instances | 1000 | 1000 |
| Availability Zones | Yes | Yes |
| Spot mix | All or none | Mix spot and regular |
| Deployment | Scale set API only | Standard VM APIs |
| Use case | Stateless app tiers | Mixed workloads |

**Uniform:** Best for homogeneous stateless app tiers (web servers, compute workers) where all VMs are identical.  
**Flexible:** Best when you need to manage individual VMs differently, or mix spot + regular instances, or use full VM management APIs.

---

### Complete CLI Commands for VMSS

```bash
# ─────────────────────────────────────────────────────────
# CREATE VMSS
# ─────────────────────────────────────────────────────────

# Create a uniform VMSS (Linux, manual scaling)
az vmss create \
  --resource-group myRG \
  --name myVMSS \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --orchestration-mode Uniform \
  --upgrade-policy-mode Automatic \  # Rolling, Manual, Automatic
  --zones 1 2 3                       # Spread across AZs

# Create VMSS with custom script (install nginx on all instances)
az vmss create \
  --resource-group myRG \
  --name myWebVMSS \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --load-balancer myLB \
  --backend-pool-name myBackendPool

# Install extension on all VMSS instances
az vmss extension set \
  --resource-group myRG \
  --vmss-name myWebVMSS \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"commandToExecute": "apt-get update && apt-get install -y nginx && systemctl enable nginx && systemctl start nginx"}'

# ─────────────────────────────────────────────────────────
# MANUAL SCALING
# ─────────────────────────────────────────────────────────

# Scale to 5 instances
az vmss scale \
  --resource-group myRG \
  --name myVMSS \
  --new-capacity 5

# ─────────────────────────────────────────────────────────
# AUTOSCALE
# ─────────────────────────────────────────────────────────

# Enable autoscale
az monitor autoscale create \
  --resource-group myRG \
  --name myVMSSAutoscale \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --min-count 2 \
  --max-count 20 \
  --count 2

# Add scale-out rule (CPU > 75% for 5 min → add 2 VMs)
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myVMSSAutoscale \
  --condition "Percentage CPU > 75 avg 5m" \
  --scale out 2 \
  --cooldown 5

# Add scale-in rule (CPU < 25% for 10 min → remove 1 VM)
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myVMSSAutoscale \
  --condition "Percentage CPU < 25 avg 10m" \
  --scale in 1 \
  --cooldown 10

# ─────────────────────────────────────────────────────────
# MANAGE INSTANCES
# ─────────────────────────────────────────────────────────

# List instances
az vmss list-instances \
  --resource-group myRG \
  --name myVMSS \
  --output table

# Restart specific instance
az vmss restart \
  --resource-group myRG \
  --name myVMSS \
  --instance-ids 1 2 3

# Update model (e.g., change VM size — requires reimage)
az vmss update \
  --resource-group myRG \
  --name myVMSS \
  --vm-sku Standard_D4s_v5

# Apply model update to all instances
az vmss update-instances \
  --resource-group myRG \
  --name myVMSS \
  --instance-ids "*"

# Reimage all instances (re-apply from base image)
az vmss reimage \
  --resource-group myRG \
  --name myVMSS \
  --instance-ids "*"

# Delete VMSS
az vmss delete \
  --resource-group myRG \
  --name myVMSS
```

---

## 9. Compute Decision Guide

### When to Use Each Service

```
START HERE: What are you running?
│
├── A full workload requiring custom OS, specific drivers, or legacy software?
│   └── Azure VM (IaaS full control)
│
├── Multiple identical VMs with load balancing and auto-scale?
│   └── VM Scale Sets (VMSS)
│
├── A web application or REST API?
│   ├── Containerized?
│   │   ├── Single container, simple?  ──► Azure Container Instances (ACI)
│   │   ├── Microservices, serverless? ──► Azure Container Apps
│   │   └── Full K8s control needed?  ──► AKS
│   └── Not containerized?
│       └── Azure App Service
│
├── Event-driven, short-lived functions?
│   └── Azure Functions (Serverless)
│
├── HPC, batch processing, rendering?
│   └── Azure Batch
│
└── Not sure? Start here:
    ├── Team is small / wants to move fast → App Service or Container Apps
    ├── Need full control / existing K8s skills → AKS
    └── Minimizing cost for bursty → Functions (Consumption)
```

### Pricing Model Summary

| Service | Pricing Unit | Free Tier | Key Cost Driver |
|---|---|---|---|
| VM | Per hour (compute + storage) | No (B1s is ~$8/mo) | VM size + uptime |
| App Service | Per App Service Plan/hour | F1 (60 CPU min/day) | Plan tier + instances |
| Functions | Per execution + GB-seconds | 1M executions/month | Execution count + duration |
| ACI | Per vCPU-second + GB-second | No | CPU+RAM × runtime |
| Container Apps | Per vCPU-second + GB-second | 180,000 vCPU-s/month | Active replicas × time |
| AKS | Per VM node hour (control plane free) | No | Node VM size + count |
| Batch | Per VM hour (pool nodes) | No | Node count × uptime |
| VMSS | Same as VMs | No | VM size + instance count |

---

*Document 10 — Azure Compute Services Complete Reference*  
*Part of the Azure Zero-to-Hero Learning Series*
