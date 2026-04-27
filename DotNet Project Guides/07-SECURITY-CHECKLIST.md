# Security Checklist for .NET Azure Projects
## File 07: Protect Your App and Infrastructure

---

## What You'll Do in This File

By the end of this guide, your project will have:
- ✅ No secrets in code or config files
- ✅ Managed Identity (passwordless authentication to Azure services)
- ✅ HTTPS enforced everywhere
- ✅ Network security configured
- ✅ A security review checklist for every deployment

**Time Required:** 1 hour

---

## The #1 Rule: No Secrets in Code

```
❌ NEVER DO THIS:
   var connString = "Server=mydb.database.windows.net;Password=secret123";
   var apiKey = "sk-abc123def456";

✅ ALWAYS DO THIS:
   var connString = builder.Configuration.GetConnectionString("DefaultConnection");
   // Value comes from Key Vault / environment variable / managed identity
```

---

## Security Checklist

### 🔑 Secrets Management

| # | Check | How |
|---|-------|-----|
| 1 | ❌ No secrets in source code | Grep your repo for "password", "secret", "key", "connectionstring" |
| 2 | ❌ No secrets in appsettings.json | Use User Secrets locally, Key Vault in Azure |
| 3 | ✅ Secrets in Azure Key Vault | Store all connection strings, API keys, certificates |
| 4 | ✅ .gitignore excludes secret files | `.env`, `*.pfx`, `appsettings.*.local.json` |

### 🆔 Identity & Authentication

| # | Check | How |
|---|-------|-----|
| 5 | ✅ Managed Identity enabled | App Service / AKS uses MI to access Azure services |
| 6 | ✅ DefaultAzureCredential in code | Works locally (dev creds) AND in Azure (managed identity) |
| 7 | ✅ No shared passwords between envs | Each environment has its own secrets |
| 8 | ✅ Minimum permissions (least privilege) | Grant only what's needed (e.g., "Key Vault Secrets User" not "Key Vault Administrator") |

### Using DefaultAzureCredential

```csharp
using Azure.Identity;

// This single class works everywhere:
// - Local dev: Uses your az login credentials
// - Azure App Service: Uses Managed Identity
// - Azure AKS: Uses Workload Identity
// - Azure DevOps: Uses Service Connection
var credential = new DefaultAzureCredential();

// Connect to Key Vault
var client = new SecretClient(
    new Uri("https://myvault.vault.azure.net/"),
    credential);

// Connect to Blob Storage
var blobClient = new BlobServiceClient(
    new Uri("https://mystorage.blob.core.windows.net/"),
    credential);

// Connect to SQL Database (passwordless!)
// Connection string: "Server=mydb.database.windows.net;Database=mydb;Authentication=Active Directory Default;"
```

### 🌐 Network Security

| # | Check | How |
|---|-------|-----|
| 9 | ✅ HTTPS only (HTTP disabled) | App Service: set `--min-tls-version 1.2` and force HTTPS redirect |
| 10 | ✅ SQL Firewall configured | Only allow App Service / AKS IPs (not 0.0.0.0/0) |
| 11 | ✅ Key Vault network restricted | Premium tier: use private endpoints |
| 12 | ✅ No public access to internal services | Use internal load balancers in AKS for service-to-service |

```powershell
# Force HTTPS on App Service
az webapp update --name $APP_NAME --resource-group $RG --https-only true

# Set minimum TLS version
az webapp config set --name $APP_NAME --resource-group $RG --min-tls-version 1.2
```

### 📦 Application Security

| # | Check | How |
|---|-------|-----|
| 13 | ✅ Input validation on all endpoints | Use data annotations, FluentValidation |
| 14 | ✅ CORS configured (not wildcard *) | Allow only your frontend domain |
| 15 | ✅ Rate limiting enabled | Use ASP.NET Rate Limiting middleware |
| 16 | ✅ Security headers set | HSTS, X-Content-Type-Options, X-Frame-Options |
| 17 | ✅ Docker container runs as non-root | `USER app` in Dockerfile |
| 18 | ✅ Dependencies scanned for vulnerabilities | `dotnet list package --vulnerable` |

```csharp
// Program.cs — Security headers middleware
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    await next();
});

// CORS — allow only your frontend
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins("https://myapp.contoso.com")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 10;
    });
});
```

### 🔍 Monitoring & Audit

| # | Check | How |
|---|-------|-----|
| 19 | ✅ Application Insights enabled | Track all requests, errors, dependencies |
| 20 | ✅ Diagnostic logging enabled | Azure Activity Log for infrastructure changes |
| 21 | ✅ Failed login attempts monitored | Alert on repeated 401/403 responses |
| 22 | ✅ Resource locks on production | Prevent accidental deletion |

```powershell
# Prevent accidental deletion of production resources
az lock create `
  --name "no-delete" `
  --resource-group "myproject-prod-rg" `
  --lock-type CanNotDelete `
  --notes "Production resources — do not delete"
```

---

## Quick Security Scan Commands

```powershell
# Check for vulnerable NuGet packages
dotnet list package --vulnerable

# Scan your codebase for secrets (install gitleaks)
# winget install Gitleaks
gitleaks detect --source .

# Check Docker image for vulnerabilities (install trivy)
# winget install aquasecurity.trivy
trivy image myprojectdevacr.azurecr.io/myapi:latest
```

---

## ✅ Security Sign-Off Checklist

Before going to production, verify every item:

- [ ] No secrets in Git (run `gitleaks detect`)
- [ ] Managed Identity used for all Azure service connections
- [ ] HTTPS only, TLS 1.2+
- [ ] SQL/Key Vault firewall restricted
- [ ] Docker container runs as non-root
- [ ] NuGet packages scanned (`dotnet list package --vulnerable`)
- [ ] CORS configured with specific origins
- [ ] Resource locks on production RG
- [ ] Application Insights collecting telemetry
- [ ] Activity Log enabled for audit trail

---

> **Next Step:** Manage costs → [08-COST-MANAGEMENT-GUIDE.md](08-COST-MANAGEMENT-GUIDE.md)
