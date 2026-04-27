# 16 – Azure Governance & Cost Management

---

## Table of Contents
1. [Azure Management Hierarchy](#1-azure-management-hierarchy)
2. [Azure Policy](#2-azure-policy)
3. [Resource Tagging Strategy](#3-resource-tagging-strategy)
4. [Azure Cost Management & Billing](#4-azure-cost-management--billing)
5. [Azure Advisor](#5-azure-advisor)
6. [Resource Locks](#6-resource-locks)
7. [Azure Landing Zones](#7-azure-landing-zones)
8. [Subscription Design Patterns](#8-subscription-design-patterns)
9. [Azure Resource Graph](#9-azure-resource-graph)
10. [Cost Optimization Checklist](#10-cost-optimization-checklist)

---

## 1. Azure Management Hierarchy

```
Root Management Group  (tenant-level, 1 per AAD tenant)
├── Management Group: Corp
│   ├── Management Group: Platform
│   │   ├── Subscription: connectivity
│   │   ├── Subscription: identity
│   │   └── Subscription: management
│   └── Management Group: Landing Zones
│       ├── Management Group: Corp-Apps
│       │   ├── Subscription: prod-app1
│       │   └── Subscription: dev-app1
│       └── Management Group: Online-Apps
│           └── Subscription: prod-web
└── Management Group: Sandbox
    └── Subscription: sandbox-01
```

- Up to **6 levels** of nesting (excluding root)
- Up to **10,000** management groups per directory
- Each management group can have up to **10,000** subscriptions

```bash
# Create management group
az account management-group create \
  --name "corp-platform" \
  --display-name "Corp Platform"

# Move subscription into management group
az account management-group subscription add \
  --name "corp-platform" \
  --subscription "<subscription-id>"

# List management groups
az account management-group list --output table

# Show hierarchy
az account management-group show --name "corp-platform" --expand --recurse
```

---

## 2. Azure Policy

Azure Policy enforces organizational rules and assesses compliance across your resources.

### Policy Effect Types

| Effect | Behavior |
|---|---|
| **Deny** | Blocks the resource create/update operation |
| **Audit** | Allows but marks non-compliant in audit log |
| **AuditIfNotExists** | Audits if a related resource doesn't exist (e.g., no diagnostics setting) |
| **DeployIfNotExists** | Deploys a related resource if it doesn't exist (e.g., auto-deploy Log Analytics agent) |
| **Modify** | Add/replace/remove tags or properties |
| **Append** | Append fields to existing resource (e.g., add allowed IPs) |
| **Disabled** | Policy exists but is not enforced |

### Important Built-in Policies

| Category | Policy Name | Effect |
|---|---|---|
| Tags | Require a tag on resources | Deny |
| Tags | Require a tag and its value on resources | Deny |
| Tags | Inherit a tag from the resource group | Modify |
| Locations | Allowed locations | Deny |
| Locations | Allowed locations for resource groups | Deny |
| Compute | Allowed virtual machine size SKUs | Deny |
| Compute | Not allowed resource types | Deny |
| Storage | Storage accounts should use customer-managed keys | Audit |
| Storage | Secure transfer to storage accounts should be enabled | Deny |
| Storage | Storage account public access should be disallowed | Deny |
| Networking | RDP access from Internet should be blocked | Deny |
| Networking | SSH access from Internet should be blocked | Deny |
| SQL | SQL servers should use customer-managed keys | Audit |
| SQL | SQL Database transparent data encryption should be enabled | DeployIfNotExists |
| Key Vault | Key vaults should have soft delete enabled | Deny |
| Key Vault | Key vaults should have purge protection enabled | Deny |
| Monitoring | Audit diagnostic setting | AuditIfNotExists |
| Security | Enable Defender for Servers | DeployIfNotExists |
| Kubernetes | Kubernetes clusters should not allow privileged containers | Deny |
| Kubernetes | Kubernetes clusters should use internal load balancers | Audit |

### Custom Policy Definition (Full JSON)

```json
{
  "mode": "All",
  "displayName": "Require CostCenter tag on resource groups",
  "description": "Enforces the presence of a CostCenter tag on all resource groups.",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Resources/subscriptions/resourceGroups"
        },
        {
          "field": "tags['CostCenter']",
          "exists": false
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

```bash
# Create policy definition
az policy definition create \
  --name "require-costcenter-tag" \
  --display-name "Require CostCenter tag" \
  --description "Denies resource groups without CostCenter tag" \
  --rules require-costcenter-tag.json \
  --mode All

# Assign policy to subscription
az policy assignment create \
  --name "require-costcenter-rg" \
  --display-name "Require CostCenter on RGs" \
  --policy "require-costcenter-tag" \
  --scope "/subscriptions/<sub-id>"

# Assign with exclusion
az policy assignment create \
  --name "require-tags-prod" \
  --policy "require-costcenter-tag" \
  --scope "/subscriptions/<sub-id>" \
  --not-scopes "/subscriptions/<sub-id>/resourceGroups/exempt-rg"

# Check compliance state
az policy state list \
  --resource-group prod-rg \
  --query "[?complianceState=='NonCompliant']" \
  --output table

# Create remediation task (for DeployIfNotExists policies)
az policy remediation create \
  --name "deploy-log-analytics" \
  --policy-assignment "/subscriptions/<sub>/providers/Microsoft.Authorization/policyAssignments/deploy-la" \
  --resource-discovery-mode ExistingNonCompliant
```

### Policy Initiatives (Policy Sets)

Group multiple policies into a single assignment:

```bash
# Create initiative
az policy set-definition create \
  --name "corp-baseline" \
  --display-name "Corporate Baseline" \
  --definitions '[
    {"policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/<id1>"},
    {"policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/<id2>"}
  ]'

# Assign initiative
az policy assignment create \
  --name "corp-baseline-assignment" \
  --policy-set-definition "corp-baseline" \
  --scope "/subscriptions/<sub-id>"
```

---

## 3. Resource Tagging Strategy

### Recommended Tag Taxonomy

| Tag | Values | Purpose |
|---|---|---|
| `Environment` | prod, staging, dev, sandbox | Cost allocation, lifecycle |
| `CostCenter` | CC-1234, CC-5678 | Chargeback |
| `Owner` | alice@contoso.com | Contact for alerts |
| `Application` | ecommerce, payments-api | App grouping |
| `ManagedBy` | terraform, bicep, manual | Governance |
| `DataClassification` | public, internal, confidential, restricted | Compliance |
| `SLA` | tier1, tier2, tier3 | Ops prioritization |
| `AutoShutdown` | true, false | Cost savings |
| `CreatedDate` | 2024-01-15 | Lifecycle management |
| `Project` | proj-phoenix | Project tracking |

**Tag limits:** 50 tags per resource, tag name ≤ 512 chars, tag value ≤ 256 chars.

### Enforce Tags with Azure Policy (Modify Effect)

```json
{
  "mode": "Indexed",
  "displayName": "Inherit Environment tag from resource group",
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "tags['Environment']", "exists": false },
        { "value": "[resourceGroup().tags['Environment']]", "notEquals": "" }
      ]
    },
    "then": {
      "effect": "modify",
      "details": {
        "roleDefinitionIds": [
          "/providers/microsoft.authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
        ],
        "operations": [{
          "operation": "add",
          "field": "tags['Environment']",
          "value": "[resourceGroup().tags['Environment']]"
        }]
      }
    }
  }
}
```

```bash
# Add tags to existing resource
az resource tag \
  --ids /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.Compute/virtualMachines/myvm \
  --tags Environment=prod CostCenter=CC-1234 Owner=alice@contoso.com

# Tag a resource group
az group update \
  --name myrg \
  --tags Environment=prod Application=ecommerce

# List resources with specific tag
az resource list \
  --tag Environment=prod \
  --output table

# PowerShell: bulk tag all resources in RG
$rg = "prod-rg"
$tags = @{Environment="prod"; CostCenter="CC-1234"}
Get-AzResource -ResourceGroupName $rg | ForEach-Object {
  Update-AzTag -ResourceId $_.ResourceId -Tag $tags -Operation Merge
}
```

---

## 4. Azure Cost Management & Billing

### Budgets with Alerts

```bash
# Create a monthly budget with email alerts at 80% and 100%
az consumption budget create \
  --budget-name "prod-monthly-budget" \
  --amount 5000 \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2025-12-31" \
  --resource-group prod-rg \
  --notifications '[
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["alice@contoso.com","billing-team@contoso.com"],
      "contactRoles": ["Owner","Contributor"]
    },
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["cto@contoso.com"]
    }
  ]'

# List budgets
az consumption budget list --output table
```

### Cost Savings Options

| Option | Discount | Best For |
|---|---|---|
| **Pay-as-you-go** | 0% | Unpredictable, short-term |
| **1-Year Reserved Instance** | ~40% | Stable workloads 1+ year |
| **3-Year Reserved Instance** | ~60-72% | Stable workloads 3+ years |
| **Azure Savings Plan (1-year)** | ~15% | Flexible compute |
| **Azure Savings Plan (3-year)** | ~17% | Flexible compute |
| **Spot VMs** | ~60-90% | Batch, interruptible workloads |
| **Azure Hybrid Benefit (Windows)** | ~40% | Existing Windows Server licenses |
| **Azure Hybrid Benefit (SQL)** | ~55% | Existing SQL Server licenses |
| **Dev/Test pricing** | ~40% | Non-production environments |

### Export Cost Data

```bash
# Export daily cost data to storage account
az costmanagement export create \
  --name "daily-cost-export" \
  --type ActualCost \
  --scope "/subscriptions/<sub-id>" \
  --storage-account-id /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.Storage/storageAccounts/mysa \
  --storage-container "cost-exports" \
  --storage-directory "azure-costs" \
  --recurrence Daily \
  --recurrence-period from="2024-01-01" to="2025-12-31"
```

---

## 5. Azure Advisor

Advisor analyzes your resource configuration and usage and gives prioritized recommendations.

### Five Pillars

| Pillar | Example Recommendations |
|---|---|
| **Cost** | Shut down unused VMs, resize underutilized VMs, buy reserved instances, delete empty App Service plans |
| **Security** | Enable MFA, apply system updates, enable Defender plans, rotate access keys |
| **Reliability** | Add redundancy, enable backup, configure health probes, enable soft delete |
| **Operational Excellence** | Use ARM templates, enable service health alerts, remove unused resources |
| **Performance** | Use Premium SSD, increase DTUs for SQL, use CDN, enable caching |

```bash
# List all recommendations
az advisor recommendation list --output table

# Filter by category
az advisor recommendation list --category Cost --output table

# Get recommendation detail
az advisor recommendation show --ids "/subscriptions/<sub>/providers/Microsoft.Advisor/recommendations/<rec-id>"

# Postpone recommendation for 7 days
az advisor recommendation postpone --ids "<rec-id>" --days 7

# Dismiss recommendation
az advisor recommendation disable --ids "<rec-id>"
```

---

## 6. Resource Locks

Resource locks prevent accidental deletion or modification of critical resources.

### Lock Types

| Lock | Delete | Modify | Use Case |
|---|---|---|---|
| **CanNotDelete** | ❌ Blocked | ✅ Allowed | Production databases, Key Vaults |
| **ReadOnly** | ❌ Blocked | ❌ Blocked | Shared networking, production config |

**Inheritance:** Locks apply to all child resources within the locked scope.

**Note:** ReadOnly on a Storage Account prevents listing keys, creating SAS tokens, and modifying blobs — it is very restrictive.

```bash
# Lock a resource group (CanNotDelete)
az lock create \
  --name "prod-rg-lock" \
  --resource-group prod-rg \
  --lock-type CanNotDelete \
  --notes "Production resources - deletion requires executive approval"

# Lock a specific resource (ReadOnly)
az lock create \
  --name "vnet-readonly" \
  --resource-group prod-rg \
  --resource-type Microsoft.Network/virtualNetworks \
  --resource-name prod-vnet \
  --lock-type ReadOnly

# List locks on a resource group
az lock list --resource-group prod-rg --output table

# Delete a lock
az lock delete \
  --name "prod-rg-lock" \
  --resource-group prod-rg
```

---

## 7. Azure Landing Zones

An Azure Landing Zone is a pre-configured, scalable environment based on Microsoft's Cloud Adoption Framework (CAF).

```
Tenant Root Group
├── Platform MG
│   ├── Management Sub    (Log Analytics, Automation, Backup Vault)
│   ├── Connectivity Sub  (Hub VNet, Firewall, ExpressRoute/VPN, DNS)
│   └── Identity Sub      (ADFS, AD Connect, Domain Controllers)
└── Landing Zones MG
    ├── Corp MG           (connected to hub VNet via peering)
    │   ├── prod-sub
    │   └── dev-sub
    └── Online MG         (internet-facing, no hub peering required)
        └── web-sub
```

### CAF Phases

| Phase | Focus |
|---|---|
| Strategy | Define motivations, business outcomes |
| Plan | Rationalize digital estate, align teams |
| Ready | Build the landing zone (infrastructure) |
| Migrate | Move existing workloads |
| Innovate | Build new cloud-native apps |
| Govern | Enforce guardrails (Policy, RBAC) |
| Manage | Operations baseline, monitoring |
| Secure | Zero Trust, Defender for Cloud |

---

## 8. Subscription Design Patterns

### Pattern Comparison

| Pattern | Pros | Cons |
|---|---|---|
| **Single subscription** | Simple | Hits limits, poor isolation |
| **Environment-based** (dev/staging/prod per sub) | Good isolation, clear billing | More management overhead |
| **Team-based** (one sub per team) | Strong blast radius isolation | Many subscriptions to manage |
| **Workload-based** (one sub per app) | Perfect isolation | Very many subscriptions |

### Key Subscription Limits

| Resource | Limit |
|---|---|
| Resource groups per subscription | 980 |
| Resources per resource group (per type) | 800 |
| vCores per region | 350 (default, can increase) |
| VNets per subscription | 1,000 |
| Public IPs per subscription | 1,000 |
| Key Vaults per subscription | 250 |

### Moving Resources Between Subscriptions

```bash
# Move resources to a different subscription
az resource move \
  --destination-group new-rg \
  --destination-subscription-id <target-sub-id> \
  --ids \
    /subscriptions/<src-sub>/resourceGroups/src-rg/providers/Microsoft.Compute/virtualMachines/myvm \
    /subscriptions/<src-sub>/resourceGroups/src-rg/providers/Microsoft.Network/networkInterfaces/mynic
```

---

## 9. Azure Resource Graph

Resource Graph lets you query your entire Azure estate using Kusto Query Language (KQL).

```bash
# Install extension
az extension add --name resource-graph

# Basic query: list all VMs
az graph query -q "Resources | where type =~ 'Microsoft.Compute/virtualMachines' | project name, location, resourceGroup, properties.hardwareProfile.vmSize"

# List resources without a specific tag
az graph query -q "
Resources
| where isnull(tags.Environment) or tags.Environment == ''
| project name, type, resourceGroup, location"

# Find all public IP addresses
az graph query -q "
Resources
| where type =~ 'Microsoft.Network/publicIPAddresses'
| project name, resourceGroup, properties.ipAddress, properties.publicIPAllocationMethod"

# Count resources per region
az graph query -q "
Resources
| summarize count() by location
| order by count_ desc"

# Find NSGs with any-any inbound rules
az graph query -q "
Resources
| where type =~ 'Microsoft.Network/networkSecurityGroups'
| mv-expand rule = properties.securityRules
| where rule.properties.direction == 'Inbound'
  and rule.properties.access == 'Allow'
  and rule.properties.sourceAddressPrefix == '*'
  and rule.properties.destinationPortRange == '*'
| project name, resourceGroup, ruleName = rule.name"

# Find orphan managed disks (not attached to any VM)
az graph query -q "
Resources
| where type =~ 'Microsoft.Compute/disks'
| where isnull(properties.diskState) or properties.diskState == 'Unattached'
| project name, resourceGroup, properties.diskSizeGB, location"
```

---

## 10. Cost Optimization Checklist

| Item | Action | Potential Savings |
|---|---|---|
| Right-size underutilized VMs | Use Advisor recommendations (< 5% avg CPU) | 20–50% |
| Delete stopped VMs with no reservation | Deallocate or delete idle VMs | 100% of VM cost |
| Purchase Reserved Instances | 1-year or 3-year for steady workloads | 40–72% |
| Enable Azure Hybrid Benefit | Apply AHUB to Windows/SQL workloads | 40–55% |
| Use Spot VMs for batch workloads | Replace on-demand with spot | 60–90% |
| Delete orphan managed disks | Find unattached disks (Resource Graph) | $10–100/month per disk |
| Delete unattached public IPs | Static IPs cost ~$4/month when idle | Small but accumulates |
| Enable auto-shutdown for dev/test VMs | Use Azure Automation or DevTest Labs policy | 70%+ on dev VMs |
| Archive old Storage to Cool/Archive tier | Move blobs not accessed for 90+ days | 50–80% storage cost |
| Delete unused App Service Plans | Empty plans still charge | 100% of plan cost |
| Review and downsize oversized SQL | Analyze DTU/vCore usage | 30–60% |
| Clean up old snapshots | Keep only last 2–3 snapshots per disk | $5–20/snapshot/month |
| Use consumption-based Functions | Replace always-on App Service with Functions | 80%+ for low-traffic |
| Optimize Cosmos DB RU/s | Use autoscale, reduce manual provisioned RU/s | 50%+ |
| Review ExpressRoute circuits | Unused circuits still charge port fee | $500–2000/month |
| Enable Cost Management budgets | Get alerted before overspend | Prevents surprises |
| Tag all resources for chargeback | Enable per-team/per-app cost visibility | Drives accountability |
| Use Policy to enforce auto-shutdown tags | Automatically enforce shutdown schedule | Reduces dev waste |
| Consolidate Log Analytics workspaces | Remove duplicate workspaces | $50–200/workspace/month |
| Review and delete unused ACR images | Old images consume storage | Small but accumulates |
