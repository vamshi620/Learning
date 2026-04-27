# Azure Setup for .NET Teams
## File 01: Get Your Team Ready on Azure (First Day)

---

## What You'll Do in This File

By the end of this guide, your team will have:
- ✅ An Azure subscription with proper organization
- ✅ A Resource Group for your project
- ✅ Every team member logged in with correct permissions
- ✅ Azure CLI configured and working
- ✅ Understanding of Azure's structure

**Time Required:** 30-45 minutes (Team Lead) + 10 minutes (each developer)

---

## Step 1: Understand Azure's Structure

Before creating anything, understand how Azure organizes resources:

```
Your Company
└── Azure Subscription (billing boundary — like a credit card)
    ├── Resource Group: "myproject-dev"     ← DEV environment
    │   ├── App Service: myapp-dev
    │   ├── SQL Database: mydb-dev
    │   └── Key Vault: mykv-dev
    │
    ├── Resource Group: "myproject-staging"  ← STAGING environment
    │   ├── App Service: myapp-staging
    │   └── SQL Database: mydb-staging
    │
    └── Resource Group: "myproject-prod"    ← PRODUCTION environment
        ├── App Service: myapp-prod
        ├── SQL Database: mydb-prod
        └── Key Vault: mykv-prod
```

### Key Concepts Explained

**Subscription** — Your billing account. All charges go here. Think of it like a project credit card.

**Resource Group (RG)** — A container that holds related Azure resources. Like a folder on your computer.
- **Rule of thumb:** One Resource Group per environment (dev, staging, prod)
- When you delete an RG, everything inside it gets deleted too
- Resources in one RG can talk to resources in another RG

**Resource** — Any Azure service you create (database, web app, storage account, etc.)

**Region** — The physical datacenter location (e.g., East US, West Europe). Choose one close to your users.

---

## Step 2: Create Your Azure Subscription

### Option A: Free Account (for learning/prototyping)

