# Identity and Access Management (IAM)
## Document 9: Service Principal, Managed Identity, Workload Identity - Complete Guide

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Azure identity types, their purposes, differences, and when to use each

---

## TABLE OF CONTENTS

1. [Identity Types Overview](#overview)
2. [Service Principal (SPN)](#spn)
3. [Managed Identity](#managed-identity)
4. [Workload Identity](#workload-identity)
5. [Comparison & Decision Tree](#comparison)
6. [Implementation Examples](#examples)
7. [Best Practices](#best-practices)

---

## Identity Types Overview {#overview}

### What is an Identity?

**Identity** = A way to prove "who you are" to Azure services

Think of it like **driver's licenses in real life:**

```
Different Types of IDs:

1. Service Principal ≈ Corporate Credentials
   ├─ Created manually by admin
   ├─ Shared by multiple apps
   ├─ Secrets/certificates to manage
   └─ Good for: Automation scripts, external apps

2. Managed Identity ≈ Employee Badge
   ├─ Created/managed by Azure automatically
   ├─ Can be shared across resources
   ├─ Secrets automatically rotated
   └─ Good for: Azure resources that need access

3. Workload Identity ≈ Biometric ID
   ├─ Combines Managed Identity + Kubernetes
   ├─ Pod-specific identity
   ├─ No secrets in pod (token-based)
   └─ Good for: Kubernetes pods with fine-grained access
```

### Identity Hierarchy

```
┌──────────────────────────────────────┐
│     User (You)                       │
│  - Sign in with email/password       │
│  - Azure AD user account             │
│  - Can use Azure Portal              │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│     Service Principal                │
│  - Created for apps/scripts          │
│  - Managed credentials               │
│  - No interactive login              │
└──────────────────────────────────────┘
        ├─ Unmanaged (manual secrets)
        └─ Managed (auto-rotated)

┌──────────────────────────────────────┐
│     Managed Identity                 │
│  - Special type of Service Principal │
│  - Created/managed by Azure          │
│  - Two flavors: System / User        │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│     Workload Identity                │
│  - Managed Identity + Kubernetes     │
│  - Pod-level identity                │
│  - No secrets in Kubernetes          │
└──────────────────────────────────────┘
```

---

## Service Principal (SPN) {#spn}

### What is Service Principal?

**Service Principal** = Application/script identity for Azure  
A "virtual user" that represents an app or automation

```
User Account:
├─ Email: you@company.com
├─ Password: [secret]
└─ Can interact with Azure

Service Principal:
├─ App Name: MyApp
├─ Client ID: 1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p
├─ Client Secret: [secret]
└─ Only apps can interact with Azure
```

### Why Use Service Principal?

**Scenario 1: Automation Scripts**
```
CI/CD Pipeline needs to:
├─ Build Docker image
├─ Push to registry
├─ Deploy to Kubernetes
└─ Who authenticates?
    └─ Service Principal!

Pipeline runner can't use your password (security risk)
Instead: Uses Service Principal with limited permissions
```

**Scenario 2: External Applications**
```
Third-party app needs to:
├─ Call Azure APIs
├─ Access resources
└─ But it's not Azure-native

Solution:
└─ Create Service Principal for that app
   ├─ Assign permissions
   └─ App authenticates as SPN
```

### Types of Service Principal

```
┌──────────────────────────────────────────┐
│        Service Principal Types           │
├──────────────────────────────────────────┤
│                                          │
│  1. Unmanaged SPN (Manual)               │
│  ├─ You create manually                  │
│  ├─ You manage credentials               │
│  ├─ Credentials expire                   │
│  ├─ Your responsibility to rotate        │
│  └─ Risk: Forgotten expiration, breaches │
│                                          │
│  2. Managed Identity (Azure-Managed)    │
│  ├─ Azure creates automatically          │
│  ├─ Azure manages credentials            │
│  ├─ Tokens auto-refresh                  │
│  ├─ No manual rotation needed            │
│  └─ Less risk: Automatic security        │
│                                          │
└──────────────────────────────────────────┘
```

### Creating Service Principal

**Via Azure CLI:**
```bash
# Create Service Principal
az ad sp create-for-rbac \
  --name "MyApplicationSP" \
  --role Contributor \
  --scopes /subscriptions/subscription-id

# Output:
# {
#   "appId": "1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p",
#   "displayName": "MyApplicationSP",
#   "password": "secret123456789",
#   "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
# }

# Store these securely in GitHub Secrets or Azure Key Vault!
```

**Via Azure Portal:**
1. Azure AD → App registrations → New registration
2. Name: "MyApp"
3. Create
4. Certificates & secrets → New client secret
5. Copy Client ID, Tenant ID, Secret

### SPN Permissions

```bash
# List all role assignments for SPN
az role assignment list \
  --assignee "1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p"

# Grant specific role
az role assignment create \
  --assignee-object-id "object-id" \
  --role "Container Registry Push" \
  --scope /subscriptions/subscription-id/resourceGroups/rg-name/providers/Microsoft.ContainerRegistry/registries/myacr

# Remove role
az role assignment delete \
  --assignee "1a2b3c4d-5e6f-7g8h-9i0j-1k2l3m4n5o6p" \
  --role Contributor \
  --scope /subscriptions/subscription-id
```

### SPN in CI/CD

**GitHub Actions Example:**
```yaml
name: Deploy to Azure

on:
  push:
    branches: [main]

env:
  REGISTRY_NAME: azurelearnacr
  IMAGE_NAME: azurelearn

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    # GitHub secrets store SPN credentials
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
        # Where AZURE_CREDENTIALS =
        # {
        #   "clientId": "SPN-App-ID",
        #   "clientSecret": "SPN-Client-Secret",
        #   "subscriptionId": "subscription-id",
        #   "tenantId": "tenant-id"
        # }
    
    - name: Build and push image
      run: |
        az acr build \
          --registry ${{ env.REGISTRY_NAME }} \
          --image ${{ env.IMAGE_NAME }}:latest .
```

### SPN Lifecycle Management

```
Create SPN
    ↓
Add to Key Vault (secret)
    ↓
Configure CI/CD to use SPN
    ↓
Monitor for expiration
    ↓
Rotate secret before expiration
    ├─ Create new secret
    ├─ Update CI/CD
    ├─ Verify working
    └─ Delete old secret
    ↓
Repeat every 1-2 years
```

**Problem:** Easy to forget rotation! → Use Managed Identity instead

---

## Managed Identity {#managed-identity}

### What is Managed Identity?

**Managed Identity** = Service Principal automatically managed by Azure  
Azure handles creation, credential rotation, and cleanup

```
You don't need to:
├─ Store secrets
├─ Manage credentials
├─ Rotate passwords
├─ Track expiration
└─ Worry about security!

Azure automatically:
├─ Creates the identity
├─ Generates tokens
├─ Refreshes tokens
├─ Cleans up when done
└─ Audits all access
```

### Why Managed Identity?

**Compare: SPN vs Managed Identity**

```
Traditional Service Principal:
├─ Create: 10 minutes (manual)
├─ Secret: Stored in code/config (risky!)
├─ Rotation: Every 12-24 months (manual)
├─ Expiration: Can forget → breaks production
├─ Cost: FREE
└─ Risk: HIGH

Managed Identity:
├─ Create: 1 minute (automatic)
├─ Secret: Never stored anywhere (secure!)
├─ Rotation: Hourly by Azure (automatic)
├─ Expiration: Impossible (automatic)
├─ Cost: FREE
└─ Risk: VERY LOW
```

### Types of Managed Identity

#### System-Assigned Identity

```
Created with resource, tied to its lifetime:

┌─────────────────────────────────────┐
│       Create App Service            │
├─────────────────────────────────────┤
│ ✓ Check "Enable System Identity"    │
│   ↓                                  │
│   System-Assigned MI created         │
│   ├─ Client ID generated             │
│   ├─ Principal ID assigned           │
│   ├─ Tied to this App Service        │
│   └─ Deleted when App Service        │
│       deleted                        │
└─────────────────────────────────────┘

When to use:
├─ One resource = one identity
├─ App is standalone
└─ Delete app → delete identity
```

#### User-Assigned Identity (What we use)

```
Created separately, can assign to multiple resources:

Step 1: Create Identity (separate resource)
┌─────────────────────────────────────┐
│ Managed Identity                    │
│ Name: azure-learn-app-pod-identity  │
│ Type: User-Assigned                 │
│ Client ID: bf9ea53d-...             │
└─────────────────────────────────────┘

Step 2: Assign to resources
┌─────────────────────────────────────┐
│ AKS Pod #1                          │
│ ↓ Use this identity                 │
│ azure-learn-app-pod-identity        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ AKS Pod #2                          │
│ ↓ Same identity                     │
│ azure-learn-app-pod-identity        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Other App Service                   │
│ ↓ Can also use                      │
│ azure-learn-app-pod-identity        │
└─────────────────────────────────────┘

When to use:
├─ Multiple resources same identity
├─ Lifecycle independent
└─ Delete one resource → identity stays
```

### Creating Managed Identity

**Via Azure CLI:**
```bash
# Create user-assigned managed identity
az identity create \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity

# Get details
az identity show \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity \
  --query "{clientId: clientId, principalId: principalId}"

# Output:
# {
#   "clientId": "bf9ea53d-6351-42bf-aa45-3328a0bd296f",
#   "principalId": "4741bbdd-46a2-40c2-95de-a4f29c3cc305"
# }
```

**Via Bicep:**
```bicep
resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'azure-learn-app-pod-identity'
  location: location
}

output managedIdentityClientId string = managedIdentity.properties.clientId
output managedIdentityPrincipalId string = managedIdentity.properties.principalId
```

### Assigning Permissions to Managed Identity

```bash
# Get Principal ID
PRINCIPAL_ID=$(az identity show \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity \
  --query principalId -o tsv)

# Grant Key Vault access
az keyvault set-policy \
  --name azurelearnkvhof7rpcc \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list

# Grant ACR access
az role assignment create \
  --assignee-object-id $PRINCIPAL_ID \
  --role "AcrPull" \
  --scope /subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev/providers/Microsoft.ContainerRegistry/registries/azurelearnacrhof7rpcc

# Grant Cosmos DB access
az cosmosdb sql role assignment create \
  --account-name azurelearndbhof7rpcc \
  --database-name AzureLearnDb \
  --principal-id $PRINCIPAL_ID \
  --role-definition-id 00000000-0000-0000-0000-000000000002  # Data Contributor
```

### Managed Identity in Application Code

**C# - Automatic with DefaultAzureCredential:**
```csharp
// In Program.cs
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Azure.Cosmos;

builder.Services.AddSingleton(sp => {
    // DefaultAzureCredential automatically handles:
    // 1. Managed Identity (if in Azure)
    // 2. Service Principal (if env vars set)
    // 3. User credentials (for local development)
    var credential = new DefaultAzureCredential();
    
    var cosmosClient = new CosmosClient(
        accountEndpoint: configuration["CosmosDb:Endpoint"],
        tokenCredential: credential
    );
    
    return cosmosClient;
});

// Key Vault access
var keyVaultUri = new Uri(configuration["KeyVault:Url"]);
var secretClient = new SecretClient(keyVaultUri, new DefaultAzureCredential());

// All authentication happens automatically!
// No passwords, no secrets, no manual token refresh
```

---

## Workload Identity {#workload-identity}

### What is Workload Identity?

**Workload Identity** = Kubernetes pod gets its own Managed Identity  
Pod authenticates as itself, not the node

```
Old Way (Node Identity):
┌─────────────────────────┐
│      AKS Node           │
│  System-Assigned MI     │
│                         │
│  ┌──────────────────┐   │
│  │ Pod A            │   │
│  │ Uses: Node MI    │   │
│  └──────────────────┘   │
│                         │
│  ┌──────────────────┐   │
│  │ Pod B            │   │
│  │ Uses: Node MI    │   │
│  └──────────────────┘   │
│                         │
│  Problem:               │
│  └─ All pods same perms │
└─────────────────────────┘

New Way (Pod Identity):
┌──────────────────────────────────────┐
│         AKS Node                     │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ Pod A                          │  │
│  │ Managed Identity: app-identity │  │
│  │ Permissions: Read Key Vault    │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ Pod B                          │  │
│  │ Managed Identity: worker-id    │  │
│  │ Permissions: Read storage      │  │
│  └────────────────────────────────┘  │
│                                      │
│  Benefit:                            │
│  └─ Each pod has own permissions     │
└──────────────────────────────────────┘
```

### How Workload Identity Works

**Flow:**

```
Step 1: Pod Startup
┌────────────────────────────────────┐
│ Pod starts in Kubernetes           │
│                                    │
│ ServiceAccount annotation:         │
│ azure.workload.identity/use: true  │
└────────────────────────────────────┘
                ↓

Step 2: OIDC Token Generation
┌────────────────────────────────────┐
│ Kubernetes OIDC Provider generates │
│ token proving:                     │
│ "This is pod X in namespace Y"     │
│                                    │
│ Token contains:                    │
│ - Cluster name: azurelearn-dev-aks │
│ - Namespace: azure-learn-app       │
│ - ServiceAccount: azure-learn-app  │
│ - Pod name: azure-learn-app-5cf... │
└────────────────────────────────────┘
                ↓

Step 3: Pod Needs Secret
┌────────────────────────────────────┐
│ Pod: "I need cosmos DB connection" │
│ Pod: DefaultAzureCredential.GetToken()
└────────────────────────────────────┘
                ↓

Step 4: Workload Identity Interceptor
┌────────────────────────────────────┐
│ Interceptor webhook intercepts     │
│ token request                      │
│                                    │
│ Reads ServiceAccount annotation:   │
│ client-id: bf9ea53d-...            │
└────────────────────────────────────┘
                ↓

Step 5: Azure AD Token Exchange
┌────────────────────────────────────┐
│ Pod sends:                         │
│ - Kubernetes OIDC token            │
│ - Managed Identity Client ID       │
│ - Request scope (Azure API)        │
│                                    │
│ To: Azure AD                       │
└────────────────────────────────────┘
                ↓

Step 6: Verification & Token Issuance
┌────────────────────────────────────┐
│ Azure AD:                          │
│ 1. Verify K8s token valid          │
│ 2. Verify pod matches client ID    │
│ 3. Issue Azure access token        │
│                                    │
│ Returns: JWT token valid for 1hr   │
└────────────────────────────────────┘
                ↓

Step 7: Pod Uses Azure Services
┌────────────────────────────────────┐
│ Pod: "Here's my token"             │
│ Key Vault: "Token valid! Here's    │
│            secret"                 │
│                                    │
│ Pod: "Thanks! Now I can connect    │
│      to Cosmos DB"                 │
└────────────────────────────────────┘
```

### Setting Up Workload Identity

**Step 1: Create Managed Identity**
```bash
# Already done:
# Name: azure-learn-app-pod-identity
# Client ID: bf9ea53d-6351-42bf-aa45-3328a0bd296f
```

**Step 2: Enable on AKS Cluster**
```bash
az aks update \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --enable-oidc-issuer \
  --enable-workload-identity
```

**Step 3: Kubernetes Configuration**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-learn-app-sa
  namespace: azure-learn-app
  annotations:
    # Link to Managed Identity
    azure.workload.identity/client-id: "bf9ea53d-6351-42bf-aa45-3328a0bd296f"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-learn-app
  namespace: azure-learn-app
spec:
  template:
    metadata:
      annotations:
        # Enable workload identity
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: azure-learn-app-sa
      containers:
      - name: app
        image: azurelearn:v1.0
        # Container automatically has Managed Identity!
```

**Step 4: Application Code (No Changes!)**
```csharp
// Existing code works automatically!
var credential = new DefaultAzureCredential();

// DefaultAzureCredential now tries:
// 1. Workload Identity (pod annotation)
// 2. Managed Identity (system-assigned)
// 3. Service Principal
// 4. Azure CLI credentials
```

### Workload Identity Benefits

```
✅ No Secrets in Pod
  └─ Kubernetes secret only has client ID (not sensitive)

✅ Pod-Specific Identity
  └─ Pod A can access Key Vault
  └─ Pod B cannot access Key Vault
  └─ Fine-grained control

✅ Automatic Token Refresh
  └─ Tokens refresh before expiry
  └─ No manual rotation

✅ Audit Trail
  └─ Azure AD logs all token requests
  └─ Know exactly which pod accessed what

✅ Compliance Ready
  └─ No credentials in YAML
  └─ Meets security standards
  └─ Suitable for regulated industries
```

---

## Comparison & Decision Tree {#comparison}

### Side-by-Side Comparison

| Feature | User | SPN | Managed Identity | Workload Identity |
|---------|------|-----|-----------------|------------------|
| **What** | Person | App/script | Auto-managed SPN | Pod in K8s |
| **Creation** | Azure AD | Manual | Automatic | Automatic |
| **Secrets** | Password | Yes (expire) | No (auto-rotate) | No (token-based) |
| **Rotation** | Manual | Manual | Automatic | Automatic |
| **Token expiry** | N/A | 1+ years | Hourly | 1 hour |
| **Where used** | Portal, CLI | CI/CD, scripts | VMs, App Service, AKS | Kubernetes pods only |
| **Cost** | FREE | FREE | FREE | FREE |
| **Setup time** | N/A | 10 mins | 2 mins | 3 mins (with WI) |
| **Audit trail** | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| **Best for** | Humans | External apps | Azure resources | K8s pods |
| **Risk level** | Medium | High | Low | Very Low |

### Decision Tree: Which Identity to Use?

```
Start: Need to access Azure services?
│
├─ YES
│  │
│  ├─ Who/What needs access?
│  │  │
│  │  ├─ A Person (You)
│  │  │  └─ Use: User Account
│  │  │     └─ Sign in with email/password
│  │  │
│  │  ├─ An App/Script (CI/CD, third-party)
│  │  │  │
│  │  │  ├─ Is it in Azure cloud?
│  │  │  │  │
│  │  │  │  ├─ YES (VM, App Service, AKS)
│  │  │  │  │  │
│  │  │  │  │  ├─ Is it a Kubernetes pod?
│  │  │  │  │  │  │
│  │  │  │  │  │  ├─ YES
│  │  │  │  │  │  │  └─ Use: Workload Identity ⭐
│  │  │  │  │  │  │     └─ Best: No secrets, pod-specific
│  │  │  │  │  │  │
│  │  │  │  │  │  └─ NO (App Service, VM, etc)
│  │  │  │  │  │     └─ Use: Managed Identity
│  │  │  │  │  │        └─ Best: Simple, auto-managed
│  │  │  │  │  │
│  │  │  │  │  └─ System vs User?
│  │  │  │  │     ├─ One resource, simple
│  │  │  │  │     │  └─ System-Assigned
│  │  │  │  │     │
│  │  │  │  │     └─ Multiple resources, shared
│  │  │  │  │        └─ User-Assigned ⭐
│  │  │  │  │
│  │  │  │  └─ NO (External, non-Azure)
│  │  │  │     └─ Use: Service Principal
│  │  │  │        └─ OK: Need to manage secrets
│  │  │  │
│  │  │  └─ Is it DevOps pipeline (CI/CD)?
│  │  │     ├─ GitHub Actions, Azure DevOps
│  │  │     │  └─ Use: Service Principal
│  │  │     │     └─ OK: Store in GitHub Secrets
│  │  │     │
│  │  │     └─ Or use Workload Identity Federation
│  │  │        └─ New: GitHub → Azure without secrets!
│  │  │
│  │  └─ Azure Service (Managed App)
│  │     └─ Use: What the service defaults to
│  │        └─ Usually Managed Identity
│  │
│  └─ Summarize Choice:
│     ├─ Human? → User Account
│     ├─ K8s Pod? → Workload Identity ⭐
│     ├─ Azure Resource? → Managed Identity
│     ├─ External App? → Service Principal
│     └─ CI/CD? → Service Principal or Fed (new)
│
└─ NO
   └─ Public access (no auth needed)
      ├─ Use: Managed Public Endpoint
      └─ Or: Shared Key (for storage)
```

---

## Implementation Examples {#examples}

### Example 1: Workload Identity Setup (Our Project)

**Step 1: Create Managed Identity**
```bash
az identity create \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity
```

**Step 2: Get Identity Details**
```bash
CLIENT_ID=$(az identity show \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity \
  --query clientId -o tsv)

PRINCIPAL_ID=$(az identity show \
  --resource-group azure-learn-rg-dev \
  --name azure-learn-app-pod-identity \
  --query principalId -o tsv)

echo "Client ID: $CLIENT_ID"
echo "Principal ID: $PRINCIPAL_ID"
```

**Step 3: Grant Permissions**
```bash
# Key Vault access
az keyvault set-policy \
  --name azurelearnkvhof7rpcc \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list

# Container Registry access
az role assignment create \
  --assignee-object-id $PRINCIPAL_ID \
  --role "AcrPull" \
  --scope /subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev/providers/Microsoft.ContainerRegistry/registries/azurelearnacrhof7rpcc
```

**Step 4: Kubernetes ServiceAccount**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-learn-app-sa
  namespace: azure-learn-app
  annotations:
    azure.workload.identity/client-id: "bf9ea53d-6351-42bf-aa45-3328a0bd296f"
```

**Step 5: Pod Uses Workload Identity**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-learn-app
spec:
  template:
    metadata:
      annotations:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: azure-learn-app-sa
      containers:
      - name: app
        image: azurelearn:v1.0
```

**Step 6: App Code (No Changes)**
```csharp
// Automatically uses Workload Identity!
var credential = new DefaultAzureCredential();
var cosmosClient = new CosmosClient(endpoint, credential);
```

### Example 2: Service Principal in GitHub Actions

**Step 1: Create SPN**
```bash
az ad sp create-for-rbac \
  --name "github-azure-deploy" \
  --role Contributor \
  --scopes /subscriptions/subscription-id

# Output: Save all values!
```

**Step 2: Add to GitHub Secrets**
```
Go to GitHub repo:
Settings → Secrets and variables → Actions → New repository secret

Name: AZURE_CREDENTIALS
Value: {
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "..."
}
```

**Step 3: GitHub Actions Workflow**
```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy
      run: |
        az deployment group create \
          --resource-group azure-learn-rg-dev \
          --template-file infra/main.bicep \
          --parameters location=westus environment=dev
```

### Example 3: Managed Identity on App Service

**Step 1: Create App Service with MI**
```bash
az appservice plan create \
  --name myplan \
  --resource-group mygroup \
  --sku B1

az webapp create \
  --resource-group mygroup \
  --plan myplan \
  --name myapp \
  --assign-identity  # Enable System-Assigned MI
```

**Step 2: Grant Permissions**
```bash
PRINCIPAL_ID=$(az webapp identity show \
  --name myapp \
  --resource-group mygroup \
  --query principalId -o tsv)

az keyvault set-policy \
  --name myvault \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list
```

**Step 3: App Code**
```csharp
var credential = new ManagedIdentityCredential();
var secretClient = new SecretClient(vaultUri, credential);
var secret = await secretClient.GetSecretAsync("MySecret");
```

---

## Best Practices {#best-practices}

### ✅ DO

```
✅ Use Managed Identity for Azure resources
   ├─ VMs, App Service, AKS, Functions
   └─ Saves time, improves security

✅ Use Workload Identity for Kubernetes pods
   ├─ Best practice for K8s
   ├─ Pod-specific permissions
   └─ No secrets in YAML

✅ Use Service Principal for CI/CD
   ├─ GitHub Actions, Azure DevOps
   ├─ Store secrets in vault
   └─ Rotate regularly

✅ Grant least privilege permissions
   ├─ Use custom roles, not Owner
   ├─ Grant only needed permissions
   └─ Use scope (not subscription)

✅ Audit and monitor access
   ├─ Enable Azure AD audit logs
   ├─ Alert on SPN secret rotation
   └─ Review permissions quarterly

✅ Rotate secrets regularly
   ├─ SPN: Every 12 months
   ├─ Passwords: Every 90 days
   └─ Set calendar reminder
```

### ❌ DON'T

```
❌ DON'T use passwords for service accounts
   └─ Passwords get stolen, expire, forgotten

❌ DON'T store secrets in code
   ├─ Git history = permanent
   ├─ Visible in logs
   └─ Compromised in breach

❌ DON'T grant "Owner" role
   ├─ Too much power
   ├─ Can delete everything
   └─ Use specific roles

❌ DON'T forget to rotate SPN secrets
   ├─ Expiration = broken pipeline
   ├─ Better: Use Managed Identity
   └─ Or: Workload Identity

❌ DON'T share credentials via Slack/email
   └─ Use Key Vault or CI/CD Secrets

❌ DON'T use System-Assigned when shared needed
   ├─ Creates duplicate identities
   ├─ Harder to manage
   └─ Use User-Assigned instead
```

---

## Key Takeaways

| Use Case | Identity Type | Why |
|----------|--------------|-----|
| You accessing Azure | User Account | Azure AD manages |
| CI/CD pipeline | Service Principal | External app |
| VM/App Service | Managed Identity | Built-in, auto-managed |
| Kubernetes pod | Workload Identity | Pod-specific, secure |
| Legacy app | Service Principal | Only option for external |
| Multiple resources | User-Assigned MI | Cost-effective sharing |
| Single resource | System-Assigned MI | Simplified lifecycle |

---

## Quick Decision Matrix

```
Question: I need my [APP] to access [SERVICE]

  Kubernetes Pod?
  ├─ YES → Use Workload Identity
  │         └─ Most secure, best practice
  │
  └─ NO
     │
     ├─ Is it in Azure?
     │  ├─ YES (VM, App Service, etc)
     │  │  └─ Use Managed Identity
     │  │     └─ Simpler, auto-managed
     │  │
     │  └─ NO (External, third-party)
     │     └─ Use Service Principal
     │        └─ Only option
     │
     └─ Is it DevOps pipeline?
        ├─ YES → Service Principal
        │        └─ Or Workload Identity Federation (new!)
        │
        └─ NO → See above
```

---

## Additional Resources

- [Workload Identity Documentation](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [Managed Identity Documentation](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
- [Service Principal Documentation](https://learn.microsoft.com/azure/active-directory/develop/app-objects-and-service-principals)
- [Azure RBAC Roles](https://learn.microsoft.com/azure/role-based-access-control/role-definitions)
