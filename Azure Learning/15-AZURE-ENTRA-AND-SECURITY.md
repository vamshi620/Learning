# 15 – Azure Entra ID & Security

---

## Table of Contents
1. [Microsoft Entra ID Overview](#1-microsoft-entra-id-overview)
2. [Authentication Methods](#2-authentication-methods)
3. [Conditional Access](#3-conditional-access)
4. [Privileged Identity Management (PIM)](#4-privileged-identity-management-pim)
5. [Azure RBAC](#5-azure-rbac)
6. [App Registrations & Service Principals](#6-app-registrations--service-principals)
7. [Managed Identities](#7-managed-identities)
8. [Azure Key Vault](#8-azure-key-vault)
9. [Microsoft Defender for Cloud](#9-microsoft-defender-for-cloud)
10. [Zero Trust on Azure](#10-zero-trust-on-azure)

---

## 1. Microsoft Entra ID Overview

Microsoft Entra ID (formerly Azure Active Directory) is Microsoft's cloud-based identity and access management service.

### Tenant, Directory, Subscription Relationship

```
Azure Account
└── Tenant (Entra ID Directory)  ←─ identity plane
    ├── Users / Groups / Devices
    ├── App Registrations
    └── Subscriptions (1..N)     ←─ billing/resource plane
        └── Resource Groups
            └── Resources
```

A **tenant** is a dedicated instance of Entra ID. A single tenant can own many subscriptions. Each subscription trusts exactly one tenant.

### Entra ID Licensing

| Feature | Free | P1 | P2 |
|---|---|---|---|
| User/group management | ✅ | ✅ | ✅ |
| SSO (cloud apps) | ✅ | ✅ | ✅ |
| MFA (basic) | ✅ | ✅ | ✅ |
| Self-Service Password Reset | ✅ | ✅ | ✅ |
| Conditional Access | ❌ | ✅ | ✅ |
| Dynamic Groups | ❌ | ✅ | ✅ |
| Hybrid Identity (PTA/PHS) | ❌ | ✅ | ✅ |
| Identity Protection | ❌ | ❌ | ✅ |
| Privileged Identity Management | ❌ | ❌ | ✅ |
| Access Reviews | ❌ | ❌ | ✅ |

### Users and Groups

```bash
# Create a user
az ad user create \
  --display-name "Alice Smith" \
  --user-principal-name alice@contoso.com \
  --password "P@ssw0rd123!" \
  --force-change-password-next-sign-in true

# List users
az ad user list --output table

# Disable a user
az ad user update --id alice@contoso.com --account-enabled false

# Delete a user
az ad user delete --id alice@contoso.com

# Create a security group
az ad group create \
  --display-name "DevTeam" \
  --mail-nickname "devteam"

# Add member to group
az ad group member add \
  --group "DevTeam" \
  --member-id <user-object-id>

# List group members
az ad group member list --group "DevTeam" --output table
```

**Group types:**
- **Security Group** – used for RBAC and app access
- **Microsoft 365 Group** – includes mailbox, Teams, SharePoint
- **Dynamic Security Group** – membership based on attribute rules (requires P1)

Dynamic membership rule example:
```
(user.department -eq "Engineering") and (user.accountEnabled -eq true)
```

### Directory Roles

| Role | Description |
|---|---|
| Global Administrator | Full control of Entra ID |
| User Administrator | Manage users and groups |
| Billing Administrator | Manage subscriptions and billing |
| Security Administrator | Manage security policies and read security data |
| Security Reader | Read-only security info |
| Application Administrator | Manage app registrations and enterprise apps |
| Cloud Application Administrator | Like App Admin but without on-prem proxy |
| Privileged Role Administrator | Manage role assignments and PIM |
| Exchange Administrator | Manage Exchange Online |
| SharePoint Administrator | Manage SharePoint Online |
| Helpdesk Administrator | Reset passwords for non-admins |

---

## 2. Authentication Methods

### Hybrid Identity Comparison

| Method | How It Works | Pros | Cons |
|---|---|---|---|
| **Password Hash Sync (PHS)** | Password hashes synced to Entra ID | Simple, resilient to on-prem outage | Hash copy in cloud |
| **Pass-through Authentication (PTA)** | Auth forwarded to on-prem AD agent | Password never leaves on-prem | Depends on on-prem availability |
| **Federation (ADFS)** | Claims-based auth via ADFS servers | Full control, smart card support | Complex, high cost |

### Multi-Factor Authentication (MFA)

MFA requires something you **know** + something you **have**/**are**.

| MFA Method | Security Level | UX |
|---|---|---|
| Microsoft Authenticator (push) | High | Excellent |
| FIDO2 security key | Highest | Good |
| Windows Hello for Business | Highest | Excellent |
| Software OATH token | Medium | Good |
| SMS / Voice call | Low | Poor |
| Hardware OATH token | Medium | OK |

**Configuring MFA Registration Policy (Entra ID P1/P2):**
- Portal: Entra ID → Security → Authentication methods → Registration campaign
- Require MFA registration for all users within 14 days

### Passwordless Authentication

```
FIDO2 key        → tap key → authenticated (phishing-resistant)
Windows Hello   → biometric/PIN → authenticated (device-bound)
Authenticator   → phone sign-in → number match challenge
```

Enable passwordless in Entra ID:
- Portal: Entra ID → Security → Authentication methods → FIDO2 / Microsoft Authenticator

### Self-Service Password Reset (SSPR)

```bash
# SSPR requires P1 or combined MFA+SSPR registration
# Enable via Portal: Entra ID → Password reset → Properties
# Authentication methods: at least 2 of (Email, Phone, Authenticator app, Security questions)
```

Writeback (write reset password back to on-prem AD) requires:
- Azure AD Connect with password writeback enabled
- P1 or P2 license

---

## 3. Conditional Access

Conditional Access enforces: **If (who + where + what) → Then (allow / block / require control).**

```
Signals               Access Controls
────────────          ───────────────
User/Group            → Require MFA
Named Location        → Require compliant device
Device state          → Require hybrid join
Application           → Require approved app
Sign-in risk          → Block
User risk             → Force password change
```

### Policy Examples

**Policy 1 – Require MFA for All Administrators**
```json
{
  "displayName": "Require MFA for Admins",
  "state": "enabled",
  "conditions": {
    "users": { "includeRoles": ["Global Administrator","Security Administrator","User Administrator"] },
    "applications": { "includeApplications": ["All"] }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
```

**Policy 2 – Block Legacy Authentication**
```json
{
  "displayName": "Block Legacy Authentication",
  "state": "enabled",
  "conditions": {
    "users": { "includeUsers": ["All"] },
    "applications": { "includeApplications": ["All"] },
    "clientAppTypes": ["exchangeActiveSync","other"]
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["block"]
  }
}
```

**Policy 3 – Require Compliant Device for Microsoft 365**
```json
{
  "displayName": "Require compliant device - M365",
  "state": "enabled",
  "conditions": {
    "users": { "includeUsers": ["All"] },
    "applications": { "includeApplications": ["Office365"] }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["compliantDevice"]
  }
}
```

**Policy 4 – Block High-Risk Sign-ins (requires P2)**
```json
{
  "displayName": "Block High Risk Sign-ins",
  "state": "enabled",
  "conditions": {
    "users": { "includeUsers": ["All"] },
    "applications": { "includeApplications": ["All"] },
    "signInRiskLevels": ["high"]
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["block"]
  }
}
```

**Policy 5 – Require MFA for Risky Sign-ins (P2)**
```json
{
  "displayName": "Require MFA for Medium Risk",
  "state": "enabled",
  "conditions": {
    "users": { "includeUsers": ["All"] },
    "applications": { "includeApplications": ["All"] },
    "signInRiskLevels": ["medium","high"]
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
```

### Session Controls

| Control | Purpose |
|---|---|
| Sign-in frequency | Force re-auth every N hours/days |
| Persistent browser session | Prevent "Stay signed in?" |
| App-enforced restrictions | Pass compliance state to Exchange/SharePoint |
| Cloud App Security session | Route through CASB proxy |

---

## 4. Privileged Identity Management (PIM)

PIM provides **Just-In-Time (JIT)** privileged access, requiring users to explicitly activate elevated roles for a limited time.

### Eligible vs Active Assignments

```
Eligible Assignment:
  User has the "right" to activate the role
  Must activate → can specify justification + duration
  Activation can require MFA + approval

Active Assignment:
  User always has the role (permanent or time-bounded)
  Used for break-glass accounts
```

### Activation Workflow

```
1. User requests activation in MyAccess portal or PIM blade
2. User provides justification and selects duration (1h, 4h, 8h…)
3. If approval required → Approver notified by email
4. Approver approves/rejects
5. Role activated for requested duration
6. Role automatically deactivated after timer expires
7. All activations logged to Entra ID audit logs
```

### PIM Configuration

```bash
# Via PowerShell (Microsoft.Graph module)
# Assign eligible role
New-MgRoleManagementDirectoryRoleEligibilityScheduleRequest `
  -Action "adminAssign" `
  -PrincipalId "<user-object-id>" `
  -RoleDefinitionId "<role-definition-id>" `
  -DirectoryScopeId "/" `
  -ScheduleInfo @{
    StartDateTime = (Get-Date)
    Expiration = @{ Type = "NoExpiration" }
  }
```

### PIM Alerts

| Alert | Meaning |
|---|---|
| Roles don't require MFA at activation | Security risk |
| Roles being used outside PIM | Permanent assignment exists |
| Too many Global Admins | More than 5 Global Admins |
| Stale privileged accounts | No sign-in for 90+ days |
| Duplicate role assignments | User eligible AND active in same role |

### Access Reviews

Configure periodic reviews (quarterly recommended for privileged roles):
- **Reviewers:** Resource owner, Selected users, Manager
- **Duration:** 3–30 days
- **On no response:** Remove access (auto-apply)
- **Scope:** Eligible assignments, Active assignments, or both

---

## 5. Azure RBAC

### Azure RBAC vs Entra ID Roles

| Dimension | Azure RBAC | Entra ID Roles |
|---|---|---|
| Controls | Azure resources (VMs, storage, etc.) | Entra ID objects (users, groups, apps) |
| Scope | Management group / Subscription / RG / Resource | Directory-wide |
| Assignment store | Azure Resource Manager | Microsoft Graph |
| Examples | Owner, Contributor, Reader | Global Admin, User Admin |

### Scope Hierarchy

```
Management Group
  └── Subscription
        └── Resource Group
              └── Resource
```
Permissions assigned at a parent scope are **inherited** by all child scopes.

### Built-in Roles

| Role | Permissions |
|---|---|
| Owner | Full access + manage access |
| Contributor | Full access, cannot manage access |
| Reader | Read-only |
| User Access Administrator | Manage access only |
| AKS Cluster Admin | Get cluster admin kubeconfig |
| AKS RBAC Admin | Manage Kubernetes RBAC |
| Key Vault Secrets Officer | Read/write/delete secrets |
| Key Vault Reader | Read Key Vault metadata |
| Storage Blob Data Contributor | Read/write/delete blobs |
| Storage Blob Data Reader | Read blobs |
| ACR Pull | Pull images from ACR |
| ACR Push | Push images to ACR |
| Monitoring Contributor | Full monitoring access |
| Network Contributor | Manage networks |

### Custom Role Definition

```json
{
  "Name": "VM Operator",
  "Description": "Can start/stop/restart VMs",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>"
  ]
}
```

```bash
# Create custom role
az role definition create --role-definition vm-operator.json

# Assign role
az role assignment create \
  --assignee alice@contoso.com \
  --role "VM Operator" \
  --scope "/subscriptions/<sub-id>/resourceGroups/prod-rg"

# List assignments on a resource group
az role assignment list \
  --resource-group prod-rg \
  --output table

# Remove assignment
az role assignment delete \
  --assignee alice@contoso.com \
  --role "VM Operator" \
  --resource-group prod-rg
```

---

## 6. App Registrations & Service Principals

### The Triangle

```
App Registration          ──── defines the app (1 per tenant)
   │
   ├── Service Principal  ──── the identity in YOUR tenant
   └── Enterprise App     ──── the service principal + user consent info
```

When you register an app in Entra ID:
1. **App Registration** is created in the home tenant (defines clientId, permissions, redirect URIs)
2. A **Service Principal** is created in the same tenant
3. When a user from another tenant consents, a **Service Principal (Enterprise App)** is created in their tenant

### Client Credentials

| Credential | Rotation | Security |
|---|---|---|
| Client secret | Manual / auto expire | Lower (string that can be leaked) |
| Certificate | Manual / policy | Higher (private key stays with app) |

```bash
# Create app registration
az ad app create --display-name "my-api"

# Create service principal for the app
az ad sp create --id <app-id>

# Create service principal with RBAC (common pattern for automation)
az ad sp create-for-rbac \
  --name "github-deploy-sp" \
  --role Contributor \
  --scopes "/subscriptions/<sub-id>/resourceGroups/prod-rg" \
  --json-auth   # outputs JSON credentials
```

### OAuth 2.0 Flows

**Authorization Code Flow** (web app + user sign-in)
```
Browser → App → Entra ID: GET /authorize?response_type=code&client_id=...
Entra ID → Browser: redirect with ?code=...
App → Entra ID: POST /token with code + client_secret
Entra ID → App: { access_token, refresh_token }
```

**Client Credentials Flow** (service-to-service, no user)
```
Service → Entra ID: POST /token
  grant_type=client_credentials
  client_id=<app-id>
  client_secret=<secret>
  scope=https://management.azure.com/.default
Entra ID → Service: { access_token }
```

**Device Code Flow** (CLI / IoT device)
```
App → Entra ID: POST /devicecode  → gets device_code + user_code
App shows user: "Go to https://microsoft.com/devicelogin and enter code ABC123"
App polls: POST /token until user completes sign-in
```

**On-Behalf-Of Flow** (API calling another API)
```
User → API1 (with user access_token)
API1 → Entra ID: POST /token
  grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
  assertion=<user-access-token>
  scope=api2/.default
Entra ID → API1: new access_token for API2
API1 → API2: calls with new token
```

---

## 7. Managed Identities

Managed identities eliminate the need to store credentials in code.

### System-Assigned vs User-Assigned

| Feature | System-Assigned | User-Assigned |
|---|---|---|
| Lifecycle | Tied to resource | Independent |
| Sharing | Cannot share | Share across multiple resources |
| Use case | Single-resource scenario | Shared identity across many resources |
| Object ID | Auto-generated | Pre-created |

### Enable and Grant Access

```bash
# Enable system-assigned MI on a VM
az vm identity assign --name myvm --resource-group myrg

# Create user-assigned MI
az identity create --name my-app-identity --resource-group myrg

# Assign user-assigned MI to App Service
az webapp identity assign \
  --name my-webapp \
  --resource-group myrg \
  --identities /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/my-app-identity

# Grant MI access to Key Vault (RBAC model)
az role assignment create \
  --assignee <principal-id-of-mi> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvault
```

### Code Example (Python)

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# DefaultAzureCredential tries: env vars → managed identity → VS Code → CLI → ...
credential = DefaultAzureCredential()

client = SecretClient(
    vault_url="https://myvault.vault.azure.net",
    credential=credential
)

secret = client.get_secret("db-connection-string")
print(secret.value)
```

### Code Example (C#)

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var credential = new DefaultAzureCredential();
var client = new SecretClient(
    new Uri("https://myvault.vault.azure.net"),
    credential
);

KeyVaultSecret secret = await client.GetSecretAsync("db-connection-string");
Console.WriteLine(secret.Value);
```

---

## 8. Azure Key Vault

Key Vault is a managed secrets, keys, and certificates store with HSM backing.

### Object Types

| Type | Examples | Use Cases |
|---|---|---|
| **Secrets** | connection strings, passwords, API keys | Any sensitive string |
| **Keys** | RSA 2048/4096, EC P-256/P-384 | Encryption, signing, TLS |
| **Certificates** | X.509 certificates | TLS, code signing |

### Access Model: RBAC (Recommended)

```bash
# Create Key Vault with RBAC authorization
az keyvault create \
  --name myvault \
  --resource-group myrg \
  --location eastus \
  --enable-rbac-authorization true

# Grant secret read to a user
az role assignment create \
  --assignee alice@contoso.com \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvault

# Set a secret
az keyvault secret set \
  --vault-name myvault \
  --name "db-password" \
  --value "S3cr3tP@ssword!"

# Get a secret value
az keyvault secret show \
  --vault-name myvault \
  --name "db-password" \
  --query value -o tsv

# Create a key
az keyvault key create \
  --vault-name myvault \
  --name mykey \
  --kty RSA \
  --size 4096

# Import a certificate
az keyvault certificate import \
  --vault-name myvault \
  --name mycert \
  --file mycert.pfx \
  --password "pfxpassword"
```

### App Service Key Vault References

Inject secrets into App Service without code changes:

```bash
# In App Service Application Settings, use:
@Microsoft.KeyVault(VaultName=myvault;SecretName=db-password)

# Or with secret version:
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/db-password/abc123version)
```

The App Service's managed identity must have `Key Vault Secrets User` role.

### Soft Delete & Purge Protection

```bash
# Soft delete is ON by default (90-day retention)
# Recover a deleted secret
az keyvault secret recover --vault-name myvault --name db-password

# Enable purge protection (prevents permanent deletion for retention period)
az keyvault update --name myvault --enable-purge-protection true
```

### Private Endpoint

```bash
# Disable public access
az keyvault update --name myvault --public-network-access Disabled

# Create private endpoint
az network private-endpoint create \
  --name kv-pe \
  --resource-group myrg \
  --vnet-name myvnet \
  --subnet private-endpoints-subnet \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/myrg/providers/Microsoft.KeyVault/vaults/myvault \
  --group-ids vault \
  --connection-name myvault-connection
```

---

## 9. Microsoft Defender for Cloud

Defender for Cloud is the Azure CSPM (Cloud Security Posture Management) and workload protection platform.

### Free (CSPM) vs Defender Plans

| Capability | Free | Defender Plan |
|---|---|---|
| Secure Score | ✅ | ✅ |
| Security recommendations | ✅ | ✅ |
| Asset inventory | ✅ | ✅ |
| Threat detection | ❌ | ✅ |
| Just-in-time VM access | ❌ | ✅ |
| Adaptive application controls | ❌ | ✅ |
| File integrity monitoring | ❌ | ✅ |
| Network map | ❌ | ✅ |
| Regulatory compliance | ❌ | ✅ |

### Secure Score

```
Secure Score = (Current score of controls) / (Max score of controls) × 100

Example:
  Control "Enable MFA":         max 10 pts, currently 4 pts unhealthy resources
  Control "Apply system updates": max 6 pts, all compliant → 6 pts
  Score = 10 / 16 = 62.5%
```

### Defender Plans and What They Protect

| Plan | Protects | Key Features |
|---|---|---|
| Defender for Servers | Azure VMs, Arc servers | Threat detection, JIT, FIM, Qualys/MDE vulnerability assessment |
| Defender for Containers | AKS, ACR, Arc K8s | Image scanning, runtime protection, K8s audit logs |
| Defender for SQL | SQL in VMs, Azure SQL, Synapse | SQL injection detection, anomalous queries |
| Defender for Storage | Blob, ADLS Gen2, Files | Malware scanning, anomalous access detection |
| Defender for App Service | Azure App Service | Threat intelligence, dangling DNS detection |
| Defender for Key Vault | Key Vault | Unusual access patterns, from suspicious IPs |
| Defender for ARM | Resource Manager | Suspicious ARM operations, lateral movement |
| Defender for DNS | DNS layer | DNS exfiltration, C2 communication via DNS |

### Just-in-Time VM Access

```bash
# Enable JIT on a VM
az security jit-policy create \
  --resource-group myrg \
  --vm-name myvm \
  --port 22 \
  --max-duration "PT3H" \
  --allowed-source-address-prefixes "203.0.113.0/24"

# Request access (opens the NSG rule for your IP for 3 hours)
az security jit-policy initiate \
  --resource-group myrg \
  --vm-name myvm \
  --port 22 \
  --source-address-prefix "203.0.113.10"
```

---

## 10. Zero Trust on Azure

### Three Principles

1. **Verify Explicitly** – Always authenticate and authorize using all available data points (identity, location, device, service, workload, data classification)
2. **Use Least Privilege** – Limit access with JIT, JEA, risk-based policies, and data protection
3. **Assume Breach** – Minimize blast radius, segment access, use end-to-end encryption, use analytics to detect threats

### Six Pillars Mapped to Azure Services

| Pillar | Azure Services |
|---|---|
| **Identity** | Entra ID, PIM, Conditional Access, Identity Protection |
| **Devices** | Intune, Entra ID device compliance, Defender for Endpoint |
| **Network** | VNet, NSG, Azure Firewall, Private Endpoints, DDoS Protection |
| **Applications** | App Service auth, APIM, Defender for App Service |
| **Data** | Key Vault, AIP/Purview, Storage encryption, TDE for SQL |
| **Infrastructure** | Defender for Cloud, ASB, JIT, RBAC, Policy |

### Zero Trust Quick-Win Checklist

```
[ ] Enable MFA for all users (start with Conditional Access)
[ ] Block legacy authentication protocols
[ ] Enable Conditional Access → require compliant device for M365
[ ] Assign PIM for all privileged roles (eliminate standing access)
[ ] Enable Defender for Cloud (free tier at minimum)
[ ] Deploy Private Endpoints for Key Vault, Storage, SQL
[ ] Enable soft delete + purge protection on all Key Vaults
[ ] Review and remove unused app registrations and service principals
[ ] Enable Microsoft Defender for Identity for on-prem AD
[ ] Run access reviews quarterly for privileged roles
[ ] Enforce tag + resource lock policies via Azure Policy
[ ] Enable Azure DDoS Protection Standard on production VNets
```
