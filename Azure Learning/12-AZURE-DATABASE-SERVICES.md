# 12 — Azure Database Services: Zero-to-Hero Reference

> **Handbook Style** | Updated for 2024 | Covers SQL, Cosmos DB, PostgreSQL, MySQL, Redis, Synapse

---

## Table of Contents

1. [Azure SQL Database](#1-azure-sql-database)
2. [Azure Cosmos DB](#2-azure-cosmos-db)
3. [Azure Database for PostgreSQL](#3-azure-database-for-postgresql)
4. [Azure Database for MySQL](#4-azure-database-for-mysql)
5. [Azure Cache for Redis](#5-azure-cache-for-redis)
6. [Azure Synapse Analytics](#6-azure-synapse-analytics)
7. [Database Comparison and Decision Guide](#7-database-comparison-and-decision-guide)

---

## 1. Azure SQL Database

### 1.1 What Is Azure SQL Database?

Azure SQL Database is a fully managed relational database service built on the latest stable version of the Microsoft SQL Server engine. It handles patching, backups, high availability, and scaling automatically — you focus on your application logic, not infrastructure.

**Three deployment options exist in the Azure SQL family:**

| Option | Description | Best For |
|--------|-------------|----------|
| **Azure SQL Database** | Single database or elastic pool, fully managed PaaS | Modern cloud-native apps, new projects |
| **Azure SQL Managed Instance** | Near-100% SQL Server compatibility, VNet-native | Lift-and-shift of on-prem SQL Server |
| **SQL Server on Azure VM** | Full control over OS and SQL Server engine | Legacy apps needing full SQL Server features |

**Key differences:**

```
Azure SQL Database
  ├── No OS access
  ├── Database-level scope
  ├── Hyperscale up to 100 TB
  ├── Built-in HA and backups
  └── Serverless (auto-pause/resume)

Azure SQL Managed Instance
  ├── Instance-level scope (SQL Server Agent, CLR, linked servers)
  ├── VNet injection (private IP)
  ├── Near 100% SQL Server compat
  └── Good for migration with minimal code change

SQL Server on Azure VM
  ├── Full OS access
  ├── Any SQL Server version
  ├── You manage patches and backups
  └── IaaS — maximum control
```

---

### 1.2 Service Tiers

#### DTU Model (Legacy but still available)

DTU (Database Transaction Unit) is a blended measure of CPU, memory, and I/O. Simple to understand but less flexible.

| Tier | DTUs | Storage | Use Case |
|------|------|---------|----------|
| Basic | 5 | Up to 2 GB | Dev/test, small apps |
| Standard (S0–S12) | 10–3000 | Up to 1 TB | Workloads with predictable requirements |
| Premium (P1–P15) | 125–4000 | Up to 4 TB | OLTP with high IOPS and low latency |

#### vCore Model (Recommended)

vCore lets you independently scale compute (vCores) and storage. You can also bring your own SQL Server license (AHUB).

**General Purpose**
- Standard SSD storage, remote storage, up to 80 vCores
- 5.1 GB RAM per vCore
- Use for most production workloads

**Business Critical**
- Local SSD (NVMe), built-in read replica, up to 80 vCores
- ~7 GB RAM per vCore
- Use for latency-sensitive OLTP, read scale-out

**Hyperscale**
- Distributed storage architecture, up to 100 TB
- Near-instantaneous backups, fast restore
- Read replicas for offloading reads
- Use for very large databases

```
                vCore Tiers
┌──────────────────────────────────────┐
│  General Purpose                     │
│  - Remote SSD, Standard latency      │
│  - 4–80 vCores, 20.4–408 GB RAM      │
│  - Good: web apps, microservices     │
├──────────────────────────────────────┤
│  Business Critical                   │
│  - Local NVMe SSD, ultra-low latency │
│  - Built-in read replica (free)      │
│  - Good: fintech, e-commerce OLTP    │
├──────────────────────────────────────┤
│  Hyperscale                          │
│  - Tiered storage, up to 100 TB      │
│  - Instant snapshots, fast restore   │
│  - Good: very large OLTP/analytics   │
└──────────────────────────────────────┘
```

---

### 1.3 Serverless SQL

Serverless automatically pauses the database during inactivity (saving compute costs) and resumes on the next connection request.

- **Auto-pause delay**: configurable from 1 hour to 7 days
- **Min/max vCores**: scale between your defined bounds based on load
- **Billed per second** of compute usage during active periods

```bash
# Create a serverless SQL database
az sql db create \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --edition GeneralPurpose \
  --family Gen5 \
  --compute-model Serverless \
  --min-capacity 0.5 \
  --max-capacity 4 \
  --auto-pause-delay 60    # pause after 60 minutes idle
```

**When to use Serverless:**
- Dev/test environments
- Databases with irregular usage patterns
- Apps with unpredictable spikes followed by long idle periods

**When NOT to use Serverless:**
- Always-on production workloads (standard vCore is cheaper)
- Latency-sensitive apps (cold start on auto-resume can take ~5 sec)

---

### 1.4 Elastic Pools

An elastic pool lets multiple databases share a set of compute (eDTUs or vCores) and storage resources. Databases with variable usage benefit because peak demand rarely aligns across all databases.

```
                 Elastic Pool
  ┌─────────────────────────────────────────┐
  │   Shared: 200 eDTU / 250 GB storage     │
  │                                         │
  │  ┌──────┐  ┌──────┐  ┌──────┐          │
  │  │ DB 1 │  │ DB 2 │  │ DB 3 │  ...     │
  │  │  50  │  │  80  │  │  10  │          │
  │  │ eDTU │  │ eDTU │  │ eDTU │          │
  │  └──────┘  └──────┘  └──────┘          │
  └─────────────────────────────────────────┘
  Total used = 140, pool has 200 — headroom for spikes
```

```bash
# Create an elastic pool
az sql elastic-pool create \
  --resource-group myRG \
  --server myserver \
  --name mypool \
  --edition Standard \
  --dtu 100 \
  --db-dtu-min 10 \
  --db-dtu-max 50

# Add a database to the pool
az sql db create \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --elastic-pool mypool
```

**SaaS multi-tenant tip:** Elastic Pools are ideal for SaaS apps where each tenant has their own database (database-per-tenant model) but usage is variable.

---

### 1.5 High Availability

#### Zone-Redundant HA

SQL Database (Business Critical and General Purpose Premium) supports zone-redundant deployment, spreading replicas across Availability Zones.

```bash
# Create zone-redundant database
az sql db create \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --edition Premium \
  --zone-redundant true
```

#### Active Geo-Replication

Creates a readable secondary database in any Azure region. You can have up to 4 secondaries.

```bash
# Create geo-replica in another region
az sql db replica create \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --partner-server myserver-secondary \
  --partner-resource-group myRG-secondary

# Manual failover
az sql db replica set-primary \
  --resource-group myRG-secondary \
  --server myserver-secondary \
  --name mydb
```

#### Auto-Failover Groups

Failover groups provide a single read-write listener endpoint that automatically redirects to the primary. You don't need to update connection strings after failover.

```
  Primary Region (East US)              Secondary Region (West US)
  ┌─────────────────────┐               ┌─────────────────────┐
  │  SQL Server A        │◄──────────── │  SQL Server B        │
  │  mydb (read-write)   │  async repl  │  mydb (readable)     │
  └─────────────────────┘               └─────────────────────┘
           │                                       │
           └──────────────┬────────────────────────┘
                          │
               Failover Group Listener
               mygroup.database.windows.net   (read-write → Primary)
               mygroup.secondary.database.windows.net (read-only → Secondary)
```

```bash
# Create failover group
az sql failover-group create \
  --name mygroup \
  --partner-server myserver-secondary \
  --resource-group myRG \
  --server myserver \
  --add-db mydb \
  --failover-policy Automatic \
  --grace-period 1   # hours before auto-failover

# Manual failover (promotes secondary to primary)
az sql failover-group set-primary \
  --name mygroup \
  --resource-group myRG-secondary \
  --server myserver-secondary
```

---

### 1.6 Backup and Restore

#### Point-in-Time Restore (PITR)

Azure SQL automatically takes full, differential, and transaction log backups.

| Tier | Default Retention | Max Retention |
|------|-------------------|---------------|
| Basic | 7 days | 7 days |
| Standard/General Purpose | 7 days | 35 days |
| Premium/Business Critical | 7 days | 35 days |

```bash
# Restore database to a point in time
az sql db restore \
  --dest-name mydb-restored \
  --edition Standard \
  --name mydb \
  --resource-group myRG \
  --server myserver \
  --time "2024-06-01T10:30:00"    # ISO 8601 UTC

# Change backup retention period
az sql db ltr-policy set \
  --resource-group myRG \
  --server myserver \
  --database mydb \
  --weekly-retention P4W \        # 4 weeks
  --monthly-retention P12M \      # 12 months
  --yearly-retention P5Y \        # 5 years
  --week-of-year 1
```

#### Long-Term Retention (LTR)

LTR stores backups in Azure Blob Storage for up to 10 years. Uses weekly, monthly, and yearly backup policies.

```bash
# List LTR backups
az sql db ltr-backup list \
  --location eastus \
  --server myserver \
  --database mydb

# Restore from LTR backup
az sql db ltr-backup restore \
  --backup-id "/subscriptions/.../backups/mydb/..." \
  --dest-database mydb-ltr-restore \
  --dest-resource-group myRG \
  --dest-server myserver
```

---

### 1.7 Security

#### Azure AD Authentication

```bash
# Set Azure AD admin on SQL Server
az sql server ad-admin create \
  --resource-group myRG \
  --server myserver \
  --display-name "MyAADAdmin" \
  --object-id <user-or-group-object-id>

# Connect with Azure AD auth (sqlcmd)
sqlcmd -S myserver.database.windows.net \
       -d mydb \
       -G \            # use Azure AD
       -U user@domain.com
```

#### Transparent Data Encryption (TDE)

TDE encrypts data at rest using AES-256. Enabled by default since 2017.

```bash
# Verify TDE is enabled
az sql db tde show \
  --resource-group myRG \
  --server myserver \
  --database mydb

# Use Customer-Managed Key (BYOK)
az sql server tde-key set \
  --resource-group myRG \
  --server myserver \
  --server-key-type AzureKeyVault \
  --kid "https://myvault.vault.azure.net/keys/mykey/version"
```

#### Always Encrypted

Always Encrypted ensures the database server never sees plaintext sensitive data. Encryption/decryption happens in the client driver.

```sql
-- Create column master key metadata (key is stored in Azure Key Vault)
CREATE COLUMN MASTER KEY MyCMK
WITH (
    KEY_STORE_PROVIDER_NAME = N'AZURE_KEY_VAULT',
    KEY_PATH = N'https://myvault.vault.azure.net/keys/MyKey/abc123'
);

-- Create column encryption key
CREATE COLUMN ENCRYPTION KEY MyCEK
WITH VALUES (
    COLUMN_MASTER_KEY = MyCMK,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x...
);

-- Define encrypted column
CREATE TABLE Patients (
    PatientId    INT IDENTITY PRIMARY KEY,
    Name         NVARCHAR(200),
    SSN          CHAR(11) ENCRYPTED WITH (
                     COLUMN_ENCRYPTION_KEY = MyCEK,
                     ENCRYPTION_TYPE = DETERMINISTIC,   -- supports equality search
                     ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
                 ),
    Salary       MONEY ENCRYPTED WITH (
                     COLUMN_ENCRYPTION_KEY = MyCEK,
                     ENCRYPTION_TYPE = RANDOMIZED,       -- more secure, no search
                     ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
                 )
);
```

#### Dynamic Data Masking

Obfuscates sensitive data in query results for non-privileged users. Data is not changed on disk.

```sql
-- Mask the email column
ALTER TABLE Users
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
-- Result: aXXX@XXXX.com

-- Mask a credit card number (show last 4 digits)
ALTER TABLE Orders
ALTER COLUMN CardNumber ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');

-- Grant unmask permission to privileged user
GRANT UNMASK TO [privileged_user];
```

#### Row-Level Security (RLS)

Filter rows so users only see rows they own.

```sql
-- Create a security predicate function
CREATE FUNCTION dbo.fn_securitypredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS result
           WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);

-- Create security policy
CREATE SECURITY POLICY TenantFilter
ADD FILTER PREDICATE dbo.fn_securitypredicate(TenantId)
ON dbo.Orders,
ADD BLOCK PREDICATE dbo.fn_securitypredicate(TenantId)
ON dbo.Orders AFTER INSERT;

-- Application sets context before querying
EXEC sp_set_session_context @key = N'TenantId', @value = 42, @read_only = 1;
SELECT * FROM Orders;  -- only returns rows for TenantId = 42
```

#### Firewall Rules

```bash
# Allow your client IP
az sql server firewall-rule create \
  --resource-group myRG \
  --server myserver \
  --name AllowMyIP \
  --start-ip-address 203.0.113.5 \
  --end-ip-address 203.0.113.5

# Allow all Azure services (use with caution)
az sql server firewall-rule create \
  --resource-group myRG \
  --server myserver \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

---

### 1.8 Performance

#### Query Performance Insight

Available in the Azure Portal under your database → Query Performance Insight. Shows top resource-consuming queries by CPU, duration, and execution count.

```bash
# Enable Query Store (required for QPI)
# Run in the database:
ALTER DATABASE mydb SET QUERY_STORE = ON;
ALTER DATABASE mydb SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    MAX_STORAGE_SIZE_MB = 1024,
    INTERVAL_LENGTH_MINUTES = 15
);
```

#### Automatic Tuning

Azure SQL can automatically create/drop indexes and force last-known-good query plans.

```bash
# Enable automatic tuning at server level
az sql server update \
  --resource-group myRG \
  --name myserver \
  --auto-tuning-server-default Enabled

# Enable specific tuning options on a database
az sql db op cancel  # not tuning command
# Use portal or REST API for granular control

# T-SQL to check automatic tuning recommendations
SELECT name, type_desc, reason, score, details
FROM sys.dm_db_tuning_recommendations
ORDER BY score DESC;

-- Apply a specific recommendation manually
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 17;
```

#### Index Advisor

```sql
-- Find missing index recommendations
SELECT
    dm_mid.database_id,
    dm_migs.avg_total_user_cost * dm_migs.avg_user_impact * (dm_migs.user_seeks + dm_migs.user_scans) AS improvement_measure,
    'CREATE INDEX [IDX_' + OBJECT_NAME(dm_mid.object_id) + '_' +
        REPLACE(REPLACE(REPLACE(ISNULL(dm_mid.equality_columns,''),', ','_'),'[',''),']','') + ']'
        + ' ON ' + dm_mid.statement
        + ' (' + ISNULL(dm_mid.equality_columns,'')
        + CASE WHEN dm_mid.inequality_columns IS NOT NULL
               THEN (CASE WHEN dm_mid.equality_columns IS NOT NULL THEN ',' ELSE '' END) + dm_mid.inequality_columns
               ELSE '' END + ')'
        + ISNULL(' INCLUDE (' + dm_mid.included_columns + ')','') AS create_index_statement
FROM sys.dm_db_missing_index_groups dm_mig
INNER JOIN sys.dm_db_missing_index_group_stats dm_migs ON dm_migs.group_handle = dm_mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details dm_mid ON dm_mig.index_handle = dm_mid.index_handle
ORDER BY improvement_measure DESC;
```

---

### 1.9 Complete CLI Commands

```bash
# ============================================================
# AZURE SQL SERVER AND DATABASE — COMPLETE CLI REFERENCE
# ============================================================

# --- Resource Group ---
az group create \
  --name myRG \
  --location eastus

# --- Create SQL Server (logical server) ---
az sql server create \
  --resource-group myRG \
  --name myserver \               # globally unique; becomes myserver.database.windows.net
  --location eastus \
  --admin-user sqladmin \
  --admin-password "P@ssw0rd!2024"

# --- Create SQL Database (vCore, General Purpose) ---
az sql db create \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 4 \                  # 4 vCores
  --zone-redundant false

# --- Create SQL Database (DTU, Standard) ---
az sql db create \
  --resource-group myRG \
  --server myserver \
  --name mydb-std \
  --service-objective S2         # S0, S1, S2, S3, P1, P2 etc.

# --- List databases ---
az sql db list \
  --resource-group myRG \
  --server myserver \
  --output table

# --- Show database details ---
az sql db show \
  --resource-group myRG \
  --server myserver \
  --name mydb

# --- Scale up/down ---
az sql db update \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 8                   # scale from 4 to 8 vCores

# --- Configure firewall ---
az sql server firewall-rule create \
  --resource-group myRG \
  --server myserver \
  --name AllowMyIP \
  --start-ip-address 203.0.113.5 \
  --end-ip-address 203.0.113.5

# --- List firewall rules ---
az sql server firewall-rule list \
  --resource-group myRG \
  --server myserver \
  --output table

# --- Delete firewall rule ---
az sql server firewall-rule delete \
  --resource-group myRG \
  --server myserver \
  --name AllowMyIP

# --- Export database to BACPAC (to Storage) ---
az sql db export \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --admin-user sqladmin \
  --admin-password "P@ssw0rd!2024" \
  --storage-key-type StorageAccessKey \
  --storage-key "<storage-account-key>" \
  --storage-uri "https://mystorage.blob.core.windows.net/backups/mydb.bacpac"

# --- Import BACPAC ---
az sql db import \
  --resource-group myRG \
  --server myserver \
  --name mydb-imported \
  --admin-user sqladmin \
  --admin-password "P@ssw0rd!2024" \
  --storage-key-type StorageAccessKey \
  --storage-key "<storage-account-key>" \
  --storage-uri "https://mystorage.blob.core.windows.net/backups/mydb.bacpac"

# --- Point-in-time restore ---
az sql db restore \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --dest-name mydb-restored \
  --time "2024-01-15T08:00:00"

# --- Delete database ---
az sql db delete \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --yes

# --- Delete SQL server ---
az sql server delete \
  --resource-group myRG \
  --name myserver \
  --yes
```

---

### 1.10 Connection Strings

```
# ADO.NET (SQL Authentication)
Server=tcp:myserver.database.windows.net,1433;
Initial Catalog=mydb;
Persist Security Info=False;
User ID=sqladmin;
Password=P@ssw0rd!2024;
MultipleActiveResultSets=False;
Encrypt=True;
TrustServerCertificate=False;
Connection Timeout=30;

# ADO.NET (Azure AD — Integrated)
Server=tcp:myserver.database.windows.net,1433;
Initial Catalog=mydb;
Authentication=Active Directory Integrated;
Encrypt=True;
TrustServerCertificate=False;
Connection Timeout=30;

# JDBC
jdbc:sqlserver://myserver.database.windows.net:1433;
database=mydb;
user=sqladmin;
password=P@ssw0rd!2024;
encrypt=true;
trustServerCertificate=false;
hostNameInCertificate=*.database.windows.net;
loginTimeout=30;

# Python (pyodbc)
Driver={ODBC Driver 18 for SQL Server};
Server=tcp:myserver.database.windows.net,1433;
Database=mydb;
Uid=sqladmin;
Pwd=P@ssw0rd!2024;
Encrypt=yes;
TrustServerCertificate=no;
Connection Timeout=30;
```

---

### 1.11 Bicep Template

```bicep
// sql-database.bicep
param location string = resourceGroup().location
param serverName string = 'myserver-${uniqueString(resourceGroup().id)}'
param dbName string = 'mydb'
param adminLogin string = 'sqladmin'

@secure()
param adminPassword string

resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: serverName
  location: location
  properties: {
    administratorLogin: adminLogin
    administratorLoginPassword: adminPassword
    version: '12.0'
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Enabled'
  }
}

resource firewallAllowAzure 'Microsoft.Sql/servers/firewallRules@2023-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAllAzureIPs'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: dbName
  location: location
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 4
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 34359738368      // 32 GB
    zoneRedundant: false
    requestedBackupStorageRedundancy: 'Geo'
    readScale: 'Disabled'
    highAvailabilityReplicaCount: 0
  }
}

// Diagnostic settings — send metrics to Log Analytics
resource diagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  scope: sqlDatabase
  name: 'sqlDiagnostics'
  properties: {
    workspaceId: '/subscriptions/${subscription().subscriptionId}/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace'
    metrics: [
      {
        category: 'Basic'
        enabled: true
        retentionPolicy: { enabled: false, days: 0 }
      }
    ]
    logs: [
      {
        category: 'SQLInsights'
        enabled: true
      }
      {
        category: 'AutomaticTuning'
        enabled: true
      }
      {
        category: 'QueryStoreRuntimeStatistics'
        enabled: true
      }
    ]
  }
}

output serverFqdn string = sqlServer.properties.fullyQualifiedDomainName
output connectionString string = 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Initial Catalog=${dbName};'
```

---

### 1.12 Best Practices

1. **Use vCore model** over DTU for new projects — more transparency and flexibility
2. **Enable Transparent Data Encryption** — on by default, verify customer-managed keys for regulated industries
3. **Use Azure AD authentication** instead of SQL authentication wherever possible
4. **Configure long-term retention** if your compliance policy requires >35 days backup
5. **Use Auto-failover groups** instead of Active Geo-Replication when you need automatic DNS cutover
6. **Monitor with Query Performance Insight** — identify the top 5 expensive queries weekly
7. **Right-size with Automatic Tuning** — let Azure recommend indexes before adding them manually
8. **Private Endpoints over firewall rules** for production — avoids exposing SQL to public internet
9. **Use connection pooling** (e.g., PgBouncer or ADO.NET pool) to manage connection limits
10. **Don't use serverless for always-on apps** — the cold-start penalty adds latency on first request after pause

---

## 2. Azure Cosmos DB

### 2.1 What Is Cosmos DB?

Azure Cosmos DB is a globally distributed, multi-model, multi-API NoSQL database. It is designed for:
- **Planetary scale**: data replicated across any number of Azure regions
- **Single-digit millisecond latency** at the 99th percentile
- **Five configurable consistency models** from strong to eventual
- **Multiple API surfaces**: use familiar MongoDB, Cassandra, Gremlin, Table, or PostgreSQL APIs on the same underlying engine

```
Azure Cosmos DB Architecture
┌───────────────────────────────────────────────────────────────┐
│                        Cosmos DB Account                       │
│                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  NoSQL API  │  │ MongoDB API  │  │  Cassandra API  ...  │ │
│  └─────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Database                              │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │              Container (Collection)                 │ │ │
│  │  │  Partition Key: /userId                             │ │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │ │ │
│  │  │  │Partition │  │Partition │  │Partition │   ...     │ │ │
│  │  │  │  Key A   │  │  Key B   │  │  Key C   │          │ │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘          │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
  Replicated to: East US ↔ West Europe ↔ Southeast Asia
```

---

### 2.2 APIs

| API | Wire Protocol | Best For |
|-----|---------------|----------|
| **NoSQL (Core)** | HTTP/JSON, SQL-like query | New projects, document model, native Cosmos features |
| **MongoDB** | MongoDB wire protocol | Existing MongoDB apps, document model |
| **Cassandra** | CQL over Thrift/native | Existing Cassandra apps, wide-column model |
| **Gremlin (Graph)** | Gremlin traversal language | Graph data: social networks, recommendation engines |
| **Table** | Azure Table Storage protocol | Lift-and-shift of Azure Table Storage |
| **PostgreSQL (Distributed)** | PostgreSQL wire protocol | Relational + Citus distributed tables |

**Recommendation:** Use the **NoSQL API** for all new projects unless you have an existing app using one of the other protocols. The NoSQL API gets first-class feature support.

---

### 2.3 Consistency Levels

Cosmos DB offers five consistency levels on a spectrum from strongest to weakest. You set a default at the account level and can relax per-request (but not strengthen).

```
Stronger consistency ←────────────────────────→ Higher availability & lower latency

 Strong  │ Bounded Staleness │ Session │ Consistent Prefix │ Eventual
─────────┼───────────────────┼─────────┼───────────────────┼──────────
 Lineariz│ Read within K ver │ Mono-   │ Reads never see   │ No order
 ability │ or T time behind  │ tonic   │ out-of-order      │ guarantee
         │ primary           │ reads   │ writes            │
```

| Level | Description | RU Cost | Use Case |
|-------|-------------|---------|----------|
| **Strong** | Always reads latest committed write, globally | Highest | Financial transactions, inventory systems |
| **Bounded Staleness** | Reads lag behind writes by at most K versions or T seconds | High | Globally distributed apps needing near-consistent reads |
| **Session** | Reads your own writes within a session; stale for others | Medium | Most web/mobile apps, user-specific data |
| **Consistent Prefix** | Never see out-of-order writes | Low | Social media timelines, event sourcing |
| **Eventual** | No ordering guarantee; maximum throughput | Lowest | Non-critical analytics, counters, likes |

```bash
# Set consistency level on account
az cosmosdb update \
  --resource-group myRG \
  --name mycosmosdb \
  --default-consistency-level Session

# Per-request override (REST header)
# x-ms-consistency-level: Eventual
```

**SDK override (C#):**
```csharp
var requestOptions = new ItemRequestOptions
{
    ConsistencyLevel = ConsistencyLevel.Eventual
};
var response = await container.ReadItemAsync<MyItem>(id, partitionKey, requestOptions);
```

---

### 2.4 Request Units (RU/s)

A **Request Unit** (RU) is a normalized unit of compute that abstracts CPU, memory, and IOPS. Every operation has an RU cost:

| Operation | Approximate RU Cost |
|-----------|---------------------|
| Read 1 KB document by ID | 1 RU |
| Write 1 KB document | ~5 RUs |
| Query returning 10 docs | 10–50+ RUs (depends on index) |
| Cross-partition query | Higher, avoid if possible |

**Calculating required RU/s:**
1. Measure peak operations per second (reads + writes)
2. Multiply each operation type by its RU cost
3. Add 20–30% buffer for growth
4. Set autoscale min/max

```bash
# Create Cosmos DB account with autoscale
az cosmosdb create \
  --resource-group myRG \
  --name mycosmosdb \
  --kind GlobalDocumentDB \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --default-consistency-level Session \
  --enable-multiple-write-locations false

# Create database with shared throughput
az cosmosdb sql database create \
  --resource-group myRG \
  --account-name mycosmosdb \
  --name mydb \
  --max-throughput 4000          # autoscale: 400–4000 RU/s

# Create container with dedicated autoscale throughput
az cosmosdb sql container create \
  --resource-group myRG \
  --account-name mycosmosdb \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/userId" \
  --max-throughput 10000         # autoscale: 1000–10000 RU/s

# Check consumed RU/s (x-ms-request-charge header in response)
# Via Azure Monitor metric: TotalRequestUnits
az monitor metrics list \
  --resource "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.DocumentDB/databaseAccounts/mycosmosdb" \
  --metric "TotalRequestUnits" \
  --interval PT1M \
  --output table
```

**RU Optimization Tips:**
- Always read by ID + partition key (1 RU) instead of querying
- Add an index only for fields you query on; exclude large text fields from indexing
- Use Cosmos DB integrated cache to serve reads without consuming RUs
- Prefer `SELECT c.name, c.email FROM c` (projected fields) over `SELECT *`
- Enable bulk mode in SDK for batch inserts

---

### 2.5 Partitioning

Cosmos DB scales horizontally by distributing data across logical partitions (identified by partition key) and physical partitions (server-side shards, max ~50 GB each).

**Partition Key Selection Rules:**

| Rule | Explanation |
|------|-------------|
| High cardinality | Many distinct values → even distribution |
| Even write distribution | Don't use a key that funnels all writes to one partition |
| Frequent query filter | Include partition key in WHERE clause to avoid cross-partition fan-out |
| Immutable field | Partition key cannot be updated after document creation |

**Good partition key examples:**
- `/userId` for user-activity data
- `/tenantId` for multi-tenant SaaS
- `/deviceId` for IoT telemetry
- `/orderId` when querying by order

**Bad partition key examples:**
- `/country` — only ~200 distinct values, hot partitions on popular countries
- `/createdDate` (daily) — today's partition gets all writes
- `/status` — very low cardinality (active/inactive)

**Synthetic partition key** (combine fields for better distribution):
```javascript
// Combine userId + date for time-series data
document.partitionKey = `${document.userId}_${document.date.substring(0,7)}`;
// e.g., "user123_2024-06"
```

---

### 2.6 Global Distribution and Multi-Region Writes

```bash
# Add secondary read region
az cosmosdb update \
  --resource-group myRG \
  --name mycosmosdb \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
               regionName=westeurope failoverPriority=1 isZoneRedundant=false \
               regionName=southeastasia failoverPriority=2 isZoneRedundant=false

# Enable multi-region writes (active-active)
az cosmosdb update \
  --resource-group myRG \
  --name mycosmosdb \
  --enable-multiple-write-locations true

# Configure manual failover priority
az cosmosdb failover-priority-change \
  --resource-group myRG \
  --name mycosmosdb \
  --failover-policies eastus=0 westeurope=1

# Initiate manual failover (promote West Europe to primary)
az cosmosdb failover-priority-change \
  --resource-group myRG \
  --name mycosmosdb \
  --failover-policies westeurope=0 eastus=1
```

**Multi-region write conflict resolution:**
```csharp
// Last Writer Wins (default — uses _ts timestamp)
ContainerProperties containerProps = new ContainerProperties("mycontainer", "/userId");
containerProps.ConflictResolutionPolicy = new ConflictResolutionPolicy
{
    Mode = ConflictResolutionMode.LastWriterWins,
    ResolutionPath = "/_ts"   // use timestamp field to resolve
};

// Custom stored procedure for merge logic
containerProps.ConflictResolutionPolicy = new ConflictResolutionPolicy
{
    Mode = ConflictResolutionMode.Custom,
    ResolutionProcedure = "dbs/mydb/colls/mycontainer/sprocs/mergeConflicts"
};
```

---

### 2.7 TTL (Time-To-Live)

TTL automatically deletes documents after a specified number of seconds. Consumed RUs for deletes do NOT cost extra.

```bash
# Enable TTL on container (default TTL = -1 means no auto-delete unless doc has ttl field)
az cosmosdb sql container update \
  --resource-group myRG \
  --account-name mycosmosdb \
  --database-name mydb \
  --name mycontainer \
  --ttl -1                       # enable TTL; docs with ttl field are deleted

# Set container-level default TTL (delete all docs after 7 days)
az cosmosdb sql container update \
  --resource-group myRG \
  --account-name mycosmosdb \
  --database-name mydb \
  --name sessions \
  --ttl 604800                   # 604800 seconds = 7 days
```

```javascript
// Per-document TTL (overrides container default)
const session = {
    id: "session-abc",
    userId: "user-123",
    data: { ... },
    ttl: 3600    // expire this document in 1 hour
};
await container.items.create(session);
```

---

### 2.8 Change Feed

The change feed is a persistent log of all inserts and updates (not deletes by default) to a Cosmos DB container. It enables event-driven architectures.

```
┌──────────────────┐    Change Feed     ┌─────────────────────────┐
│  Cosmos DB       │ ─────────────────► │  Azure Function          │
│  Container       │                    │  (Cosmos DB trigger)     │
└──────────────────┘                    └──────┬──────────────────┘
                                               │
                          ┌────────────────────┼────────────────────┐
                          ▼                    ▼                    ▼
                   Cache Invalidate    Send Notification     Update Search Index
                   (Redis)             (Event Grid)          (Cognitive Search)
```

```csharp
// Azure Function — Cosmos DB Change Feed trigger
public class ChangeFeedProcessor
{
    [FunctionName("ProcessChanges")]
    public async Task Run(
        [CosmosDBTrigger(
            databaseName: "mydb",
            containerName: "orders",
            Connection = "CosmosDbConnection",
            LeaseContainerName = "leases",
            CreateLeaseContainerIfNotExists = true)]
        IReadOnlyList<Order> changes,
        ILogger log)
    {
        foreach (var order in changes)
        {
            log.LogInformation($"Processing order: {order.Id}");
            // Invalidate Redis cache, send notification, etc.
            await ProcessOrderAsync(order);
        }
    }
}
```

```python
# Python SDK — Change Feed processor
from azure.cosmos import CosmosClient
from azure.cosmos.aio import CosmosClient as AsyncCosmosClient

async def read_change_feed():
    client = AsyncCosmosClient(url=COSMOS_URL, credential=COSMOS_KEY)
    container = client.get_database_client("mydb").get_container_client("orders")

    async for item in container.query_items_change_feed(is_start_from_beginning=True):
        print(f"Changed item: {item['id']}")
        await process_item(item)
```

---

### 2.9 Integrated Cache

The Cosmos DB integrated cache (dedicated gateway) serves repeated point reads and query results without consuming RUs. It's a server-side in-memory cache.

```bash
# Create dedicated gateway (required for integrated cache)
az cosmosdb service create \
  --resource-group myRG \
  --account-name mycosmosdb \
  --kind DedicatedGateway \
  --count 1 \
  --size Cosmos.D4s
```

```csharp
// Use integrated cache — connect via dedicated gateway endpoint
// Set ContentResponseOnWrite = false to enable cache for reads
var options = new CosmosClientOptions
{
    ConnectionMode = ConnectionMode.Gateway,    // must use gateway mode
    GatewayModeMaxConnectionLimit = 10
};
var client = new CosmosClient(
    accountEndpoint: "https://mycosmosdb.documents.azure.com:443/",
    authKeyOrResourceToken: key,
    clientOptions: options
);

// Read with cache — set MaxIntegratedCacheStaleness
var requestOptions = new ItemRequestOptions
{
    DedicatedGatewayRequestOptions = new DedicatedGatewayRequestOptions
    {
        MaxIntegratedCacheStaleness = TimeSpan.FromMinutes(5)
    }
};
```

---

### 2.10 Cosmos DB Emulator

The Cosmos DB Emulator runs locally for development without incurring Azure costs.

```bash
# Run Cosmos DB Emulator with Docker
docker run \
  --publish 8081:8081 \
  --publish 10250-10255:10250-10255 \
  --name cosmosdb-emulator \
  --detach \
  mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator

# Access Data Explorer at: https://localhost:8081/_explorer/index.html

# Default credentials
# Endpoint: https://localhost:8081
# Key: C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==
```

```python
# Connect Python SDK to local emulator
import os
os.environ["PYTHONHTTPSVERIFY"] = "0"   # disable SSL verification for emulator

from azure.cosmos import CosmosClient

client = CosmosClient(
    url="https://localhost:8081",
    credential="C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw=="
)
```

---

### 2.11 SDK Examples

#### C# (.NET SDK v3)

```csharp
using Microsoft.Azure.Cosmos;

// Initialize client (singleton — reuse across requests)
CosmosClient client = new CosmosClient(
    accountEndpoint: Environment.GetEnvironmentVariable("COSMOS_ENDPOINT"),
    authKeyOrResourceToken: Environment.GetEnvironmentVariable("COSMOS_KEY"),
    new CosmosClientOptions { SerializerOptions = new CosmosSerializationOptions
    { PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase } }
);

Container container = client.GetContainer("mydb", "orders");

// CREATE
var order = new Order
{
    Id = Guid.NewGuid().ToString(),
    UserId = "user-123",
    Items = new[] { new OrderItem { ProductId = "prod-1", Qty = 2 } },
    Total = 49.99m,
    CreatedAt = DateTime.UtcNow
};
ItemResponse<Order> createResponse = await container.CreateItemAsync(
    item: order,
    partitionKey: new PartitionKey(order.UserId)
);
Console.WriteLine($"Created: {createResponse.Resource.Id}, RU: {createResponse.RequestCharge}");

// READ (by ID + partition key — cheapest, 1 RU)
ItemResponse<Order> readResponse = await container.ReadItemAsync<Order>(
    id: order.Id,
    partitionKey: new PartitionKey(order.UserId)
);

// UPDATE (replace entire document)
order.Total = 59.99m;
await container.ReplaceItemAsync(order, order.Id, new PartitionKey(order.UserId));

// PATCH (partial update — only update specified fields)
var patchOps = new List<PatchOperation>
{
    PatchOperation.Replace("/total", 59.99m),
    PatchOperation.Add("/tags/-", "vip")
};
await container.PatchItemAsync<Order>(order.Id, new PartitionKey(order.UserId), patchOps);

// DELETE
await container.DeleteItemAsync<Order>(order.Id, new PartitionKey(order.UserId));

// QUERY
var query = new QueryDefinition("SELECT * FROM c WHERE c.userId = @userId AND c.total > @minTotal")
    .WithParameter("@userId", "user-123")
    .WithParameter("@minTotal", 30.0);

using FeedIterator<Order> feed = container.GetItemQueryIterator<Order>(query);
while (feed.HasMoreResults)
{
    FeedResponse<Order> page = await feed.ReadNextAsync();
    foreach (var item in page)
    {
        Console.WriteLine(item.Id);
    }
}
```

#### Python (azure-cosmos SDK)

```python
from azure.cosmos import CosmosClient, PartitionKey, exceptions
import os

# Initialize client
client = CosmosClient(
    url=os.environ["COSMOS_ENDPOINT"],
    credential=os.environ["COSMOS_KEY"]
)

db = client.get_database_client("mydb")
container = db.get_container_client("orders")

# CREATE
order = {
    "id": "order-001",
    "userId": "user-123",
    "items": [{"productId": "prod-1", "qty": 2}],
    "total": 49.99
}
response = container.create_item(body=order)
print(f"Created: {response['id']}")

# READ
try:
    item = container.read_item(item="order-001", partition_key="user-123")
    print(item["total"])
except exceptions.CosmosResourceNotFoundError:
    print("Item not found")

# UPSERT (create or replace)
order["total"] = 59.99
container.upsert_item(body=order)

# QUERY
query = "SELECT * FROM c WHERE c.userId = @userId"
items = list(container.query_items(
    query=query,
    parameters=[{"name": "@userId", "value": "user-123"}],
    enable_cross_partition_query=False   # efficient — stays within partition
))

# BULK operations
from azure.cosmos import _cosmos_client_connection
operations = [
    ("create", ({"id": f"bulk-{i}", "userId": "user-123"},), {}),
    for i in range(100)
]
# Use TransactionalBatch for atomic operations within a partition
batch = container.create_transactional_batch(partition_key="user-123")
batch.create_item({"id": "item-a", "userId": "user-123", "value": 1})
batch.create_item({"id": "item-b", "userId": "user-123", "value": 2})
batch_result = container.execute_item_batch(batch, partition_key="user-123")
```

#### JavaScript (Node.js)

```javascript
const { CosmosClient } = require("@azure/cosmos");

const client = new CosmosClient({
    endpoint: process.env.COSMOS_ENDPOINT,
    key: process.env.COSMOS_KEY
});

const container = client.database("mydb").container("orders");

// CREATE
async function createOrder(order) {
    const { resource } = await container.items.create(order);
    console.log(`Created: ${resource.id}`);
    return resource;
}

// READ
async function readOrder(id, userId) {
    const { resource } = await container.item(id, userId).read();
    return resource;
}

// QUERY
async function getOrdersByUser(userId) {
    const querySpec = {
        query: "SELECT * FROM c WHERE c.userId = @userId ORDER BY c._ts DESC",
        parameters: [{ name: "@userId", value: userId }]
    };
    const { resources } = await container.items.query(querySpec).fetchAll();
    return resources;
}

// BULK (parallel upsert)
async function bulkUpsert(items) {
    const operations = items.map(item => ({
        operationType: "Upsert",
        resourceBody: item,
        partitionKey: item.userId
    }));
    const result = await container.items.bulk(operations);
    return result;
}
```

---

### 2.12 Indexing Policies

By default, Cosmos DB indexes all fields. You can customize this to reduce RU cost and storage.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/userId/?" },
    { "path": "/total/?" },
    { "path": "/createdAt/?" }
  ],
  "excludedPaths": [
    { "path": "/largeTextField/*" },
    { "path": "/binaryData/*" },
    { "path": "/*" }
  ],
  "compositeIndexes": [
    [
      { "path": "/userId", "order": "ascending" },
      { "path": "/createdAt", "order": "descending" }
    ]
  ],
  "spatialIndexes": [
    {
      "path": "/location/*",
      "types": ["Point", "Polygon"]
    }
  ]
}
```

```bash
# Apply custom indexing policy
az cosmosdb sql container update \
  --resource-group myRG \
  --account-name mycosmosdb \
  --database-name mydb \
  --name mycontainer \
  --idx @indexing-policy.json
```

---

### 2.13 Cost Optimization

| Strategy | Savings |
|----------|---------|
| Use autoscale (not manual throughput) | Pay only for max RUs actually used each hour |
| Use serverless for dev/test | No reserved throughput; pure pay-per-request |
| Exclude large fields from indexing | Reduces write RU cost and storage |
| Read by ID + partition key | 1 RU vs 10–100+ RU for a query |
| Use TTL for transient data | Free deletes; reclaim storage automatically |
| Enable integrated cache for repeated reads | Serve from cache; consume 0 RUs |
| Reserved capacity (1 or 3 year) | Up to 65% discount on provisioned throughput |
| Choose Session consistency | Lower RU cost vs Strong/Bounded Staleness |

---

### 2.14 Best Practices

1. **Choose partition key carefully** — it cannot be changed after container creation without migration
2. **Use the NoSQL API** for new projects to get first-class feature support
3. **Never use cross-partition queries in hot paths** — always include partition key in query filter
4. **Use the SDK in Gateway mode** for serverless functions (Connection Pool issues in Direct mode)
5. **Make the CosmosClient a singleton** — initializing it per-request is expensive
6. **Monitor 429 (Too Many Requests)** — this means you're being throttled; scale up RU/s or optimize queries
7. **Use Transactional Batch** for atomic multi-document operations within a single partition
8. **Enable diagnostics** to track slow queries (> 100 ms at p99)
9. **Test with the emulator locally** — same behavior as the cloud service
10. **Use composite indexes** for ORDER BY on multiple fields

---

## 3. Azure Database for PostgreSQL

### 3.1 Single Server vs Flexible Server

| Feature | Single Server (Legacy) | Flexible Server (Recommended) |
|---------|----------------------|-------------------------------|
| Availability | GA (being retired 2025) | GA |
| HA | Zone-redundant with standby | Zone-redundant standby or same-zone |
| Compute tiers | Basic, General Purpose, Memory Optimized | Burstable, General Purpose, Memory Optimized |
| Maintenance window | System-managed | Customer-configurable |
| Stop/start | Not available | Supported |
| PgBouncer | Not built-in | Built-in |
| Major version upgrade | Not in-place | Supported |
| Custom params | Limited | Full access |

**Use Flexible Server for all new deployments.** Single Server is on a deprecation path.

---

### 3.2 Compute Tiers

| Tier | vCores | RAM/vCore | Use Case |
|------|--------|-----------|----------|
| **Burstable** (B-series) | 1, 2 | 2 GB | Dev/test, low-traffic apps, intermittent workloads |
| **General Purpose** (D-series) | 2–96 | 4 GB | Most production workloads, balanced compute/memory |
| **Memory Optimized** (E-series) | 2–96 | 8 GB | High-memory workloads: analytics, large caches, frequent in-memory sorts |

```bash
# Create Flexible Server (General Purpose, zone-redundant HA)
az postgres flexible-server create \
  --resource-group myRG \
  --name mypgserver \
  --location eastus \
  --admin-user pgadmin \
  --admin-password "P@ssw0rd!2024" \
  --sku-name Standard_D4s_v3 \    # 4 vCores, General Purpose
  --tier GeneralPurpose \
  --storage-size 128 \             # GB
  --version 15 \
  --high-availability ZoneRedundant \
  --zone 1 \                       # primary in AZ 1
  --standby-zone 2 \               # standby in AZ 2
  --vnet myVnet \
  --subnet mySubnet

# Create database
az postgres flexible-server db create \
  --resource-group myRG \
  --server-name mypgserver \
  --database-name myappdb \
  --charset utf8 \
  --collation en_US.utf8

# Configure firewall (public access)
az postgres flexible-server firewall-rule create \
  --resource-group myRG \
  --name mypgserver \
  --rule-name AllowMyIP \
  --start-ip-address 203.0.113.5 \
  --end-ip-address 203.0.113.5

# Connect with psql
psql "host=mypgserver.postgres.database.azure.com \
      port=5432 \
      dbname=myappdb \
      user=pgadmin \
      password=P@ssw0rd!2024 \
      sslmode=require"

# Scale compute
az postgres flexible-server update \
  --resource-group myRG \
  --name mypgserver \
  --sku-name Standard_D8s_v3     # scale up to 8 vCores

# Scale storage (can only grow, not shrink)
az postgres flexible-server update \
  --resource-group myRG \
  --name mypgserver \
  --storage-size 256

# Stop server (saves compute cost, storage still billed)
az postgres flexible-server stop \
  --resource-group myRG \
  --name mypgserver

# Start server
az postgres flexible-server start \
  --resource-group myRG \
  --name mypgserver

# Point-in-time restore
az postgres flexible-server restore \
  --resource-group myRG \
  --name mypgserver-restored \
  --source-server mypgserver \
  --restore-time "2024-06-01T10:30:00"
```

---

### 3.3 Extensions

Azure Database for PostgreSQL supports many popular extensions:

```sql
-- List available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- Enable commonly used extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";       -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";        -- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS "pg_trgm";         -- Fuzzy text search
CREATE EXTENSION IF NOT EXISTS "btree_gist";      -- GiST index for B-tree types
CREATE EXTENSION IF NOT EXISTS "postgis";         -- Geospatial data
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- Query performance tracking
CREATE EXTENSION IF NOT EXISTS "vector";          -- pgvector for AI embeddings
```

```bash
# Allow extensions via server parameter
az postgres flexible-server parameter set \
  --resource-group myRG \
  --server-name mypgserver \
  --name azure.extensions \
  --value "uuid-ossp,pgcrypto,pg_trgm,postgis,vector"
```

---

### 3.4 Connection Pooling with PgBouncer

PgBouncer is built into Flexible Server. It reduces connection overhead by maintaining a pool of server-side connections.

```bash
# Enable PgBouncer
az postgres flexible-server update \
  --resource-group myRG \
  --name mypgserver \
  --enable-pgbouncer true

# PgBouncer port: 6432 (vs PostgreSQL port 5432)
# Connect via PgBouncer
psql "host=mypgserver.postgres.database.azure.com \
      port=6432 \
      dbname=myappdb \
      user=pgadmin \
      password=P@ssw0rd!2024 \
      sslmode=require"
```

**PgBouncer modes:**
- **Session** (default): one server connection per client session — similar to direct connection
- **Transaction**: one server connection per transaction — most efficient for apps using short transactions
- **Statement**: one server connection per statement — only for apps with no multi-statement transactions

---

### 3.5 Best Practices

1. **Use Flexible Server** — Single Server is deprecated; migrate before the deadline
2. **Use PgBouncer** (transaction pooling mode) for web apps with many short-lived connections
3. **Use Private Endpoints** — disable public network access in production
4. **Configure `pg_stat_statements`** — identify slow queries and optimize indexes
5. **Tune `work_mem` carefully** — too high causes OOM; too low causes disk sorts
6. **Use connection string with `sslmode=require`** — enforce encrypted connections
7. **Use read replicas** to offload reporting queries from the primary
8. **Automate backups** — PITR is automatic; set retention to 35 days for compliance
9. **Monitor with Azure Monitor** — alert on CPU > 80%, storage > 85%, connections > 80% max
10. **Use `VACUUM ANALYZE`** regularly on large tables with heavy UPDATE/DELETE workloads

---

## 4. Azure Database for MySQL

### 4.1 Single Server vs Flexible Server

Similar to PostgreSQL, MySQL also offers Single Server (legacy) and Flexible Server (recommended).

| Feature | Flexible Server |
|---------|----------------|
| MySQL versions | 5.7, 8.0 |
| HA | Same-zone or zone-redundant |
| Stop/start | Supported |
| IOPS | Provisioned or auto-scale |
| Maintenance window | Customer-configurable |

---

### 4.2 CLI Commands

```bash
# Create MySQL Flexible Server
az mysql flexible-server create \
  --resource-group myRG \
  --name mymysqlserver \
  --location eastus \
  --admin-user mysqladmin \
  --admin-password "P@ssw0rd!2024" \
  --sku-name Standard_D4ds_v4 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --version 8.0 \
  --high-availability ZoneRedundant

# Create database
az mysql flexible-server db create \
  --resource-group myRG \
  --server-name mymysqlserver \
  --database-name myappdb

# Configure firewall
az mysql flexible-server firewall-rule create \
  --resource-group myRG \
  --name mymysqlserver \
  --rule-name AllowMyIP \
  --start-ip-address 203.0.113.5 \
  --end-ip-address 203.0.113.5

# Connect with MySQL CLI
mysql \
  --host=mymysqlserver.mysql.database.azure.com \
  --user=mysqladmin \
  --password="P@ssw0rd!2024" \
  --database=myappdb \
  --ssl-mode=REQUIRED \
  --ssl-ca=/path/to/DigiCertGlobalRootCA.crt.pem

# Scale up
az mysql flexible-server update \
  --resource-group myRG \
  --name mymysqlserver \
  --sku-name Standard_D8ds_v4

# Restore (PITR)
az mysql flexible-server restore \
  --resource-group myRG \
  --name mymysqlserver-restored \
  --source-server mymysqlserver \
  --restore-time "2024-06-01T10:30:00"

# Enable slow query log
az mysql flexible-server parameter set \
  --resource-group myRG \
  --server-name mymysqlserver \
  --name slow_query_log \
  --value ON

az mysql flexible-server parameter set \
  --resource-group myRG \
  --server-name mymysqlserver \
  --name long_query_time \
  --value 2      # log queries taking > 2 seconds
```

---

### 4.3 Connection String Examples

```
# .NET
Server=mymysqlserver.mysql.database.azure.com;
UserID=mysqladmin;
Password=P@ssw0rd!2024;
Database=myappdb;
SslMode=Required;

# Python (mysql-connector)
import mysql.connector
conn = mysql.connector.connect(
    host="mymysqlserver.mysql.database.azure.com",
    user="mysqladmin",
    password="P@ssw0rd!2024",
    database="myappdb",
    ssl_ca="/path/to/DigiCertGlobalRootCA.crt.pem",
    ssl_disabled=False
)

# Node.js (mysql2)
const mysql = require('mysql2/promise');
const conn = await mysql.createConnection({
    host: 'mymysqlserver.mysql.database.azure.com',
    user: 'mysqladmin',
    password: 'P@ssw0rd!2024',
    database: 'myappdb',
    ssl: { rejectUnauthorized: true }
});
```

---

### 4.4 Best Practices

1. **Use Flexible Server** — more features, zone-redundant HA, better performance
2. **Enable slow query log** — identify queries over 2 seconds and add indexes
3. **Use InnoDB** engine — it's the default and the only supported engine in Azure
4. **Set `max_connections`** appropriately — ~(RAM / 1 MB) is a rough guide; use connection pooling (ProxySQL) for apps with many connections
5. **Enable automated backups** with 35-day retention for production
6. **Use Private Endpoints** — restrict MySQL access to your VNet
7. **Monitor replica lag** for read replicas — alert if lag exceeds 60 seconds
8. **Use charset utf8mb4** and collation `utf8mb4_unicode_ci` for full Unicode support (including emoji)

---

## 5. Azure Cache for Redis

### 5.1 What Is Redis Cache?

Azure Cache for Redis is a fully managed, in-memory data store based on the open-source Redis project. It provides sub-millisecond response times for caching, session storage, pub/sub messaging, and more.

```
Application Layer
       │
       │  cache miss
       ▼
┌──────────────────┐     hit    ┌─────────────────────┐
│  Azure Cache     │◄──────────│    Your Application  │
│  for Redis       │           │                      │
│  (in-memory)     │──────────►│  Response < 1ms      │
└──────────────────┘           └──────────────────────┘
       │ cache miss (first time)
       ▼
┌──────────────────┐
│  SQL Database /  │
│  Cosmos DB       │
│  (persistent)    │
└──────────────────┘
```

---

### 5.2 Tiers

| Tier | Max Memory | Clustering | Persistence | Replication | Use Case |
|------|------------|------------|-------------|-------------|----------|
| **Basic** | 53 GB | No | No | No | Dev/test only |
| **Standard** | 53 GB | No | No | Yes (2-node) | Production, non-critical |
| **Premium** | 530 GB | Yes (10 shards) | RDB + AOF | Yes | Production, critical workloads |
| **Enterprise** | 2 TB | Yes | RDB + AOF | Yes | Large-scale, modules (RediSearch, RedisJSON) |
| **Enterprise Flash** | 13 TB | Yes | RDB + AOF | Yes | Very large datasets (uses NVMe SSD) |

```bash
# Create Premium Redis cache with clustering and persistence
az redis create \
  --resource-group myRG \
  --name myrediscache \
  --location eastus \
  --sku Premium \
  --vm-size P3 \                 # P1=6GB, P2=13GB, P3=26GB, P4=53GB, P5=120GB
  --enable-non-ssl-port false \  # force SSL only
  --shard-count 3 \              # 3 shards for clustering
  --minimum-tls-version 1.2

# Enable persistence
az redis update \
  --resource-group myRG \
  --name myrediscache \
  --set redisConfiguration.rdb-backup-enabled=true \
  --set redisConfiguration.rdb-backup-frequency=60 \    # every 60 minutes
  --set redisConfiguration.rdb-backup-max-snapshot-count=1 \
  --set redisConfiguration.rdb-storage-connection-string="<storage-connection-string>"

# Get connection keys
az redis list-keys \
  --resource-group myRG \
  --name myrediscache

# Get hostname
az redis show \
  --resource-group myRG \
  --name myrediscache \
  --query "hostName" \
  --output tsv
```

---

### 5.3 Redis Data Types

```bash
# Connect to Redis CLI
redis-cli \
  -h myrediscache.redis.cache.windows.net \
  -p 6380 \
  --tls \
  -a <access-key>

# ---- STRINGS ----
SET user:100:name "Alice"
GET user:100:name             # "Alice"
SETEX session:abc 3600 "user_data"    # expires in 3600 seconds
INCR pageviews:home           # atomic increment (counter pattern)
INCRBY score:user:100 10      # increment by 10

# ---- HASHES ----
HSET user:100 name "Alice" email "alice@example.com" age 30
HGET user:100 name            # "Alice"
HGETALL user:100              # { name: Alice, email: alice@..., age: 30 }
HINCRBY user:100 age 1        # increment age field

# ---- LISTS ----
RPUSH notifications:user:100 "msg1" "msg2" "msg3"  # right push
LPUSH queue:jobs "job-abc"                          # left push (FIFO queue head)
LRANGE notifications:user:100 0 -1                  # all items
LPOP queue:jobs               # dequeue from left
BRPOP queue:jobs 30           # blocking pop (wait 30s for item — worker pattern)

# ---- SETS ----
SADD tags:post:42 "azure" "cloud" "databases"
SMEMBERS tags:post:42         # all members
SISMEMBER tags:post:42 "azure" # 1 (exists)
SINTER tags:post:42 tags:post:10  # intersection of two sets

# ---- SORTED SETS (Leaderboard) ----
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2200 "player:bob"
ZADD leaderboard 1800 "player:charlie"
ZRANGE leaderboard 0 -1 WITHSCORES REV    # top players, highest first
ZRANK leaderboard "player:alice"           # position (0-indexed)
ZINCRBY leaderboard 300 "player:alice"     # add 300 points

# ---- STREAMS (Event Log) ----
XADD events:orders "*" orderId "ord-001" userId "user-123" total "49.99"
XREAD COUNT 10 STREAMS events:orders 0-0  # read first 10 events
XLEN events:orders            # total events in stream
```

---

### 5.4 Common Patterns

#### Cache-Aside (Lazy Loading)

```csharp
// C# — Cache-Aside with StackExchange.Redis
public async Task<User> GetUserAsync(string userId)
{
    var db = _redis.GetDatabase();
    string cacheKey = $"user:{userId}";

    // Try cache first
    RedisValue cached = await db.StringGetAsync(cacheKey);
    if (cached.HasValue)
    {
        return JsonSerializer.Deserialize<User>(cached);
    }

    // Cache miss — load from database
    User user = await _dbContext.Users.FindAsync(userId);
    if (user != null)
    {
        // Store in cache with 15-minute expiry
        await db.StringSetAsync(
            cacheKey,
            JsonSerializer.Serialize(user),
            TimeSpan.FromMinutes(15)
        );
    }
    return user;
}

// Invalidate on update
public async Task UpdateUserAsync(User user)
{
    await _dbContext.SaveChangesAsync();
    var db = _redis.GetDatabase();
    await db.KeyDeleteAsync($"user:{user.Id}");
}
```

#### Session Store

```csharp
// ASP.NET Core — Use Redis as distributed session store
// Install: Microsoft.Extensions.Caching.StackExchangeRedis

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "myrediscache.redis.cache.windows.net:6380,ssl=true,password=<key>";
    options.InstanceName = "MyApp_";
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

// In controller
HttpContext.Session.SetString("UserId", userId);
string? storedUserId = HttpContext.Session.GetString("UserId");
```

#### Pub/Sub

```python
# Publisher
import redis
r = redis.Redis(
    host="myrediscache.redis.cache.windows.net",
    port=6380,
    password="<access-key>",
    ssl=True
)
r.publish("order-events", '{"orderId": "ord-001", "status": "shipped"}')

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe("order-events")
for message in pubsub.listen():
    if message["type"] == "message":
        print(f"Received: {message['data'].decode()}")
```

#### Leaderboard with Sorted Sets

```python
import redis

r = redis.Redis(host="myrediscache.redis.cache.windows.net", port=6380, password="<key>", ssl=True)

def add_score(player_id, score):
    r.zadd("game:leaderboard", {player_id: score})

def increment_score(player_id, points):
    r.zincrby("game:leaderboard", points, player_id)

def get_top_players(count=10):
    # Returns list of (player_id, score) tuples, highest first
    return r.zrange("game:leaderboard", 0, count - 1, withscores=True, desc=True)

def get_player_rank(player_id):
    rank = r.zrevrank("game:leaderboard", player_id)
    return rank + 1 if rank is not None else None  # 1-indexed

# Usage
add_score("alice", 1500)
add_score("bob", 2200)
increment_score("alice", 500)   # alice now has 2000
print(get_top_players(3))       # [('bob', 2200.0), ('alice', 2000.0)]
print(get_player_rank("alice")) # 2
```

---

### 5.5 .NET (StackExchange.Redis) Connection

```csharp
// Install: StackExchange.Redis
// Singleton pattern — reuse ConnectionMultiplexer

public class RedisConnectionFactory
{
    private static Lazy<ConnectionMultiplexer> _connection =
        new Lazy<ConnectionMultiplexer>(() =>
        {
            var config = ConfigurationOptions.Parse(
                "myrediscache.redis.cache.windows.net:6380,ssl=true,password=<key>,abortConnect=false"
            );
            config.ConnectRetry = 3;
            config.ReconnectRetryPolicy = new ExponentialRetry(5000);
            return ConnectionMultiplexer.Connect(config);
        });

    public static ConnectionMultiplexer Connection => _connection.Value;
}

// Usage
IDatabase db = RedisConnectionFactory.Connection.GetDatabase();

// String operations
await db.StringSetAsync("key", "value", TimeSpan.FromMinutes(10));
string value = await db.StringGetAsync("key");

// Hash operations
await db.HashSetAsync("user:100", new HashEntry[]
{
    new HashEntry("name", "Alice"),
    new HashEntry("email", "alice@example.com")
});
HashEntry[] user = await db.HashGetAllAsync("user:100");

// Transactions
ITransaction tx = db.CreateTransaction();
tx.AddCondition(Condition.KeyNotExists("lock:resource"));
tx.StringSetAsync("lock:resource", "locked", TimeSpan.FromSeconds(30));
bool committed = await tx.ExecuteAsync();

// Lua scripts (atomic operations)
var script = LuaScript.Prepare(@"
    local current = redis.call('INCR', @key)
    if current > @limit then
        redis.call('DECR', @key)
        return 0
    end
    return current
");
var result = await db.ScriptEvaluateAsync(script,
    new { key = (RedisKey)"rate:user:100", limit = 100 });
```

---

### 5.6 Python Connection

```python
import redis
from redis.retry import Retry
from redis.backoff import ExponentialBackoff
import json

# Connection with retry
retry = Retry(ExponentialBackoff(), 3)
r = redis.Redis(
    host="myrediscache.redis.cache.windows.net",
    port=6380,
    password="<access-key>",
    ssl=True,
    retry=retry,
    retry_on_error=[redis.exceptions.ConnectionError, redis.exceptions.TimeoutError],
    decode_responses=True   # return strings instead of bytes
)

# Test connection
r.ping()

# Pipelining (batch commands — reduces round trips)
pipe = r.pipeline()
pipe.set("a", 1)
pipe.set("b", 2)
pipe.set("c", 3)
results = pipe.execute()   # sends all 3 SET commands in one round trip

# Cache decorator pattern
import functools

def cache(ttl=300):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = f"cache:{func.__name__}:{args}:{kwargs}"
            cached = r.get(key)
            if cached:
                return json.loads(cached)
            result = func(*args, **kwargs)
            r.setex(key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

@cache(ttl=60)
def get_product(product_id):
    # expensive database call
    return db.query(f"SELECT * FROM products WHERE id = {product_id}")
```

---

### 5.7 Best Practices

1. **Singleton ConnectionMultiplexer** — creating a new connection per request is expensive and will exhaust ports
2. **Always set TTL** — never store data without expiry; prevents unbounded memory growth
3. **Use Premium tier for production** — clustering, persistence, and zone redundancy
4. **Enable non-SSL port disabled** — always use port 6380 (TLS), never 6379
5. **Avoid `KEYS *`** in production — it blocks the server; use `SCAN` instead
6. **Monitor memory usage** — alert at 80%; eviction policies (`allkeys-lru`) prevent OOM crashes
7. **Use pipelining** for bulk operations — dramatically reduces round-trip time
8. **Design for cache stampede** — use locking/probabilistic early expiration for hot keys
9. **Use geo-replication** (Premium) for multi-region read access with low latency
10. **Track cache hit ratio** — hit rate below 80% means cache sizing or TTL tuning is needed

---

## 6. Azure Synapse Analytics

### 6.1 What Is Synapse?

Azure Synapse Analytics is an enterprise analytics service that brings together big data analytics and data warehousing. It unifies:

- **Dedicated SQL pools** (formerly Azure SQL Data Warehouse) — provisioned MPP data warehouse
- **Serverless SQL pools** — pay-per-query analytics over data lake files
- **Apache Spark pools** — distributed big data processing
- **Integration pipelines** — ETL/ELT similar to Azure Data Factory
- **Power BI integration** — embedded reporting

```
                    Azure Synapse Analytics Workspace
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────────┐   ┌──────────────────┐                  │
│  │ Dedicated SQL    │   │ Serverless SQL   │                  │
│  │ Pool             │   │ Pool             │                  │
│  │ (DW100c–DW30000c)│   │ (Pay per TB      │                  │
│  │ Provisioned MPP  │   │  scanned)        │                  │
│  └──────────────────┘   └──────────────────┘                  │
│                                                                │
│  ┌──────────────────┐   ┌──────────────────┐                  │
│  │ Apache Spark     │   │ Integration      │                  │
│  │ Pool             │   │ Pipelines        │                  │
│  │ (PySpark, Scala) │   │ (ADF-like ETL)   │                  │
│  └──────────────────┘   └──────────────────┘                  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              ADLS Gen2 (Data Lake Storage)               │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

### 6.2 Dedicated SQL Pools

A dedicated SQL pool is a provisioned Massively Parallel Processing (MPP) data warehouse with fixed compute (DWUs).

- **DWU (Data Warehouse Unit)**: unit of compute for dedicated pools
- Range: DW100c to DW30000c
- Pause/resume to save costs when not in use

```sql
-- Create table with distribution strategy (dedicated SQL pool)
CREATE TABLE FactSales
(
    SaleId        INT           NOT NULL,
    CustomerId    INT           NOT NULL,
    ProductId     INT           NOT NULL,
    SaleDate      DATE          NOT NULL,
    Amount        DECIMAL(10,2) NOT NULL
)
WITH
(
    DISTRIBUTION = HASH(CustomerId),   -- distribute rows by CustomerId
    CLUSTERED COLUMNSTORE INDEX        -- best for analytical queries
);

-- Replicated table for small dimension tables
CREATE TABLE DimProduct
(
    ProductId   INT           NOT NULL,
    ProductName NVARCHAR(100),
    Category    NVARCHAR(50)
)
WITH
(
    DISTRIBUTION = REPLICATE,  -- copy to every compute node
    CLUSTERED INDEX (ProductId)
);

-- CTAS (Create Table As Select) — efficient bulk load pattern
CREATE TABLE FactSales_2024
WITH
(
    DISTRIBUTION = HASH(CustomerId),
    CLUSTERED COLUMNSTORE INDEX
)
AS
SELECT * FROM FactSales WHERE YEAR(SaleDate) = 2024;
```

```bash
# Pause dedicated pool (stop billing for compute)
az synapse sql pool pause \
  --resource-group myRG \
  --workspace-name mysynapse \
  --name mydwh

# Resume
az synapse sql pool resume \
  --resource-group myRG \
  --workspace-name mysynapse \
  --name mydwh

# Scale (change DWU level)
az synapse sql pool update \
  --resource-group myRG \
  --workspace-name mysynapse \
  --name mydwh \
  --performance-level DW500c
```

---

### 6.3 Serverless SQL Pool

Query files in ADLS Gen2 directly using T-SQL — no provisioning required.

```sql
-- Query Parquet files
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://mydatalake.dfs.core.windows.net/raw/sales/2024/**',
    FORMAT = 'PARQUET'
) AS rows;

-- Query CSV files with schema definition
SELECT *
FROM OPENROWSET(
    BULK 'https://mydatalake.dfs.core.windows.net/raw/customers/*.csv',
    FORMAT = 'CSV',
    FIRSTROW = 2,    -- skip header row
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n'
) WITH (
    CustomerId   INT,
    Name         NVARCHAR(100),
    Email        NVARCHAR(200),
    Country      NVARCHAR(50)
) AS customers;

-- Create external table for reuse
CREATE EXTERNAL DATA SOURCE myDataLake
WITH ( LOCATION = 'https://mydatalake.dfs.core.windows.net/processed' );

CREATE EXTERNAL FILE FORMAT parquetFormat
WITH ( FORMAT_TYPE = PARQUET );

CREATE EXTERNAL TABLE dbo.SalesExternal
(
    SaleId      INT,
    CustomerId  INT,
    Amount      DECIMAL(10,2),
    SaleDate    DATE
)
WITH (
    LOCATION = '/sales/',
    DATA_SOURCE = myDataLake,
    FILE_FORMAT = parquetFormat
);

SELECT * FROM dbo.SalesExternal WHERE SaleDate > '2024-01-01';
```

---

### 6.4 Apache Spark Pools

```python
# PySpark in Synapse notebook — read from ADLS Gen2
df = spark.read.parquet("abfss://processed@mydatalake.dfs.core.windows.net/sales/")

# Transform
from pyspark.sql.functions import col, year, month, sum as spark_sum

monthly_sales = df \
    .filter(col("status") == "completed") \
    .groupBy(year("sale_date").alias("year"), month("sale_date").alias("month")) \
    .agg(spark_sum("amount").alias("total_sales")) \
    .orderBy("year", "month")

monthly_sales.show()

# Write results back to data lake
monthly_sales.write \
    .mode("overwrite") \
    .format("delta") \
    .save("abfss://curated@mydatalake.dfs.core.windows.net/monthly_sales/")

# Write to dedicated SQL pool
monthly_sales.write \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .option("url", "jdbc:sqlserver://mysynapse.sql.azuresynapse.net:1433;database=mydwh") \
    .option("dbtable", "dbo.MonthlySales") \
    .option("user", "sqladmin") \
    .option("password", "<password>") \
    .mode("overwrite") \
    .save()
```

---

### 6.5 When to Use Synapse vs Databricks

| Factor | Azure Synapse | Azure Databricks |
|--------|--------------|-----------------|
| Primary workload | SQL analytics + warehousing | Data engineering + ML |
| SQL experience | Excellent (T-SQL native) | Good (Spark SQL) |
| Python/Scala ML | Basic | Best-in-class (MLflow, Feature Store) |
| Team skills | SQL-heavy teams | Python/Scala engineers |
| Integration | Tight with Power BI, ADF | Tight with MLflow, Delta Lake |
| Real-time streaming | Limited | Excellent (Structured Streaming) |
| Cost model | Pause/resume dedicated pool | Cluster auto-termination |
| Delta Lake | Supported | Native, primary format |
| Recommendation | Enterprise BI/DWH modernization | Advanced analytics, ML pipelines |

---

## 7. Database Comparison and Decision Guide

### 7.1 Service Comparison Table

| Service | Type | Consistency | Scale | Latency | Best For |
|---------|------|-------------|-------|---------|----------|
| Azure SQL Database | Relational | Strong | Up to 100 TB (Hyperscale) | 1–10 ms | OLTP, line-of-business apps |
| SQL Managed Instance | Relational | Strong | Up to 8 TB | 1–10 ms | SQL Server lift-and-shift |
| Cosmos DB NoSQL | Document | Configurable (5 levels) | Unlimited | < 10 ms (p99) | Global apps, event sourcing |
| Cosmos DB MongoDB | Document | Configurable | Unlimited | < 10 ms (p99) | Existing MongoDB workloads |
| Cosmos DB Cassandra | Wide-column | Configurable | Unlimited | < 10 ms (p99) | Existing Cassandra workloads |
| PostgreSQL Flexible | Relational | Strong | 64 TB storage | 1–5 ms | Open-source PostgreSQL apps |
| MySQL Flexible | Relational | Strong | 16 TB storage | 1–5 ms | LAMP stack, WordPress, Drupal |
| Azure Cache for Redis | In-memory | N/A (cache) | Up to 13 TB | < 1 ms | Caching, sessions, pub/sub |
| Synapse Dedicated | Analytical | Strong | 5 PB | Seconds–minutes | Enterprise DWH, BI |
| Synapse Serverless | Analytical | N/A (query) | Unlimited (scan) | Seconds | Ad-hoc data lake queries |

---

### 7.2 SQL vs NoSQL Decision Guide

```
Start here
    │
    ├── Do you need ACID transactions across multiple tables?
    │       YES → Use SQL (Azure SQL, PostgreSQL, MySQL)
    │       NO  → Continue
    │
    ├── Do you need a fixed/strict schema?
    │       YES → Use SQL
    │       NO  → Continue
    │
    ├── Do you need global distribution with < 10ms latency?
    │       YES → Use Cosmos DB
    │       NO  → Continue
    │
    ├── Is your data model graph-like (nodes + edges)?
    │       YES → Use Cosmos DB Gremlin API
    │       NO  → Continue
    │
    ├── Do you have existing MongoDB/Cassandra code?
    │       YES → Use Cosmos DB MongoDB/Cassandra API
    │       NO  → Continue
    │
    ├── Is this OLAP/analytics with large datasets?
    │       YES → Use Synapse Analytics
    │       NO  → Continue
    │
    └── Flexible schema, document model, or need horizontal scale?
            YES → Use Cosmos DB NoSQL API
            NO  → Use Azure SQL Database (default choice)
```

---

### 7.3 Pricing Summary (Approximate, East US)

| Service | Entry-Level Cost | Notes |
|---------|-----------------|-------|
| Azure SQL Database | ~$15/mo (Basic 5 DTU) | ~$150/mo for S2, ~$370/mo for GP_Gen5_2 |
| Cosmos DB | ~$24/mo (400 RU/s + 25 GB) | Serverless: ~$0.25/M RUs |
| PostgreSQL Flexible | ~$25/mo (B1ms, 20 GB) | ~$100/mo for GP D2s_v3 |
| MySQL Flexible | ~$25/mo (B1ms, 20 GB) | ~$100/mo for GP D2s_v3 |
| Redis Cache | ~$16/mo (Basic C0, 250 MB) | ~$100/mo for Standard C2 |
| Synapse Dedicated | ~$1.51/DWU/hr (DW100c = ~$73/day) | Pause when not in use |
| Synapse Serverless | $5 per TB scanned | First 10 TB/mo free |

---

### 7.4 Architecture Reference: Multi-Tier Application

```
                         ┌──────────────────────────────────┐
                         │         Azure Front Door          │
                         │         (Global Load Balancer)    │
                         └───────────────┬──────────────────┘
                                         │
                         ┌───────────────▼──────────────────┐
                         │         App Service / AKS         │
                         │         (API / Web Layer)         │
                         └──┬────────────┬──────────────┬───┘
                            │            │              │
          ┌─────────────────▼──┐  ┌──────▼──────┐  ┌───▼──────────┐
          │  Azure Cache for   │  │  Azure SQL  │  │  Cosmos DB   │
          │  Redis             │  │  Database   │  │  (NoSQL API) │
          │  (Session/Cache)   │  │  (OLTP)     │  │  (Events)    │
          └────────────────────┘  └──────┬──────┘  └──────────────┘
                                         │
                          ┌──────────────▼────────────────┐
                          │   Azure Synapse Analytics      │
                          │   (Reporting / Analytics)      │
                          └────────────────────────────────┘
```

---

*Last updated: 2024 | Part of the Azure Zero-to-Hero Learning Series*
*Next: [13-AZURE-NETWORKING-COMPLETE.md](./13-AZURE-NETWORKING-COMPLETE.md)*