Go to [azure.microsoft.com/free](https://azure.microsoft.com/free) and sign up:
- $200 free credit for 30 days
- 12 months of free services (including App Service, SQL Database)
- Always-free services (Azure Functions 1M executions, etc.)

### Option B: Company Subscription (for real projects)

Your company admin creates a Pay-As-You-Go or Enterprise Agreement subscription. Ask your IT department for access.

### Verify Your Subscription

```powershell
# Login to Azure (opens browser)
az login

# See your subscription
az account show --output table

# If you have multiple subscriptions, set the right one
az account list --output table
az account set --subscription "Your Subscription Name"
```

---

## Step 3: Create Resource Groups

Resource Groups organize your Azure resources by environment. Here's exactly what to create:

```powershell
# ─── Define your project variables ─────────────────────────────────
# Change these to match your project!

$PROJECT = "myproject"           # Your project name (lowercase, no spaces)
$LOCATION = "eastus"             # Azure region (see options below)

# To see all available regions:
# az account list-locations --output table

# ─── Create Resource Groups ────────────────────────────────────────

# Development environment
az group create `
  --name "$PROJECT-dev-rg" `
  --location $LOCATION `
  --tags Environment=Development Project=$PROJECT Team=DotNet

# Staging environment
az group create `
  --name "$PROJECT-staging-rg" `
  --location $LOCATION `
  --tags Environment=Staging Project=$PROJECT Team=DotNet

# Production environment
az group create `
  --name "$PROJECT-prod-rg" `
  --location $LOCATION `
  --tags Environment=Production Project=$PROJECT Team=DotNet

# Verify they were created
az group list --output table
```

### Why Tags Matter

Tags are key-value labels on resources. They help with:
- **Cost tracking** — "Which project is costing us the most?"
- **Automation** — "Delete all resources tagged Environment=Development at night"
- **Organization** — "Who owns this resource?"

Always tag with at minimum: `Environment`, `Project`, `Team`

---

## Step 4: Set Up Team Access (RBAC)

RBAC (Role-Based Access Control) controls who can do what in Azure.

### Understanding Roles

| Role | What They Can Do | Who Gets It |
|------|-----------------|-------------|
| **Owner** | Everything + manage access | Project lead / Admin only |
| **Contributor** | Create, modify, delete resources (cannot manage access) | Developers, DevOps |
| **Reader** | View resources only | Stakeholders, QA |
| **Specific roles** | Limited access (e.g., "Website Contributor") | Specialized team members |

### Assign Roles to Team Members

```powershell
# ─── Add a developer as Contributor to the dev Resource Group ──────

# First, find the user's Object ID
az ad user show --id "developer@yourcompany.com" --query id --output tsv

# Assign Contributor role on dev Resource Group
az role assignment create `
  --assignee "developer@yourcompany.com" `
  --role "Contributor" `
  --scope "/subscriptions/<sub-id>/resourceGroups/$PROJECT-dev-rg"

# For multiple developers, repeat the command:
az role assignment create `
  --assignee "dev2@yourcompany.com" `
  --role "Contributor" `
  --scope "/subscriptions/<sub-id>/resourceGroups/$PROJECT-dev-rg"

# Give a more restricted role for production
az role assignment create `
  --assignee "developer@yourcompany.com" `
  --role "Reader" `
  --scope "/subscriptions/<sub-id>/resourceGroups/$PROJECT-prod-rg"

# Verify assignments
az role assignment list `
  --resource-group "$PROJECT-dev-rg" `
  --output table
```

### Security Best Practice

```
DEV Resource Group:
  ├── Developers → Contributor (can create/modify/delete)
  └── QA → Reader (view only)

STAGING Resource Group:
  ├── DevOps Lead → Contributor
  └── Developers → Reader

PRODUCTION Resource Group:
  ├── DevOps Lead → Contributor (through CI/CD only)
  └── Everyone else → Reader
```

---

## Step 5: Each Team Member — Login and Verify

Every developer on the team should run these commands:

```powershell
# ─── ONE-TIME SETUP (each developer) ───────────────────────────────

# 1. Login to Azure
az login

# 2. Verify you're on the correct subscription
az account show --output table

# 3. Verify you can see the Resource Groups
az group list --output table

# 4. Test that you have the right permissions
az group show --name "myproject-dev-rg" --output table
# If this works → you have at least Reader access

# 5. Set convenient defaults (so you don't type them every time)
az configure --defaults `
  group="myproject-dev-rg" `
  location="eastus"

# 6. Verify defaults are set
az configure --list-defaults
```

---

## Step 6: Naming Convention

Agree on a naming convention NOW. It saves confusion later.

### Recommended Pattern

```
{project}-{environment}-{resource-type}

Examples:
  myproject-dev-rg          ← Resource Group
  myproject-dev-app         ← App Service
  myproject-dev-sql         ← SQL Database
  myproject-dev-kv          ← Key Vault
  myproject-dev-acr         ← Container Registry (AKS path)
  myproject-dev-aks         ← AKS Cluster
  myproject-dev-ai          ← Application Insights
```

### Azure Naming Rules

| Resource | Rules | Example |
|----------|-------|---------|
| Resource Group | 1-90 chars, alphanumeric + hyphens | `myproject-dev-rg` |
| App Service | 2-60 chars, globally unique | `myproject-dev-app` |
| SQL Server | 1-63 chars, globally unique, lowercase | `myproject-dev-sql` |
| Key Vault | 3-24 chars, globally unique | `myproject-dev-kv` |
| Storage Account | 3-24 chars, globally unique, lowercase, no hyphens | `myprojectdevsa` |
| ACR | 5-50 chars, alphanumeric only | `myprojectdevacr` |
| AKS Cluster | 1-63 chars | `myproject-dev-aks` |

---

## Step 7: Understanding the Azure Portal

The Azure Portal ([portal.azure.com](https://portal.azure.com)) is the web UI for managing everything.

### Key Pages to Bookmark

```
portal.azure.com
├── Home                          ← Dashboard with recent resources
├── Resource Groups               ← Your organized containers
├── All Resources                 ← Everything you've created
├── Cost Management + Billing     ← Track spending
├── Microsoft Entra ID            ← User management, app registrations
├── Azure Active Directory        ← Same as above (old name)
└── Cloud Shell                   ← Terminal in the browser (top-right icon)
```

### Portal vs CLI — When to Use What

| Task | Use Portal | Use CLI |
|------|-----------|---------|
| Exploring/learning | ✅ Visual, easy to understand | |
| One-time setup | ✅ Point and click | |
| Repeated tasks | | ✅ Scriptable, faster |
| CI/CD pipelines | | ✅ Required for automation |
| Debugging | ✅ Logs, metrics, visual tools | |
| Team automation | | ✅ Share scripts in Git |

**Recommendation:** Use the **Portal** to learn and understand services. Use the **CLI** for anything you'll do more than once.

---

## ✅ Checkpoint — Are You Ready?

Before moving to File 02, verify:

- [ ] Every team member can run `az login` successfully
- [ ] Resource Groups exist for dev (and optionally staging/prod)
- [ ] Team members have Contributor access to the dev Resource Group
- [ ] You've agreed on a naming convention
- [ ] You've bookmarked the Azure Portal

---

> **Next Step:** Open [02-PREPARE-DOTNET-APP-FOR-CLOUD.md](02-PREPARE-DOTNET-APP-FOR-CLOUD.md) to make your .NET app cloud-ready.
