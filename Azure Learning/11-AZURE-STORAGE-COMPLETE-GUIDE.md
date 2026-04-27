# Azure Storage — Complete Reference Guide
## Document 11: Zero-to-Hero Storage Handbook

**Last Updated:** 2026  
**Document Version:** 1.0  
**Focus:** Every Azure Storage service — Blob, Files, Queue, Table, ADLS Gen2, Managed Disks, Security, AzCopy

---

## TABLE OF CONTENTS

1. [Azure Storage Account](#1-azure-storage-account)
2. [Azure Blob Storage](#2-azure-blob-storage)
3. [Azure Files](#3-azure-files)
4. [Azure Queue Storage](#4-azure-queue-storage)
5. [Azure Table Storage](#5-azure-table-storage)
6. [Azure Data Lake Storage Gen2 (ADLS Gen2)](#6-azure-data-lake-storage-gen2)
7. [Azure Managed Disks](#7-azure-managed-disks)
8. [Storage Security](#8-storage-security)
9. [AzCopy Complete Reference](#9-azcopy-complete-reference)

---

## Azure Storage Overview

```
Azure Storage Account — The Container for All Storage Services
┌──────────────────────────────────────────────────────────────────┐
│  Storage Account: mystorageaccount                               │
│  Location: East US | Replication: GRS | Tier: Standard          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  ┌────────┐  │
│  │ Blob Storage │  │ Azure Files  │  │  Queues  │  │ Tables │  │
│  │ (containers) │  │ (file shares)│  │          │  │        │  │
│  │              │  │              │  │          │  │        │  │
│  │ Block blobs  │  │ SMB 3.0      │  │ Max 64KB │  │ NoSQL  │  │
│  │ Page blobs   │  │ NFS 4.1      │  │ per msg  │  │ KV     │  │
│  │ Append blobs │  │              │  │          │  │        │  │
│  └──────────────┘  └──────────────┘  └──────────┘  └────────┘  │
│                                                                  │
│  Also has: Static Website hosting, ADLS Gen2 (if HNS enabled)   │
└──────────────────────────────────────────────────────────────────┘
```

| Service | Use Case | Max Size |
|---|---|---|
| Blob Storage | Unstructured data, images, videos, backups | 4.75 TB per blob (block) |
| Azure Files | Network file shares (SMB/NFS) | 100 TiB per share |
| Queue Storage | Message queuing between services | 64 KB per message |
| Table Storage | NoSQL key-value store | No limit |
| ADLS Gen2 | Big data analytics with HDFS-compatible API | Petabytes |
| Managed Disks | VM OS and data disks | 32 TiB |

---

## 1. Azure Storage Account

### What is a Storage Account?

A storage account is the top-level Azure resource that provides a unique namespace for all your Azure Storage services. Every blob, file, queue, and table lives inside a storage account.

The storage account provides:
- **A unique namespace:** `https://<account>.blob.core.windows.net`
- **Access control:** Keys, SAS tokens, Azure AD, private endpoints
- **Replication:** LRS, ZRS, GRS, GZRS for durability
- **Tiers:** Standard (HDD-backed) or Premium (SSD-backed)

---

### Storage Account Types

| Type | Description | Services | Use Case |
|---|---|---|---|
| **General Purpose v2 (GPv2)** | Standard; recommended for most | Blob, Files, Queue, Table | Default choice for everything |
| **Blob Storage** | Legacy blob-only account | Blob only | Blob-only workloads (migrate to GPv2) |
| **BlockBlobStorage** | Premium performance for block blobs | Block Blob, Append Blob | Low-latency blob I/O, AI/ML pipelines |
| **FileStorage** | Premium performance for file shares | Azure Files only | High-performance SMB/NFS shares |

**Always use GPv2** unless you specifically need Premium for blobs (BlockBlobStorage) or file shares (FileStorage).

---

### Performance Tiers

| Tier | Backend | Latency | Use Case |
|---|---|---|---|
| **Standard** | HDD-backed | ms to seconds | General use, backups, cold data |
| **Premium** | SSD-backed | Single-digit ms | Low-latency databases, media processing |

**Standard:** Default. Works for most workloads. Supports all replication options.  
**Premium:** Only available for BlockBlobStorage and FileStorage account types. Does not support RA-GRS replication.

---

### Replication Options

Replication protects your data from hardware failures. Azure offers several levels of redundancy.

```
LRS — Locally Redundant Storage (3 copies in 1 datacenter)
┌──────────────────────────┐
│  Datacenter A            │
│  ┌──────┐ ┌──────┐ ┌──┐ │
│  │ Copy1│ │ Copy2│ │C3│ │
│  └──────┘ └──────┘ └──┘ │
└──────────────────────────┘
SLA: 99.999999999% (11 nines) durability
Protects: Drive/rack failure
Does NOT protect: Datacenter fire, flood, power outage

ZRS — Zone-Redundant Storage (1 copy in each of 3 zones)
Zone 1       Zone 2       Zone 3
┌──────┐     ┌──────┐     ┌──────┐
│ Copy1│     │ Copy2│     │ Copy3│
└──────┘     └──────┘     └──────┘
SLA: 99.9999999999% (12 nines) durability
Protects: Full datacenter/zone failure

GRS — Geo-Redundant Storage (LRS in primary + LRS in secondary)
Primary Region              Secondary Region (paired)
┌────────────────────┐      ┌────────────────────┐
│ DC A               │      │ DC B               │
│ Copy1 Copy2 Copy3  │─────►│ Copy4 Copy5 Copy6  │
└────────────────────┘      └────────────────────┘
SLA: 99.99999999999999% (16 nines) durability
Secondary: Read-only, only accessible after failover (unless RA-GRS)

GZRS — Geo-Zone-Redundant Storage (ZRS in primary + LRS in secondary)
Primary Region (3 zones)    Secondary Region
┌──────┬──────┬──────┐      ┌────────────────────┐
│Zone1 │Zone2 │Zone3 │─────►│ Copy4 Copy5 Copy6  │
└──────┴──────┴──────┘      └────────────────────┘
Best overall protection

RA-GRS / RA-GZRS — Read Access variants
Same as GRS/GZRS but secondary region is readable at all times
URL: https://<account>-secondary.blob.core.windows.net
```

| Replication | Copies | Regions | Read Secondary | Annual Durability | Monthly Cost (vs LRS) |
|---|---|---|---|---|---|
| LRS | 3 | 1 | No | 11 nines | Baseline |
| ZRS | 3 | 1 (3 zones) | No | 12 nines | +25% |
| GRS | 6 | 2 | No | 16 nines | +50% |
| RA-GRS | 6 | 2 | Yes | 16 nines | +100% |
| GZRS | 6 | 2 (ZRS+GRS) | No | 16 nines | +75% |
| RA-GZRS | 6 | 2 | Yes | 16 nines | +125% |

**Recommendation by tier:**
- Dev/test → LRS (cheapest)
- Important single-region → ZRS
- Business-critical data → GRS or GZRS
- Compliance/DR with read access → RA-GRS or RA-GZRS

---

### Access Tiers

Access tiers apply to **Blob Storage only** and control storage vs access cost trade-offs.

| Tier | Storage Cost | Access Cost | Retrieval Time | Min Storage Duration | Use Case |
|---|---|---|---|---|---|
| **Hot** | High | Low | Immediate | None | Frequently accessed data |
| **Cool** | Medium | Medium | Immediate | 30 days | Infrequently accessed, short-term backup |
| **Cold** | Low | High | Immediate | 90 days | Rarely accessed, long-term backup |
| **Archive** | Very Low | Very High | 1-15 hours | 180 days | Compliance, rarely accessed |

```
Access Tier Cost Relationship:
                Hot    Cool   Cold   Archive
Storage cost:   ████   ███    ██     █
Access cost:    █      ██     ███    █████
Retrieval:     Instant Instant Instant 1-15hrs
```

**Rehydration from Archive:** Moving a blob from Archive to Hot/Cool/Cold requires rehydration. Priority levels:
- **Standard:** 1-15 hours (lower cost)
- **High:** Within 1 hour (higher cost)

```bash
# Rehydrate a blob from Archive to Hot
az storage blob set-tier \
  --account-name mysa \
  --container-name mycontainer \
  --name archived-backup.zip \
  --tier Hot \
  --rehydrate-priority High
```

---

### Storage Firewall and Virtual Network Rules

```
Storage Account Network Access Control:
                           ┌────────────────────────────┐
                           │  Storage Account            │
  Internet ──────── Denied │                            │
  (by default when         │  Firewall Rules:           │
   firewall enabled)       │  - Allow IP: 203.0.113.0   │
                           │  - Allow VNet: myVNet/sub1 │
  Your VNet ──── Allowed ──►│  - Allow Azure Services    │
                           │                            │
  Trusted Azure ── Allowed ►│  Private Endpoint (optional)│
  Services                 └────────────────────────────┘
```

---

### Complete CLI Commands for Storage Accounts

```bash
# ─────────────────────────────────────────────────────────
# CREATE STORAGE ACCOUNTS
# ─────────────────────────────────────────────────────────

# Create Standard GPv2 storage account (recommended default)
az storage account create \
  --name mystorageaccount123 \
  --resource-group myRG \
  --location eastus \
  --sku Standard_GRS \          # LRS, ZRS, GRS, GZRS, RAGRS, RAGZRS
  --kind StorageV2 \
  --access-tier Hot \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --https-only true

# Create Premium BlockBlob account (for low-latency blob I/O)
az storage account create \
  --name mypremiumblob123 \
  --resource-group myRG \
  --location eastus \
  --sku Premium_LRS \
  --kind BlockBlobStorage \
  --https-only true

# Create Premium FileStorage account (for high-perf shares)
az storage account create \
  --name mypremiumfiles123 \
  --resource-group myRG \
  --location eastus \
  --sku Premium_LRS \
  --kind FileStorage

# ─────────────────────────────────────────────────────────
# MANAGE STORAGE ACCOUNTS
# ─────────────────────────────────────────────────────────

# List storage accounts in subscription
az storage account list --output table

# Show account details
az storage account show \
  --name mystorageaccount123 \
  --resource-group myRG \
  --output json

# Get connection string
az storage account show-connection-string \
  --name mystorageaccount123 \
  --resource-group myRG \
  --output tsv

# Get account keys
az storage account keys list \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --output table

# Rotate key (regenerate)
az storage account keys renew \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --key primary

# ─────────────────────────────────────────────────────────
# NETWORK RULES
# ─────────────────────────────────────────────────────────

# Enable firewall (deny all public access by default)
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --default-action Deny

# Allow specific IP range
az storage account network-rule add \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --ip-address 203.0.113.0/24

# Allow specific VNet subnet
az storage account network-rule add \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet

# Allow Azure trusted services (Azure Backup, Azure Data Factory, etc.)
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --bypass AzureServices Logging Metrics

# ─────────────────────────────────────────────────────────
# CONFIGURE BLOB PUBLIC ACCESS
# ─────────────────────────────────────────────────────────

# Disable public blob access (security best practice)
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --allow-blob-public-access false

# Upgrade to GPv2 from GPv1
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --set kind=StorageV2

# Change replication
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --sku Standard_RAGRS

# Change default access tier
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --access-tier Cool

# Delete storage account
az storage account delete \
  --name mystorageaccount123 \
  --resource-group myRG \
  --yes
```

---

## 2. Azure Blob Storage

### What is Blob Storage?

Azure Blob Storage is Microsoft's object storage solution for the cloud. "Blob" stands for Binary Large OBject. It stores unstructured data — anything that doesn't fit in a relational table: images, videos, documents, backups, log files, VM disk images, ML datasets.

```
Blob Storage Hierarchy:
Storage Account
└── Container (like a folder, but flat)
    ├── block-blob-1.jpg            (Block Blob)
    ├── append-log.txt              (Append Blob)
    ├── virtual/folder/myfile.json  (simulated folder with / in name)
    └── page-blob.vhd               (Page Blob)
```

---

### Blob Types

#### Block Blob
The most common type. Data is stored in blocks that can be uploaded independently and committed together. Best for discrete, large files.

- **Max size:** 4.75 TB (50,000 blocks × 4,000 MB each)
- **Use for:** Images, videos, documents, backups, log archives
- **Upload pattern:** Upload blocks (parallel), then commit block list

#### Page Blob
Optimized for random read/write operations. Organized into 512-byte pages.

- **Max size:** 8 TB
- **Use for:** Azure VM unmanaged disks (VHDs), databases requiring random I/O
- **Note:** For VM disks, always prefer Managed Disks over page blobs

#### Append Blob
Optimized for append operations only. You can only add data to the end.

- **Max size:** ~195 GB
- **Use for:** Log files, audit trails, IoT telemetry streams
- **Note:** Cannot modify existing data; only append

---

### Blob Lifecycle Management Policies

Lifecycle policies automatically transition blobs between access tiers or delete them based on age.

```
Lifecycle Policy Example:
Upload ──► Hot (0-30 days) ──► Cool (30-90 days) ──► Cold (90-365 days) ──► Archive (365+ days) ──► Delete (730+ days)
```

```json
{
  "rules": [
    {
      "name": "tier-down-policy",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/", "backups/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToCold": {
              "daysAfterModificationGreaterThan": 90
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
            },
            "delete": {
              "daysAfterModificationGreaterThan": 730
            }
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          },
          "version": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        }
      }
    }
  ]
}
```

```bash
# Apply lifecycle policy from JSON file
az storage account management-policy create \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --policy @lifecycle-policy.json

# Show current policy
az storage account management-policy show \
  --account-name mystorageaccount123 \
  --resource-group myRG
```

---

### Immutable Blob Storage

Immutable blob storage allows you to write data once and then not modify or delete it for a specified period. Required for WORM (Write Once, Read Many) compliance.

#### Time-Based Retention Policy
Blobs cannot be deleted or modified for N days after they are written.

```bash
# Create immutability policy on a container
az storage container immutability-policy create \
  --account-name mystorageaccount123 \
  --container-name compliance-data \
  --period 365 \           # Retention in days
  --allow-protected-append-writes true

# Lock the policy (makes it irrevocable — cannot be removed or shortened!)
az storage container immutability-policy lock \
  --account-name mystorageaccount123 \
  --container-name compliance-data \
  --if-match "*"
```

#### Legal Hold Policy
Indefinite hold — blobs cannot be deleted until hold is explicitly removed.

```bash
# Set a legal hold tag
az storage container legal-hold set \
  --account-name mystorageaccount123 \
  --container-name legal-case-123 \
  --tags "case-2024-001" "case-2024-002"

# Clear a legal hold tag
az storage container legal-hold clear \
  --account-name mystorageaccount123 \
  --container-name legal-case-123 \
  --tags "case-2024-001"
```

---

### Blob Versioning and Soft Delete

#### Versioning
Every write operation on a blob creates a new version. You can restore a previous version at any time.

```bash
# Enable versioning
az storage account blob-service-properties update \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --enable-versioning true

# List versions of a blob
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --include v \                  # Include versions
  --query "[?name=='myfile.txt']" \
  --output table

# Restore a specific version (copy version to current)
az storage blob copy start \
  --account-name mystorageaccount123 \
  --destination-container mycontainer \
  --destination-blob myfile.txt \
  --source-account-name mystorageaccount123 \
  --source-container mycontainer \
  --source-blob myfile.txt \
  --source-version-id 2024-01-15T10:00:00.0000000Z
```

#### Soft Delete
When soft delete is enabled, deleted blobs are retained for a configurable number of days before permanent removal.

```bash
# Enable soft delete (keep deleted blobs for 14 days)
az storage account blob-service-properties update \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --enable-delete-retention true \
  --delete-retention-days 14

# Enable soft delete for containers too
az storage account blob-service-properties update \
  --account-name mystorageaccount123 \
  --resource-group myRG \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

# List soft-deleted blobs
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --include d \                  # Include deleted
  --output table

# Restore a soft-deleted blob
az storage blob undelete \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt
```

---

### Complete CLI Commands for Blob Storage

```bash
# ─────────────────────────────────────────────────────────
# CONTAINERS
# ─────────────────────────────────────────────────────────

# Set environment variable for convenience
export AZURE_STORAGE_ACCOUNT=mystorageaccount123
export AZURE_STORAGE_KEY=$(az storage account keys list \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --resource-group myRG \
  --query [0].value --output tsv)

# Create a container
az storage container create \
  --name mycontainer \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --public-access off          # off, blob, container

# List containers
az storage container list \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --output table

# Show container properties
az storage container show \
  --name mycontainer \
  --account-name $AZURE_STORAGE_ACCOUNT

# Delete a container (and ALL blobs in it!)
az storage container delete \
  --name mycontainer \
  --account-name $AZURE_STORAGE_ACCOUNT

# ─────────────────────────────────────────────────────────
# UPLOAD BLOBS
# ─────────────────────────────────────────────────────────

# Upload a single file
az storage blob upload \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./local/myfile.txt \
  --overwrite

# Upload with access tier and metadata
az storage blob upload \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name archive/backup-2024.zip \
  --file ./backup-2024.zip \
  --tier Cool \
  --metadata environment=production backup-date=2024-01-01

# Upload entire directory (batch upload)
az storage blob upload-batch \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --destination mycontainer \
  --source ./local-folder/ \
  --pattern "*.jpg" \           # Optional: only upload matching files
  --destination-path images/    # Optional: prefix in container

# ─────────────────────────────────────────────────────────
# DOWNLOAD BLOBS
# ─────────────────────────────────────────────────────────

# Download a single blob
az storage blob download \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./downloaded-myfile.txt

# Download all blobs matching a pattern
az storage blob download-batch \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --source mycontainer \
  --destination ./downloaded/ \
  --pattern "images/*.jpg"

# ─────────────────────────────────────────────────────────
# LIST AND MANAGE BLOBS
# ─────────────────────────────────────────────────────────

# List blobs in container
az storage blob list \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --output table

# List blobs with prefix (simulate folder)
az storage blob list \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --prefix "images/" \
  --output table

# Show blob properties
az storage blob show \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt

# Get blob URL
az storage blob url \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt \
  --output tsv

# Copy blob (within same account)
az storage blob copy start \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --destination-container archive \
  --destination-blob myfile-archive.txt \
  --source-container mycontainer \
  --source-blob myfile.txt

# Change blob access tier
az storage blob set-tier \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt \
  --tier Archive

# Delete a blob
az storage blob delete \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt

# Delete all blobs matching pattern
az storage blob delete-batch \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --source mycontainer \
  --pattern "logs/2022-*.log"

# ─────────────────────────────────────────────────────────
# SAS TOKEN FOR BLOB
# ─────────────────────────────────────────────────────────

# Generate SAS token for specific blob (read-only, expires in 1 hour)
az storage blob generate-sas \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \            # r=read, w=write, d=delete, c=create, a=add
  --expiry $(date -u -d '+1 hour' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --output tsv

# Generate SAS for container (write access, 24 hours)
az storage container generate-sas \
  --account-name $AZURE_STORAGE_ACCOUNT \
  --name mycontainer \
  --permissions rwdl \         # r=read, w=write, d=delete, l=list
  --expiry $(date -u -d '+1 day' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --output tsv
```

---

### SDK Code Examples

#### Python SDK (azure-storage-blob)

```python
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient
from azure.identity import DefaultAzureCredential
import os

# ─────────────────────────────────────────────────────────
# AUTHENTICATION — Use Managed Identity (preferred)
# ─────────────────────────────────────────────────────────
account_url = "https://mystorageaccount123.blob.core.windows.net"
credential = DefaultAzureCredential()   # Uses Managed Identity, env vars, etc.
blob_service_client = BlobServiceClient(account_url, credential=credential)

# ─────────────────────────────────────────────────────────
# CONTAINER OPERATIONS
# ─────────────────────────────────────────────────────────
container_name = "my-container"

# Create container
container_client = blob_service_client.create_container(container_name)

# Get container client
container_client = blob_service_client.get_container_client(container_name)

# List blobs
for blob in container_client.list_blobs():
    print(f"Name: {blob.name}, Size: {blob.size}, Modified: {blob.last_modified}")

# List blobs with prefix (simulate folder)
for blob in container_client.list_blobs(name_starts_with="images/"):
    print(blob.name)

# ─────────────────────────────────────────────────────────
# UPLOAD BLOBS
# ─────────────────────────────────────────────────────────
blob_client = blob_service_client.get_blob_client(
    container=container_name, blob="myfile.txt"
)

# Upload from file
with open("./local/myfile.txt", "rb") as data:
    blob_client.upload_blob(data, overwrite=True)

# Upload from string/bytes
content = "Hello, Azure Storage!"
blob_client.upload_blob(content.encode("utf-8"), overwrite=True)

# Upload with metadata and tier
with open("./large-file.zip", "rb") as data:
    blob_client.upload_blob(
        data,
        overwrite=True,
        metadata={"department": "engineering", "project": "analytics"},
        standard_blob_tier="Cool",
        max_concurrency=4   # Parallel upload for large files
    )

# ─────────────────────────────────────────────────────────
# DOWNLOAD BLOBS
# ─────────────────────────────────────────────────────────

# Download to file
with open("./downloaded.txt", "wb") as file:
    download_stream = blob_client.download_blob()
    file.write(download_stream.readall())

# Download as string
content = blob_client.download_blob().readall().decode("utf-8")
print(content)

# ─────────────────────────────────────────────────────────
# SAS TOKEN GENERATION (Python)
# ─────────────────────────────────────────────────────────
from azure.storage.blob import generate_blob_sas, BlobSasPermissions
from datetime import datetime, timedelta, timezone

sas_token = generate_blob_sas(
    account_name="mystorageaccount123",
    container_name=container_name,
    blob_name="myfile.txt",
    account_key=os.environ["STORAGE_KEY"],
    permission=BlobSasPermissions(read=True),
    expiry=datetime.now(timezone.utc) + timedelta(hours=1)
)

sas_url = f"https://mystorageaccount123.blob.core.windows.net/{container_name}/myfile.txt?{sas_token}"
print(f"SAS URL: {sas_url}")

# ─────────────────────────────────────────────────────────
# APPEND BLOB (log streaming)
# ─────────────────────────────────────────────────────────
from azure.storage.blob import AppendBlobClient

append_client = blob_service_client.get_blob_client(
    container=container_name, blob="app.log"
)
append_client.create_append_blob()

# Append log entries
log_entry = f"[{datetime.now().isoformat()}] INFO: Application started\n"
append_client.append_block(log_entry.encode("utf-8"))
```

#### C# SDK

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Identity;

// ─────────────────────────────────────────────────────────
// AUTHENTICATION — Use Managed Identity (preferred in Azure)
// ─────────────────────────────────────────────────────────
var accountUrl = new Uri("https://mystorageaccount123.blob.core.windows.net");
var credential = new DefaultAzureCredential();
var blobServiceClient = new BlobServiceClient(accountUrl, credential);

// ─────────────────────────────────────────────────────────
// UPLOAD A BLOB
// ─────────────────────────────────────────────────────────
var containerClient = blobServiceClient.GetContainerClient("my-container");
await containerClient.CreateIfNotExistsAsync();

var blobClient = containerClient.GetBlobClient("myfile.txt");

// Upload from local file
await using var uploadStream = File.OpenRead("./local/myfile.txt");
await blobClient.UploadAsync(uploadStream, overwrite: true);

// Upload with options (tier, metadata)
var uploadOptions = new BlobUploadOptions
{
    AccessTier = AccessTier.Cool,
    Metadata = new Dictionary<string, string>
    {
        { "department", "engineering" },
        { "created", DateTime.UtcNow.ToString("O") }
    }
};
await using var fileStream = File.OpenRead("./large-file.zip");
await blobClient.UploadAsync(fileStream, uploadOptions);

// ─────────────────────────────────────────────────────────
// DOWNLOAD A BLOB
// ─────────────────────────────────────────────────────────
var downloadResponse = await blobClient.DownloadAsync();
await using var downloadStream = File.Create("./downloaded.txt");
await downloadResponse.Value.Content.CopyToAsync(downloadStream);

// ─────────────────────────────────────────────────────────
// LIST BLOBS
// ─────────────────────────────────────────────────────────
await foreach (var blobItem in containerClient.GetBlobsAsync(prefix: "images/"))
{
    Console.WriteLine($"Name: {blobItem.Name}, Size: {blobItem.Properties.ContentLength}");
}

// ─────────────────────────────────────────────────────────
// DELETE A BLOB
// ─────────────────────────────────────────────────────────
await blobClient.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);
```

---

### Blob Storage Best Practices

| Area | Best Practice |
|---|---|
| **Security** | Disable public blob access; use SAS or Azure AD |
| **Security** | Use User Delegation SAS (RBAC-based) instead of Account Key SAS |
| **Security** | Set minimum TLS version to 1.2 |
| **Security** | Enable Microsoft Defender for Storage |
| **Data Protection** | Enable versioning + soft delete for production data |
| **Cost** | Use lifecycle management to auto-tier blobs to Cool/Archive |
| **Performance** | Use parallel uploads (`max_concurrency`) for large files |
| **Performance** | Use CDN in front of Blob Storage for global static content |
| **Naming** | Avoid too many blobs with the same prefix (creates hot partition) |
| **Access** | Use Managed Identity instead of connection strings in code |
| **Compliance** | Use immutable storage for WORM compliance requirements |

---

## 3. Azure Files

### What is Azure Files?

Azure Files provides fully managed cloud file shares accessible via **SMB (Server Message Block)** and **NFS (Network File System)** protocols. You can mount Azure file shares just like a traditional network drive — on Windows, Linux, and macOS.

**Use Azure Files when:**
- You need a lift-and-shift network share (replacing on-premises NAS/file server)
- Multiple VMs/apps need to share the same files simultaneously
- You need persistent storage for containers (AKS, ACI, App Service)
- You need to mount storage in code that uses file system APIs
- You're using Azure File Sync to hybrid-extend your on-premises file servers

**Blob vs Files decision:**

| Need | Use |
|---|---|
| Object/unstructured data (images, video, backups) | Blob Storage |
| Shared network drive (SMB/NFS mount) | Azure Files |
| Random read/write via file system APIs | Azure Files |
| Single app reading its own files | Blob Storage |
| Multiple VMs sharing the same files | Azure Files |

---

### SMB vs NFS Protocol

| Feature | SMB 3.0 | NFS 4.1 |
|---|---|---|
| OS Support | Windows, Linux, macOS | Linux only |
| Authentication | AD DS, Azure AD DS | No auth (IP-based) |
| Encryption | In-transit (AES-128/256) | In-transit (TLS) |
| Tier support | All tiers | Premium only |
| Use case | Windows apps, general sharing | Linux, POSIX workloads |
| Port | 445 | 2049 |

---

### File Share Tiers

| Tier | Performance | Account Type | Use Case | Pricing |
|---|---|---|---|---|
| **Transaction Optimized** | Standard | GPv2 | General purpose, dev/test | Storage + transactions |
| **Hot** | Standard | GPv2 | Frequently accessed shares | Higher storage, lower transactions |
| **Cool** | Standard | GPv2 | Infrequently accessed | Lower storage, higher transactions |
| **Premium** | SSD-backed | FileStorage | Databases, low-latency | Per provisioned GiB |

**Premium File Shares:** You provision capacity upfront (e.g., 100 GiB). IOPS and throughput scale with provisioned size. Single-digit ms latency.  
**Standard File Shares:** Pay-per-use. IOPS burst up to 10,000. Suitable for general file sharing.

---

### Azure File Sync

Azure File Sync transforms your on-premises Windows file server into a cache for Azure Files. Files are tiered (less-used files stored only in cloud, frequently-used files cached locally).

```
Azure File Sync Architecture:
┌──────────────────────────────────────────────────────┐
│  On-premises                    Azure Cloud           │
│                                                       │
│  ┌──────────────────┐          ┌──────────────────┐  │
│  │  Windows Server  │◄────────►│  Azure Files     │  │
│  │  File Server     │  Sync    │  Share           │  │
│  │                  │          │                  │  │
│  │  Hot files: local│          │  All files: cloud│  │
│  │  Cold files:     │          │  (authoritative  │  │
│  │  cloud pointer   │          │   source)        │  │
│  └──────────────────┘          └──────────────────┘  │
│                                                       │
│  Sync Group ────────────────────── Storage Sync Svc  │
└──────────────────────────────────────────────────────┘
```

Benefits:
- Unlimited cloud storage, local SSD performance for hot files
- Multi-site sync (multiple servers share same files)
- Disaster recovery (rebuild from cloud in minutes)
- Centralized backup (backup Azure Files share)

---

### Mount Azure Files on Different OSes

#### Windows (SMB)

```powershell
# Mount Azure Files share on Windows
# First, open port 445 in your network (common ISP block!)

$storageAccountName = "mystorageaccount123"
$shareName = "myshare"
$storageAccountKey = "<your-storage-key>"

# Mount as drive Z:
net use Z: \\$storageAccountName.file.core.windows.net\$shareName `
  /user:AZURE\$storageAccountName $storageAccountKey `
  /persistent:yes

# Or use cmdkey for credential persistence
cmdkey /add:"$storageAccountName.file.core.windows.net" `
  /user:"AZURE\$storageAccountName" `
  /pass:$storageAccountKey

net use Z: \\$storageAccountName.file.core.windows.net\$shareName /persistent:yes

# Verify mount
Get-PSDrive Z
```

#### Linux (SMB with cifs)

```bash
# Install CIFS utilities
sudo apt-get install cifs-utils -y    # Ubuntu/Debian
sudo yum install cifs-utils -y        # RHEL/CentOS

# Create mount point
sudo mkdir -p /mnt/myshare

# Mount Azure Files share
sudo mount -t cifs \
  //mystorageaccount123.file.core.windows.net/myshare \
  /mnt/myshare \
  -o vers=3.0,username=mystorageaccount123,password=<storage-key>,dir_mode=0777,file_mode=0777,serverino

# Mount permanently via /etc/fstab
echo "//mystorageaccount123.file.core.windows.net/myshare /mnt/myshare cifs nofail,vers=3.0,credentials=/etc/smbcredentials/mystorageaccount123.cred,dir_mode=0777,file_mode=0777,serverino" | sudo tee -a /etc/fstab

# Create credentials file
sudo mkdir /etc/smbcredentials
echo "username=mystorageaccount123" | sudo tee /etc/smbcredentials/mystorageaccount123.cred
echo "password=<storage-key>" | sudo tee -a /etc/smbcredentials/mystorageaccount123.cred
sudo chmod 600 /etc/smbcredentials/mystorageaccount123.cred
```

#### Linux (NFS — Premium shares only)

```bash
# Install NFS client
sudo apt-get install nfs-common -y

# Mount NFS Azure Files share
sudo mkdir -p /mnt/mynfsshare

sudo mount -t nfs \
  mystorageaccount123.file.core.windows.net:/mystorageaccount123/mynfsshare \
  /mnt/mynfsshare \
  -o vers=4,minorversion=1,sec=sys

# /etc/fstab entry
echo "mystorageaccount123.file.core.windows.net:/mystorageaccount123/mynfsshare /mnt/mynfsshare nfs defaults,vers=4,minorversion=1,sec=sys,nofail 0 0" | sudo tee -a /etc/fstab
```

#### macOS (SMB)

```bash
# Mount via Finder
# Go → Connect to Server → smb://mystorageaccount123.file.core.windows.net/myshare

# Mount via terminal
sudo mkdir -p /Volumes/myshare
sudo mount_smbfs //AZURE;mystorageaccount123:KEY@mystorageaccount123.file.core.windows.net/myshare /Volumes/myshare
```

---

### Complete CLI Commands for Azure Files

```bash
# ─────────────────────────────────────────────────────────
# CREATE AND MANAGE FILE SHARES
# ─────────────────────────────────────────────────────────

# Create a Standard file share
az storage share-rm create \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --name myshare \
  --quota 100 \              # Size in GiB
  --access-tier Hot

# Create a Premium file share (FileStorage account required)
az storage share-rm create \
  --resource-group myRG \
  --storage-account mypremiumfiles123 \
  --name mypremiumshare \
  --quota 500 \              # Provision capacity (IOPS scale with this)
  --enabled-protocols SMB

# Create NFS share (Premium, FileStorage, private endpoint required)
az storage share-rm create \
  --resource-group myRG \
  --storage-account mypremiumfiles123 \
  --name mynfsshare \
  --quota 1024 \
  --enabled-protocols NFS \
  --root-squash NoRootSquash

# List file shares
az storage share-rm list \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --output table

# Show share details
az storage share-rm show \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --name myshare

# Resize a share
az storage share-rm update \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --name myshare \
  --quota 200

# ─────────────────────────────────────────────────────────
# UPLOAD/DOWNLOAD FILES
# ─────────────────────────────────────────────────────────

# Upload a file
az storage file upload \
  --account-name mystorageaccount123 \
  --share-name myshare \
  --source ./localfile.txt \
  --path remote/path/localfile.txt

# Upload directory
az storage file upload-batch \
  --account-name mystorageaccount123 \
  --destination myshare \
  --source ./local-dir/

# Download a file
az storage file download \
  --account-name mystorageaccount123 \
  --share-name myshare \
  --path remote/path/localfile.txt \
  --dest ./downloaded.txt

# List files in directory
az storage file list \
  --account-name mystorageaccount123 \
  --share-name myshare \
  --path remote/path \
  --output table

# Create directory in share
az storage directory create \
  --account-name mystorageaccount123 \
  --share-name myshare \
  --name logs

# Delete a file
az storage file delete \
  --account-name mystorageaccount123 \
  --share-name myshare \
  --path remote/path/oldfile.txt

# Delete share
az storage share-rm delete \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --name myshare \
  --yes
```

### Azure Files Best Practices

| Area | Best Practice |
|---|---|
| **Security** | Require secure transfer (HTTPS/SMB encryption) |
| **Security** | Use Private Endpoints to restrict access to VNet |
| **Security** | Use Azure AD Kerberos authentication for identity-based access |
| **Performance** | Use Premium shares for latency-sensitive workloads (databases) |
| **Availability** | Use ZRS or GRS for production shares |
| **Backup** | Enable Azure Backup for Azure Files (daily snapshots) |
| **Cost** | Use Cool tier for archival file shares |
| **Sync** | Use Azure File Sync for large on-premises file server migrations |

---

## 4. Azure Queue Storage

### What is Queue Storage?

Azure Queue Storage is a simple, highly available message queue service. It decouples application components so they can scale and fail independently. A queue stores messages that are processed by consumers.

```
Queue Storage Pattern (Producer-Consumer):

Producer (Web App)           Queue              Consumer (Worker)
┌──────────────────┐        ┌──────┐           ┌──────────────────┐
│                  │        │ msg1 │           │                  │
│  Order created   │──────► │ msg2 │ ─────────►│  Process order   │
│                  │  enqueue│ msg3 │  dequeue  │  Send email      │
│                  │        │ msg4 │           │  Update DB        │
└──────────────────┘        └──────┘           └──────────────────┘
                                                  (auto-scales
                                                   independently)
```

**Key characteristics:**
- **Max message size:** 64 KB (use Blob Storage + queue with pointer for larger)
- **Max queue size:** Unlimited (effectively)
- **Message TTL:** Up to 7 days (default: 7 days)
- **Visibility timeout:** After dequeue, message is hidden for timeout period (prevents double-processing)
- **Throughput:** Thousands of messages per second
- **Ordering:** Best-effort FIFO (not guaranteed)

**Use Queue Storage when:**
- Decoupling microservices
- Background job processing
- Load leveling (absorb bursts, smooth processing rate)
- Simple task distribution

**Use Service Bus instead when:**
- You need guaranteed ordering (FIFO with sessions)
- You need message size > 64 KB (Service Bus: 256 KB - 100 MB)
- You need competing consumers with ordering guarantees
- You need dead-letter queues with rich retry logic
- You need topics/subscriptions (pub/sub pattern)

---

### Visibility Timeout and Poison Messages

```
Message Processing Flow:
                    ┌─────────────────────────────────────────┐
Queue: [msg1][msg2] │                                         │
                    │  1. Consumer dequeues msg1               │
                    │  2. msg1 becomes invisible (timeout=30s) │
                    │  3. Consumer processes msg1              │
                    │  4a. SUCCESS: Consumer deletes msg1      │
                    │  4b. FAILURE: msg1 reappears after 30s   │
                    │  5. Retry count increments               │
                    │  6. After N retries → poison queue       │
                    └─────────────────────────────────────────┘

Poison Queue Pattern:
my-queue              →  my-queue-poison
[failed-msg (5x)]  →  [failed-msg moved here for investigation]
```

---

### Complete CLI Commands for Queue Storage

```bash
# ─────────────────────────────────────────────────────────
# CREATE AND MANAGE QUEUES
# ─────────────────────────────────────────────────────────

# Create a queue
az storage queue create \
  --account-name mystorageaccount123 \
  --name my-queue

# List queues
az storage queue list \
  --account-name mystorageaccount123 \
  --output table

# Show queue metadata (approximate message count)
az storage queue metadata show \
  --account-name mystorageaccount123 \
  --name my-queue

# Delete a queue
az storage queue delete \
  --account-name mystorageaccount123 \
  --name my-queue

# ─────────────────────────────────────────────────────────
# SEND AND RECEIVE MESSAGES
# ─────────────────────────────────────────────────────────

# Enqueue (send) a message
az storage message put \
  --account-name mystorageaccount123 \
  --queue-name my-queue \
  --content "Process order 12345" \
  --time-to-live 3600    # TTL in seconds (1 hour)

# Peek at messages (without dequeuing — non-destructive)
az storage message peek \
  --account-name mystorageaccount123 \
  --queue-name my-queue \
  --num-messages 10 \
  --output table

# Dequeue (get and hide) messages
az storage message get \
  --account-name mystorageaccount123 \
  --queue-name my-queue \
  --num-messages 5 \
  --visibility-timeout 30 \    # Hidden for 30 seconds
  --output json

# Delete a message (after processing)
# First dequeue returns: id and popReceipt
az storage message delete \
  --account-name mystorageaccount123 \
  --queue-name my-queue \
  --id "message-id-from-dequeue" \
  --pop-receipt "pop-receipt-from-dequeue"

# Clear all messages from queue
az storage message clear \
  --account-name mystorageaccount123 \
  --queue-name my-queue
```

#### Python Queue Example

```python
from azure.storage.queue import QueueClient
from azure.identity import DefaultAzureCredential
import base64
import json

account_url = "https://mystorageaccount123.queue.core.windows.net"
credential = DefaultAzureCredential()
queue_client = QueueClient(account_url, queue_name="my-queue", credential=credential)

# ─────────────────────────────────────────────────────────
# PRODUCER: Send messages
# ─────────────────────────────────────────────────────────
message = {
    "orderId": "ORD-12345",
    "customer": "Alice Smith",
    "amount": 99.99
}
# Queue messages are always strings; encode JSON
queue_client.send_message(json.dumps(message), time_to_live=3600)

# ─────────────────────────────────────────────────────────
# CONSUMER: Process messages
# ─────────────────────────────────────────────────────────
MAX_RETRIES = 5

while True:
    messages = queue_client.receive_messages(
        max_messages=10,
        visibility_timeout=60  # 60 seconds to process
    )
    
    for msg in messages:
        try:
            body = json.loads(msg.content)
            
            # Poison message check
            if msg.dequeue_count > MAX_RETRIES:
                print(f"Poison message detected: {msg.id}, moving to DLQ")
                # Send to poison queue manually
                poison_client = QueueClient(account_url, "my-queue-poison", credential=credential)
                poison_client.send_message(msg.content)
                queue_client.delete_message(msg)
                continue
            
            # Process the message
            process_order(body)
            
            # Delete on success
            queue_client.delete_message(msg)
            
        except Exception as e:
            print(f"Error processing message {msg.id}: {e}")
            # Don't delete — it will reappear after visibility timeout
```

---

## 5. Azure Table Storage

### What is Azure Table Storage?

Azure Table Storage is a serverless NoSQL key-value store designed for storing large amounts of structured, non-relational data. Data is organized in tables with rows (entities), each identified by a **PartitionKey** + **RowKey** combination.

```
Table Storage Structure:
Table: Orders
┌──────────────────────┬──────────────────────┬──────────────────────────────┐
│ PartitionKey         │ RowKey               │ Properties                   │
├──────────────────────┼──────────────────────┼──────────────────────────────┤
│ "customer-001"       │ "order-2024-001"     │ Amount: 99.99, Status: Paid  │
│ "customer-001"       │ "order-2024-002"     │ Amount: 149.99, Status: Ship │
│ "customer-002"       │ "order-2024-003"     │ Amount: 29.99, Status: Paid  │
│ "customer-003"       │ "order-2024-004"     │ Amount: 499.99, Status: Pend │
└──────────────────────┴──────────────────────┴──────────────────────────────┘

PartitionKey = "customer-001" → All customer-001 rows on same partition (fast queries)
RowKey = "order-2024-001"    → Unique within partition (unique per entity)
```

**Table Storage vs CosmosDB Table API:**

| Feature | Azure Table Storage | CosmosDB Table API |
|---|---|---|
| SLA | 99.9% | 99.999% |
| Global distribution | No | Yes (multi-region) |
| Consistency | Eventual | 5 consistency levels |
| Indexing | PK + RK only | Secondary indexes |
| Throughput | Shared | Provisioned or serverless |
| Max entity size | 1 MB | 2 MB |
| Price | Very cheap | More expensive |
| Best for | Simple, cheap NoSQL | Global, high-SLA NoSQL |

**Use Table Storage when:**
- You need a simple, cheap NoSQL store
- Global distribution and high SLA are not required
- Data access patterns are simple (PK + RK lookups)
- You need device telemetry, user preferences, metadata

---

### Partition Key and Row Key Design

Good PartitionKey design is critical for performance:

```
BAD PartitionKey designs:
- Single partition key for all data → hot partition, throttling
  PartitionKey = "all" → all 10M records on one partition

- Random UUIDs as partition key → queries scan all partitions
  PartitionKey = "550e8400-e29b..." → no locality, slow queries

GOOD PartitionKey designs:
- Customer ID: PartitionKey = "customer-{customerId}"
  → all data for a customer in one partition, fast per-customer queries

- Time-bucketed: PartitionKey = "2024-01-15"
  → all records for a day in one partition

- Region-based: PartitionKey = "us-east"
  → all records for a region together

GOOD RowKey designs:
- Reverse-chronological timestamp: RowKey = (MaxTick - DateTime.UtcNow.Ticks).ToString("d19")
  → most recent records first when listed (useful for logs)

- Sequential ID: RowKey = orderId
  → unique per partition, fast point lookups
```

---

### Complete CLI Commands for Table Storage

```bash
# ─────────────────────────────────────────────────────────
# CREATE AND MANAGE TABLES
# ─────────────────────────────────────────────────────────

# Create a table
az storage table create \
  --account-name mystorageaccount123 \
  --name orders

# List tables
az storage table list \
  --account-name mystorageaccount123 \
  --output table

# Delete a table
az storage table delete \
  --account-name mystorageaccount123 \
  --name orders \
  --yes

# ─────────────────────────────────────────────────────────
# ENTITIES (ROWS)
# ─────────────────────────────────────────────────────────

# Insert an entity
az storage entity insert \
  --account-name mystorageaccount123 \
  --table-name orders \
  --entity \
    PartitionKey=customer-001 \
    RowKey=order-2024-001 \
    Amount=99.99 \
    Status=Paid \
    CustomerName="Alice Smith"

# Show a specific entity (point lookup — fastest)
az storage entity show \
  --account-name mystorageaccount123 \
  --table-name orders \
  --partition-key customer-001 \
  --row-key order-2024-001

# Query entities (filter)
az storage entity query \
  --account-name mystorageaccount123 \
  --table-name orders \
  --filter "PartitionKey eq 'customer-001'" \
  --select "RowKey,Amount,Status" \
  --output table

# Update entity (merge)
az storage entity merge \
  --account-name mystorageaccount123 \
  --table-name orders \
  --entity \
    PartitionKey=customer-001 \
    RowKey=order-2024-001 \
    Status=Shipped

# Replace entity (full overwrite)
az storage entity replace \
  --account-name mystorageaccount123 \
  --table-name orders \
  --entity \
    PartitionKey=customer-001 \
    RowKey=order-2024-001 \
    Amount=99.99 \
    Status=Delivered \
    CustomerName="Alice Smith"

# Delete an entity
az storage entity delete \
  --account-name mystorageaccount123 \
  --table-name orders \
  --partition-key customer-001 \
  --row-key order-2024-001
```

---

## 6. Azure Data Lake Storage Gen2

### What is ADLS Gen2?

Azure Data Lake Storage Gen2 (ADLS Gen2) combines Azure Blob Storage with a **Hierarchical Namespace (HNS)**. It adds real folders (directory operations are atomic and efficient), POSIX-style ACLs, and Hadoop-compatible APIs — making it ideal for big data analytics.

```
ADLS Gen2 vs Standard Blob Storage:

Standard Blob Storage:          ADLS Gen2 (HNS enabled):
"Flat" namespace:               "Hierarchical" namespace:
Container: data                 Container (filesystem): data
├── a/b/c/file1.csv   ─────►   ├── a/ (real directory)
├── a/b/c/file2.csv             │   └── b/ (real directory)
├── a/b/d/file3.csv             │       ├── c/ (real directory)
└── e/file4.csv                 │       │   ├── file1.csv
                                │       │   └── file2.csv
(folders simulated             │       └── d/
 by blob name prefix)          │           └── file3.csv
                                └── e/
                                    └── file4.csv

Operations:
- Rename directory: O(1) in HNS  →  O(n) in flat namespace (n=file count)
- Delete directory: O(1) in HNS  →  O(n) in flat namespace
- List directory: Efficient       →  Requires prefix scan
```

**Use ADLS Gen2 when:**
- Big data analytics (Azure Synapse, Databricks, HDInsight)
- Data lake storage (raw, curated, and aggregated layers)
- Machine learning data storage
- ETL pipelines with hierarchical data organization
- Hadoop-compatible workloads

---

### Hierarchical Namespace and ACLs

With HNS, you can set POSIX-style ACLs on individual files and directories — much more granular than Blob Storage RBAC.

```
ADLS Gen2 ACL Structure:
Directory: /data/finance/
├── Access ACL:
│   ├── User (owner): rwx
│   ├── Group (finance-team): r-x
│   ├── Other: ---
│   └── Named user (data-analyst@company.com): r--
└── Default ACL (inherited by new children):
    ├── User (owner): rwx
    ├── Group (finance-team): r-x
    └── Other: ---
```

ACL permissions:
- **r** (read): List directory / read file
- **w** (write): Create files in directory / write file
- **x** (execute): Enter/traverse directory (required to navigate)

```bash
# Enable Hierarchical Namespace on new storage account
az storage account create \
  --name mydatalake123 \
  --resource-group myRG \
  --location eastus \
  --sku Standard_RAGRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true    # This enables ADLS Gen2

# Create filesystem (top-level container in ADLS Gen2)
az storage fs create \
  --name raw \
  --account-name mydatalake123 \
  --auth-mode login

# Create directory
az storage fs directory create \
  --name finance/2024/q1 \
  --file-system raw \
  --account-name mydatalake123

# List directory
az storage fs directory list \
  --name finance \
  --file-system raw \
  --account-name mydatalake123 \
  --output table

# Upload a file
az storage fs file upload \
  --source ./data.csv \
  --path finance/2024/q1/report.csv \
  --file-system raw \
  --account-name mydatalake123

# Set ACLs on a directory
az storage fs access set \
  --acl "user::rwx,group::r-x,other::---,user:00000000-0000-0000-0000-000000000001:r--" \
  --path finance/2024 \
  --file-system raw \
  --account-name mydatalake123

# Show ACLs
az storage fs access show \
  --path finance/2024 \
  --file-system raw \
  --account-name mydatalake123

# Update ACLs recursively on directory tree
az storage fs access update-recursive \
  --acl "user:00000000-0000-0000-0000-000000000001:r--" \
  --path finance \
  --file-system raw \
  --account-name mydatalake123

# Move/rename (atomic, O(1) with HNS)
az storage fs file move \
  --path finance/2024/q1/old-name.csv \
  --new-path finance/2024/q1/new-name.csv \
  --file-system raw \
  --account-name mydatalake123
```

---

### Integration with Azure Analytics Services

```
ADLS Gen2 as Data Lake Foundation:

Raw Data Sources               ADLS Gen2 Zones           Analytics Engines
                                                          
CSV, JSON, Parquet ──────────► /raw/                ────► Azure Synapse Analytics
IoT telemetry       ──────────► /curated/            ────► Azure Databricks
Logs, events        ──────────► /aggregated/         ────► Azure HDInsight
Database exports    ──────────► /ml-features/        ────► Azure Machine Learning
                                                          ► Power BI (via Synapse)
```

```bash
# Grant Azure Synapse access to ADLS Gen2 via Managed Identity
# First get Synapse workspace's Managed Identity Object ID
SYNAPSE_MI_ID=$(az synapse workspace show \
  --name mysynapse \
  --resource-group myRG \
  --query identity.principalId --output tsv)

# Grant Storage Blob Data Contributor role
az role assignment create \
  --assignee "$SYNAPSE_MI_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mydatalake123"
```

---

## 7. Azure Managed Disks

### What is a Managed Disk?

A Managed Disk is a virtual hard disk (VHD) that Azure manages for you. You don't deal with storage accounts, VHD files, or storage IOPS limits. Azure handles placement, replication, and durability.

Managed Disks are used as:
- **OS disks:** The boot disk for a VM
- **Data disks:** Additional disks attached to a VM for data

---

### Disk Types

| Type | Max IOPS | Max Throughput | Max Size | Latency | Use Case |
|---|---|---|---|---|---|
| **Ultra Disk** | 160,000 | 2,000 MB/s | 65,536 GiB | Sub-ms | Mission-critical DBs (SAP HANA, Oracle) |
| **Premium SSD v2** | 80,000 | 1,200 MB/s | 65,536 GiB | ~1 ms | Flexible production DBs, variable IOPS |
| **Premium SSD** | 20,000 | 900 MB/s | 32,767 GiB | ~5 ms | Production workloads, SQL Server |
| **Standard SSD** | 6,000 | 750 MB/s | 32,767 GiB | ~10 ms | Web servers, dev/test, light workloads |
| **Standard HDD** | 2,000 | 500 MB/s | 32,767 GiB | ~20 ms | Backups, infrequent access, dev/test |

```
IOPS Comparison:
Ultra SSD   ████████████████████████████████████████ 160,000
Prem SSD v2 ████████████████████ 80,000
Prem SSD    █████ 20,000
Std SSD     ██ 6,000
Std HDD     █ 2,000

Ultra Disk = 80× the IOPS of Standard HDD
```

---

### Disk Sizes and IOPS (Premium SSD)

Premium SSD IOPS and throughput scale with disk size:

| Disk Size (GiB) | IOPS | Throughput (MB/s) | Monthly Cost (approx) |
|---|---|---|---|
| P1 (4 GiB) | 120 | 25 | $0.61 |
| P6 (64 GiB) | 240 | 50 | $9.60 |
| P10 (128 GiB) | 500 | 100 | $19.71 |
| P20 (512 GiB) | 2,300 | 150 | $73.22 |
| P30 (1,024 GiB) | 5,000 | 200 | $135.17 |
| P40 (2,048 GiB) | 7,500 | 250 | $255.64 |
| P50 (4,096 GiB) | 7,500 | 250 | $511.28 |
| P60 (8,192 GiB) | 16,000 | 500 | $1,027 |
| P80 (32,767 GiB) | 20,000 | 900 | $4,132 |

**Premium SSD bursting:** Disks ≤ 512 GiB can burst up to 3,500 IOPS and 170 MB/s for up to 30 minutes per day.

---

### Snapshots and Disk Backup

```bash
# ─────────────────────────────────────────────────────────
# SNAPSHOTS
# ─────────────────────────────────────────────────────────

# Get OS disk resource ID of a VM
DISK_ID=$(az vm show \
  --resource-group myRG \
  --name myVM \
  --query storageProfile.osDisk.managedDisk.id \
  --output tsv)

# Create incremental snapshot (recommended — only stores changes)
az snapshot create \
  --resource-group myRG \
  --name mySnapshot-$(date +%Y%m%d) \
  --source "$DISK_ID" \
  --incremental true \
  --sku Standard_ZRS \    # Zone-redundant snapshot storage
  --location eastus

# Create full snapshot
az snapshot create \
  --resource-group myRG \
  --name myFullSnapshot \
  --source "$DISK_ID" \
  --incremental false

# List snapshots
az snapshot list \
  --resource-group myRG \
  --output table

# Create a new disk from snapshot (for restore)
az disk create \
  --resource-group myRG \
  --name restoredDisk \
  --source /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Compute/snapshots/mySnapshot-20240115 \
  --sku Premium_LRS

# Swap OS disk on a VM (restore scenario)
az vm update \
  --resource-group myRG \
  --name myVM \
  --os-disk restoredDisk

# Delete snapshot
az snapshot delete \
  --resource-group myRG \
  --name mySnapshot-20240115 \
  --yes

# ─────────────────────────────────────────────────────────
# MANAGED DISK OPERATIONS
# ─────────────────────────────────────────────────────────

# Create a standalone managed disk
az disk create \
  --resource-group myRG \
  --name myDataDisk \
  --size-gb 256 \
  --sku Premium_LRS \
  --location eastus \
  --zone 1 \               # Zone-pinned disk
  --encryption-type EncryptionAtRestWithPlatformKey

# Resize a disk (can only grow, not shrink; VM must be deallocated for OS disk)
az disk update \
  --resource-group myRG \
  --name myDataDisk \
  --size-gb 512

# Change disk SKU
az disk update \
  --resource-group myRG \
  --name myDataDisk \
  --sku StandardSSD_LRS

# Convert unmanaged disk to managed (legacy migration)
az vm convert \
  --resource-group myRG \
  --name myVM

# Grant SAS URI for downloading disk (offline export)
az disk grant-access \
  --resource-group myRG \
  --name myDataDisk \
  --duration-in-seconds 86400 \   # 24 hours
  --access-level Read

# Revoke SAS access
az disk revoke-access \
  --resource-group myRG \
  --name myDataDisk
```

---

### Disk Encryption

Azure provides multiple layers of disk encryption:

| Encryption Type | What It Encrypts | Key Management | Notes |
|---|---|---|---|
| **SSE with PMK** | At-rest on storage | Microsoft managed | Default; automatic; no action needed |
| **SSE with CMK** | At-rest on storage | Customer-managed (Key Vault) | Bring your own key; meet compliance |
| **Azure Disk Encryption (ADE)** | OS + data disk volumes | Customer-managed | BitLocker (Win) / DM-Crypt (Linux) |
| **Encryption at host** | Temp disk + cache | Platform or CMK | Extends SSE to temp disk |

```bash
# ─────────────────────────────────────────────────────────
# SERVER-SIDE ENCRYPTION WITH CUSTOMER-MANAGED KEY (CMK)
# ─────────────────────────────────────────────────────────

# Create Key Vault with soft delete (required for disk encryption)
az keyvault create \
  --name myDiskEncryptionKV \
  --resource-group myRG \
  --location eastus \
  --enable-soft-delete true \
  --enable-purge-protection true \
  --sku premium

# Create encryption key
az keyvault key create \
  --vault-name myDiskEncryptionKV \
  --name myDiskKey \
  --kty RSA \
  --size 4096

# Get key URL
KEY_URL=$(az keyvault key show \
  --vault-name myDiskEncryptionKV \
  --name myDiskKey \
  --query key.kid --output tsv)

# Create Disk Encryption Set
az disk-encryption-set create \
  --name myDES \
  --resource-group myRG \
  --location eastus \
  --key-url "$KEY_URL" \
  --source-vault myDiskEncryptionKV \
  --encryption-type EncryptionAtRestWithCustomerKey

# Get DES principal ID and grant Key Vault access
DES_PRINCIPAL=$(az disk-encryption-set show \
  --name myDES \
  --resource-group myRG \
  --query identity.principalId --output tsv)

az keyvault set-policy \
  --name myDiskEncryptionKV \
  --object-id "$DES_PRINCIPAL" \
  --key-permissions wrapKey unwrapKey get

# Create disk using CMK encryption
DES_ID=$(az disk-encryption-set show \
  --name myDES \
  --resource-group myRG \
  --query id --output tsv)

az disk create \
  --resource-group myRG \
  --name myEncryptedDisk \
  --size-gb 256 \
  --sku Premium_LRS \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set "$DES_ID"

# ─────────────────────────────────────────────────────────
# AZURE DISK ENCRYPTION (BitLocker/DM-Crypt)
# ─────────────────────────────────────────────────────────

# Enable ADE on a Linux VM
az vm encryption enable \
  --resource-group myRG \
  --name myVM \
  --disk-encryption-keyvault myDiskEncryptionKV \
  --key-encryption-key myDiskKey \
  --volume-type All    # OS, Data, or All

# Check encryption status
az vm encryption show \
  --resource-group myRG \
  --name myVM \
  --output json
```

---

## 8. Storage Security

### Microsoft Entra ID Authentication

Azure Storage supports Azure AD (Microsoft Entra ID) authentication for Blob, Queue, Table, and Files (via Azure AD Kerberos for SMB). This is more secure than using account keys.

```
Authentication Methods (Most Secure → Least):
1. Managed Identity (Azure resources)     ← Best practice for Azure workloads
2. Service Principal (apps/scripts)       ← Best for external apps
3. User Delegation SAS                    ← Best for user-facing temporary access
4. Account-Level SAS                      ← Use when AD not possible
5. Account Keys                           ← Avoid; treat like root password
```

```bash
# Assign Storage Blob Data Contributor to a user
az role assignment create \
  --assignee user@company.com \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount123/blobServices/default/containers/mycontainer"

# Available storage RBAC roles:
# Storage Blob Data Owner         — Full access to blobs (including POSIX ACL management)
# Storage Blob Data Contributor   — Read, write, delete blobs
# Storage Blob Data Reader        — Read-only blobs
# Storage Queue Data Contributor  — Read, write, delete queue messages
# Storage Table Data Contributor  — Read, write, delete table entities
# Storage File Data SMB Share Contributor  — Mount and read/write file shares

# Use Azure AD auth (not account key) in CLI commands
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --auth-mode login    # Uses your logged-in identity

# Assign Managed Identity to a VM and grant storage access
VM_PRINCIPAL=$(az vm show \
  --resource-group myRG \
  --name myVM \
  --query identity.principalId --output tsv)

az role assignment create \
  --assignee "$VM_PRINCIPAL" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount123"
```

---

### Shared Access Signatures (SAS)

A SAS token is a signed URL that grants scoped, time-limited access to a storage resource without sharing your account key.

#### Types of SAS

| SAS Type | Signed With | Revokable | Best For |
|---|---|---|---|
| **Account SAS** | Account key | Only by rotating key | Broad access across services |
| **Service SAS** | Account key | Only by rotating key | Single service (blob, file, queue, table) |
| **User Delegation SAS** | Azure AD credentials | Yes (revoke Azure AD token) | Recommended; most secure |

#### SAS Permission Codes

| Code | Permission | Applies To |
|---|---|---|
| r | Read | Blob, File, Queue, Table |
| w | Write | Blob, File, Table |
| d | Delete | Blob, File, Queue, Table |
| l | List | Container, Share, Queue |
| a | Add | Blob, Queue, Table |
| c | Create | Blob, Container |
| u | Update | Blob, Table |
| p | Process | Queue (dequeue) |
| t | Tag | Blob |

```bash
# ─────────────────────────────────────────────────────────
# ACCOUNT SAS
# ─────────────────────────────────────────────────────────

# Generate Account SAS (read/write blobs and files, 24 hours)
az storage account generate-sas \
  --account-name mystorageaccount123 \
  --services bf \              # b=blob, f=file, q=queue, t=table
  --resource-types sco \       # s=service, c=container, o=object
  --permissions rwdlac \
  --expiry $(date -u -d '+1 day' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --output tsv

# ─────────────────────────────────────────────────────────
# SERVICE SAS FOR BLOB
# ─────────────────────────────────────────────────────────

# SAS for a specific blob (read-only, 1 hour)
az storage blob generate-sas \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry $(date -u -d '+1 hour' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --output tsv

# ─────────────────────────────────────────────────────────
# USER DELEGATION SAS (requires Azure AD login)
# ─────────────────────────────────────────────────────────

# Generate User Delegation SAS (most secure — no account key used)
az storage blob generate-sas \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry $(date -u -d '+1 hour' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --auth-mode login \          # Uses Azure AD login instead of account key
  --as-user \                  # Creates user delegation SAS
  --output tsv
```

---

### Customer-Managed Keys (CMK)

By default, Azure encrypts all storage data at rest using **Platform-Managed Keys (PMK)** — Microsoft generates and manages the keys. With **Customer-Managed Keys (CMK)**, you control the keys in Azure Key Vault.

```
CMK Architecture:
Your Key Vault                   Azure Storage
┌────────────────────┐           ┌──────────────────────┐
│  myDiskKey         │◄─────────►│  Storage Account     │
│  (RSA 2048/4096)   │  Wrap/    │  Data encrypted with │
│                    │  Unwrap   │  DEK, DEK wrapped    │
│  You control:      │  Key      │  with your CMK       │
│  - Rotation        │           │                      │
│  - Revocation      │           │  If key revoked:     │
│  - Expiry          │           │  Data inaccessible!  │
└────────────────────┘           └──────────────────────┘
```

```bash
# Enable CMK on an existing storage account
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault https://myvault.vault.azure.net/ \
  --encryption-key-name myKey \
  --encryption-key-version <key-version>

# Enable auto-rotation (always use latest key version)
az storage account update \
  --name mystorageaccount123 \
  --resource-group myRG \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault https://myvault.vault.azure.net/ \
  --encryption-key-name myKey \
  --encryption-key-version ""    # Empty = always use latest version
```

---

### Microsoft Defender for Storage

Microsoft Defender for Storage monitors your storage accounts for anomalous activity, malware uploads, and data exfiltration attempts.

**Threat detections include:**
- Access from Tor exit nodes
- Unusual data extraction (volume or location anomaly)
- Anonymous access to previously non-public blobs
- Malware hash reputation (scans uploaded blobs against Microsoft Threat Intelligence)
- Sensitive data threat detection

```bash
# Enable Defender for Storage at subscription level (covers all storage accounts)
az security pricing create \
  --name StorageAccounts \
  --tier Standard

# Enable on specific storage account
az security atp storage update \
  --resource-group myRG \
  --storage-account mystorageaccount123 \
  --is-enabled true

# View security alerts
az security alert list \
  --resource-group myRG \
  --output table
```

---

## 9. AzCopy Complete Reference

### What is AzCopy?

AzCopy is a command-line utility designed for high-performance copying of data to/from Azure Storage. It's optimized for large-scale transfers — far faster than the Azure CLI for bulk operations.

```
When to use AzCopy vs Azure CLI:
- Single file upload: az storage blob upload is fine
- Millions of files / TBs of data: USE AZCOPY
- Cross-account/cross-region copy: USE AZCOPY
- Sync directories: USE AZCOPY
- Resume interrupted transfers: USE AZCOPY
```

---

### Installation

```bash
# ─────────────────────────────────────────────────────────
# LINUX
# ─────────────────────────────────────────────────────────
# Download and install
wget https://aka.ms/downloadazcopy-v10-linux
tar -xvf downloadazcopy-v10-linux
sudo mv azcopy_linux_amd64_*/azcopy /usr/local/bin/azcopy
chmod +x /usr/local/bin/azcopy
azcopy --version

# ─────────────────────────────────────────────────────────
# WINDOWS
# ─────────────────────────────────────────────────────────
# Via PowerShell (download and add to PATH)
Invoke-WebRequest -Uri "https://aka.ms/downloadazcopy-v10-windows" -OutFile AzCopy.zip
Expand-Archive ./AzCopy.zip ./AzCopy
$Env:PATH += ";$(Get-ChildItem ./AzCopy -Recurse -File azcopy.exe).DirectoryName"

# ─────────────────────────────────────────────────────────
# MACOS
# ─────────────────────────────────────────────────────────
brew install azcopy
# or manually:
wget https://aka.ms/downloadazcopy-v10-mac
tar -xvf downloadazcopy-v10-mac
sudo mv azcopy_darwin_amd64_*/azcopy /usr/local/bin/azcopy
```

---

### Authentication

#### Method 1: Azure AD Login (Recommended)

```bash
# Login interactively (opens browser)
azcopy login

# Login with Service Principal (for CI/CD)
export AZCOPY_SPA_CLIENT_SECRET=<client-secret>
azcopy login \
  --service-principal \
  --application-id <app-id> \
  --tenant-id <tenant-id>

# Login with Managed Identity (on Azure VMs, ACI, AKS)
azcopy login --identity
# or for user-assigned managed identity:
azcopy login --identity --identity-client-id <client-id>

# Check login status
azcopy login status

# Logout
azcopy logout
```

#### Method 2: SAS Token

```bash
# Append SAS token to the URL
# Format: https://<account>.blob.core.windows.net/<container>?<SAS-token>

SAS_TOKEN=$(az storage container generate-sas \
  --account-name mystorageaccount123 \
  --name mycontainer \
  --permissions rwdl \
  --expiry $(date -u -d '+1 day' +%Y-%m-%dT%H:%MZ) \
  --https-only \
  --output tsv)

CONTAINER_URL="https://mystorageaccount123.blob.core.windows.net/mycontainer?${SAS_TOKEN}"
```

---

### Copy Commands

#### Local to Blob (Upload)

```bash
# Upload a single file
azcopy copy \
  './localfile.txt' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/localfile.txt' \
  --overwrite true

# Upload with SAS token
azcopy copy \
  './localfile.txt' \
  "https://mystorageaccount123.blob.core.windows.net/mycontainer/localfile.txt?${SAS_TOKEN}"

# Upload an entire directory (recursively)
azcopy copy \
  './local-folder/' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  --recursive \
  --overwrite true

# Upload with pattern matching (only *.csv files)
azcopy copy \
  './data/*.csv' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/csv-files/' \
  --recursive

# Upload and set access tier
azcopy copy \
  './archive.zip' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/archive.zip' \
  --blob-type BlockBlob \
  --block-blob-tier Cool

# Upload with MD5 verification
azcopy copy \
  './important-file.zip' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/important-file.zip' \
  --check-md5 FailIfDifferentOrMissing
```

#### Blob to Local (Download)

```bash
# Download a single blob
azcopy copy \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/myfile.txt' \
  './downloaded-myfile.txt'

# Download entire container
azcopy copy \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  './downloaded-container/' \
  --recursive

# Download blobs matching pattern
azcopy copy \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/*' \
  './local-dir/' \
  --include-pattern '*.jpg;*.png' \
  --recursive

# Download blobs modified after date
azcopy copy \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  './recent-files/' \
  --include-after '2024-01-01T00:00:00Z' \
  --recursive
```

#### Blob to Blob (Cross-Account / Server-Side Copy)

Server-side copy: data flows between Azure datacenters without going through your machine. Extremely fast for large datasets.

```bash
# Copy blob within same account
azcopy copy \
  'https://mystorageaccount123.blob.core.windows.net/source/file.txt' \
  'https://mystorageaccount123.blob.core.windows.net/destination/file.txt'

# Copy between accounts (both must be authenticated or use SAS)
azcopy copy \
  "https://account1.blob.core.windows.net/container1/file.txt?${SAS_SOURCE}" \
  "https://account2.blob.core.windows.net/container2/file.txt?${SAS_DEST}"

# Copy entire container (cross-account, server-side)
azcopy copy \
  "https://source-account.blob.core.windows.net/my-container?${SAS_SOURCE}" \
  "https://dest-account.blob.core.windows.net/my-container?${SAS_DEST}" \
  --recursive

# Copy between regions (server-side, very fast)
azcopy copy \
  "https://eastus-account.blob.core.windows.net/mycontainer" \
  "https://westeurope-account.blob.core.windows.net/mycontainer" \
  --recursive \
  --s2s-preserve-access-tier false    # Reset tier on destination

# Copy from AWS S3 to Azure Blob (migration)
azcopy copy \
  "https://s3.amazonaws.com/my-s3-bucket/my-folder/" \
  "https://mystorageaccount123.blob.core.windows.net/mycontainer/from-s3/" \
  --recursive \
  --from-to S3Blob
```

---

### Sync Command

The sync command copies only files that are new or changed (like rsync). Optionally deletes files in destination that don't exist in source.

```bash
# Sync local directory to blob container (upload new/changed files)
azcopy sync \
  './local-folder/' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  --recursive \
  --delete-destination false   # Don't delete extra files in destination (default)

# Sync and delete files in destination not in source
azcopy sync \
  './local-folder/' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  --recursive \
  --delete-destination true    # Mirror mode — destination matches source exactly

# Sync blob container to local (download changed files)
azcopy sync \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  './local-folder/' \
  --recursive

# Sync between two blob accounts
azcopy sync \
  "https://source.blob.core.windows.net/container?${SAS_SOURCE}" \
  "https://dest.blob.core.windows.net/container?${SAS_DEST}" \
  --recursive

# Sync with exclude patterns
azcopy sync \
  './local-folder/' \
  'https://mystorageaccount123.blob.core.windows.net/mycontainer/' \
  --recursive \
  --exclude-pattern '*.tmp;*.log;.DS_Store' \
  --exclude-path 'node_modules;.git'
```

---

### Jobs Management

AzCopy creates a job for each transfer operation. Jobs can be listed, resumed, and removed.

```bash
# List all AzCopy jobs
azcopy jobs list

# Show status of a specific job
azcopy jobs show <job-id>

# Resume a failed or interrupted job
azcopy jobs resume <job-id>

# Resume with new SAS token (when original expired)
azcopy jobs resume <job-id> \
  --source-sas "?<new-source-sas>" \
  --destination-sas "?<new-dest-sas>"

# Remove completed job records
azcopy jobs remove <job-id>
azcopy jobs clean    # Remove all completed jobs

# Show detailed transfer log
azcopy jobs show <job-id> --with-status Failed    # Show only failed transfers
```

---

### AzCopy Performance Tuning

```bash
# Set concurrency (default: auto-detects based on CPU cores)
azcopy copy \
  './data/' \
  'https://mystorageaccount123.blob.core.windows.net/container/' \
  --recursive \
  --cap-mbps 500 \                   # Cap bandwidth at 500 Mbps
  --block-size-mb 128 \              # Block size for upload (default: auto)
  --parallel-level 32                # Number of concurrent operations

# Use environment variable for concurrency
export AZCOPY_CONCURRENCY_VALUE=32

# Tune buffer size for large files
export AZCOPY_BUFFER_GB=4           # 4 GB buffer per job

# Dry run (list files that WOULD be transferred, no actual copy)
azcopy copy \
  './data/' \
  'https://mystorageaccount123.blob.core.windows.net/container/' \
  --recursive \
  --dry-run

# Generate log file for troubleshooting
azcopy copy \
  './data/' \
  'https://mystorageaccount123.blob.core.windows.net/container/' \
  --recursive \
  --log-level INFO \
  --output-type text

# Check where AzCopy stores logs and plan files
azcopy env   # Shows all AzCopy environment variables and paths
```

---

### Storage Service Comparison and Decision Matrix

```
FINAL DECISION GUIDE:
┌─────────────────────────────────────────────────────────────┐
│ What type of data?                                          │
│                                                             │
├─► UNSTRUCTURED FILES (images, videos, documents, backups)   │
│   └─► Azure Blob Storage                                    │
│       ├── Access pattern: Frequent?   → Hot tier           │
│       ├── Access pattern: Monthly?   → Cool tier           │
│       ├── Access pattern: Yearly?    → Archive tier        │
│       └── Need HDFS/analytics?       → Enable HNS (ADLS2)  │
│                                                             │
├─► NETWORK FILE SHARE (mount like a drive)                   │
│   └─► Azure Files                                           │
│       ├── Windows/multi-OS clients?  → SMB                 │
│       └── Linux, high-perf?          → NFS (Premium)       │
│                                                             │
├─► MESSAGE QUEUE (decouple services)                         │
│   ├── Simple, cheap, < 64KB messages? → Queue Storage       │
│   └── Complex, ordered, pub/sub?      → Service Bus        │
│                                                             │
├─► STRUCTURED KEY-VALUE DATA                                 │
│   ├── Simple queries, cheap?          → Table Storage       │
│   └── Global, high-SLA, rich queries? → CosmosDB           │
│                                                             │
└─► VM DISKS                                                  │
    └─► Managed Disks                                         │
        ├── Mission-critical DB?        → Ultra or P SSD v2  │
        ├── Production workloads?       → Premium SSD         │
        ├── Web/dev workloads?          → Standard SSD        │
        └── Backups, archives?          → Standard HDD        │
```

| Storage Service | Durability | Max Throughput | Auth Options | Protocol |
|---|---|---|---|---|
| Blob Storage | 16 nines (RA-GZRS) | Petabytes/day | Keys, SAS, AAD | HTTPS, REST |
| Azure Files | 16 nines (RA-GZRS) | 10 GiB/s | Keys, SAS, AAD, AD | SMB, NFS, REST |
| Queue Storage | 16 nines (RA-GZRS) | Thousands/sec | Keys, SAS, AAD | HTTPS, REST |
| Table Storage | 16 nines (RA-GZRS) | Millions/row/day | Keys, SAS, AAD | HTTPS, OData |
| ADLS Gen2 | 16 nines (RA-GZRS) | Petabytes/day | Keys, SAS, AAD, ACL | HTTPS, HDFS |
| Managed Disks | 5 nines (ZRS) | 20,000 IOPS (P) | IAM only | VHD (attached) |

---

*Document 11 — Azure Storage Complete Reference Guide*  
*Part of the Azure Zero-to-Hero Learning Series*
