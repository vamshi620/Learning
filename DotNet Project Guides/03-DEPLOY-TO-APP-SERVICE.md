# Deploy .NET App to Azure App Service
## File 03: Your First Azure Deployment (Step-by-Step)

---

## What You'll Do in This File

By the end of this guide, your .NET app will be:
- ✅ Running live on Azure App Service with a public URL
- ✅ Connected to an Azure SQL Database
- ✅ Secrets stored securely in Key Vault
- ✅ Deployment slots for zero-downtime deployments
- ✅ Auto-scaling configured

**Time Required:** 1-2 hours

---

## What is App Service?

Azure App Service is a **fully managed web hosting platform**. You give it your code, and Azure handles:

```
You manage:                    Azure manages:
──────────                     ──────────────
✅ Your application code       ✅ Operating system
✅ Configuration               ✅ Patching & updates
✅ Scaling rules               ✅ Load balancing
✅ Custom domain (optional)    ✅ SSL/TLS certificates
                               ✅ Health monitoring
                               ✅ Auto-restart on crash
```

Think of it as: **"I give you my .NET app, you run it for me."**

---

## Step 1: Create Azure Infrastructure

```powershell
# ─── Variables (change these!) ──────────────────────────────────
$PROJECT = "myproject"
$ENV = "dev"
$RG = "$PROJECT-$ENV-rg"
$LOCATION = "eastus"

# Names for Azure resources (must be globally unique)
$APP_PLAN = "$PROJECT-$ENV-plan"
$APP_NAME = "$PROJECT-$ENV-app"          # This becomes your URL: myproject-dev-app.azurewebsites.net
$SQL_SERVER = "$PROJECT-$ENV-sql"
$SQL_DB = "$PROJECT-$ENV-db"
$KEYVAULT = "$PROJECT-$ENV-kv"
$APPINSIGHTS = "$PROJECT-$ENV-ai"

# ─── Step 1a: Create App Service Plan ──────────────────────────
# The Plan defines the VM(s) that will run your app

az appservice plan create `
  --name $APP_PLAN `
  --resource-group $RG `
  --location $LOCATION `
  --sku B1 `
  --is-linux

# sku options:
#   F1     = Free (1 GB RAM, 60 min/day CPU, no custom domains)
#   B1     = Basic ($13/month, 1.75 GB RAM, custom domains, always on)
#   S1     = Standard ($73/month, slots, auto-scale, backups)
#   P1v3   = Premium ($138/month, better perf, more slots)
```

### Understanding App Service Plans

```
App Service Plan = The "server" that runs your app

                F1 (Free)    B1 (Basic)    S1 (Standard)    P1v3 (Premium)
Price:          $0           ~$13/mo       ~$73/mo          ~$138/mo
RAM:            1 GB         1.75 GB       1.75 GB          8 GB
CPU:            Shared       1 core        1 core           2 cores
Custom domain:  ❌           ✅            ✅               ✅
SSL:            ❌           ✅            ✅               ✅
Always On:      ❌           ✅            ✅               ✅
Slots:          ❌           ❌            5 slots          20 slots
Auto-scale:     ❌           ❌            ✅ (10 instances) ✅ (30 instances)
Best for:       Learning     Dev/test      Production       High-traffic
```

> **Recommendation:** Start with **B1** for development, **S1** for production.

```powershell
# ─── Step 1b: Create the App Service ───────────────────────────

az webapp create `
  --name $APP_NAME `
  --resource-group $RG `
  --plan $APP_PLAN `
  --runtime "DOTNETCORE:8.0"

# Your app is now live at: https://myproject-dev-app.azurewebsites.net
# (It shows a default page until you deploy your code)

# ─── Step 1c: Configure App Settings ──────────────────────────

# Enable "Always On" (prevents app from sleeping)
az webapp config set `
  --name $APP_NAME `
  --resource-group $RG `
  --always-on true `
  --min-tls-version 1.2

# Set the health check path (Azure will monitor this)
az webapp config set `
  --name $APP_NAME `
  --resource-group $RG `
  --generic-configurations '{\"healthCheckPath\": \"/health/live\"}'
```

---

## Step 2: Create SQL Database

```powershell
# ─── Step 2a: Create SQL Server (logical server) ──────────────
$SQL_ADMIN = "sqladmin"
$SQL_PASS = "YourStr0ng!P@ssw0rd"    # Change this!

az sql server create `
  --name $SQL_SERVER `
  --resource-group $RG `
  --location $LOCATION `
  --admin-user $SQL_ADMIN `
  --admin-password $SQL_PASS

# ─── Step 2b: Create Database ─────────────────────────────────

az sql db create `
  --name $SQL_DB `
  --resource-group $RG `
  --server $SQL_SERVER `
  --service-objective S0 `
  --backup-storage-redundancy Local

# S0 = ~$15/month, 10 DTUs — good for dev/test
# S1 = ~$30/month, 20 DTUs — small production
# S2 = ~$75/month, 50 DTUs — medium production

# ─── Step 2c: Allow App Service to access SQL ─────────────────

# Allow Azure services (App Service) to connect
az sql server firewall-rule create `
  --name AllowAzureServices `
  --resource-group $RG `
  --server $SQL_SERVER `
  --start-ip-address 0.0.0.0 `
  --end-ip-address 0.0.0.0

# Allow your current IP (for local development)
$MY_IP = (Invoke-RestMethod -Uri "https://api.ipify.org")
az sql server firewall-rule create `
  --name AllowMyIP `
  --resource-group $RG `
  --server $SQL_SERVER `
  --start-ip-address $MY_IP `
  --end-ip-address $MY_IP

# ─── Step 2d: Get connection string ───────────────────────────

$CONN_STRING = "Server=tcp:$SQL_SERVER.database.windows.net,1433;Database=$SQL_DB;User ID=$SQL_ADMIN;Password=$SQL_PASS;Encrypt=True;TrustServerCertificate=False;"

echo "Connection String: $CONN_STRING"
```

---

## Step 3: Set Up Key Vault for Secrets

### What is Key Vault?

Key Vault is Azure's **secret safe**. Instead of putting passwords in config files, you store them in Key Vault and your app reads them securely.

```
❌ Without Key Vault:
   appsettings.json → "Password=secret123"  → Visible in source code!

✅ With Key Vault:
   Key Vault → stores "Password=secret123" → App reads at runtime
   Source code has ZERO secrets
```

```powershell
# ─── Step 3a: Create Key Vault ────────────────────────────────

az keyvault create `
  --name $KEYVAULT `
  --resource-group $RG `
  --location $LOCATION `
  --enable-rbac-authorization true

# ─── Step 3b: Store secrets in Key Vault ──────────────────────

az keyvault secret set `
  --vault-name $KEYVAULT `
  --name "ConnectionStrings--DefaultConnection" `
  --value $CONN_STRING

# Note: Use "--" instead of ":" in secret names (Key Vault doesn't allow colons)
# .NET automatically converts "--" back to ":"

# ─── Step 3c: Enable App Service to read Key Vault ────────────

# Turn on System Managed Identity for App Service
az webapp identity assign `
  --name $APP_NAME `
  --resource-group $RG

# Get the identity's Object ID
$IDENTITY_ID = az webapp identity show `
  --name $APP_NAME `
  --resource-group $RG `
  --query principalId `
  --output tsv

# Grant Key Vault Secrets User role (read-only access to secrets)
az role assignment create `
  --assignee $IDENTITY_ID `
  --role "Key Vault Secrets User" `
  --scope (az keyvault show --name $KEYVAULT --query id --output tsv)

# ─── Step 3d: Tell App Service to use Key Vault ──────────────

# Add Key Vault reference as an app setting
az webapp config appsettings set `
  --name $APP_NAME `
  --resource-group $RG `
  --settings "KeyVaultName=$KEYVAULT"
```

### Connect Key Vault in Your .NET App

```csharp
// Program.cs — Add Key Vault as a configuration source
var builder = WebApplication.CreateBuilder(args);

// In Azure, read secrets from Key Vault
if (!builder.Environment.IsDevelopment())
{
    var keyVaultName = builder.Configuration["KeyVaultName"];
    if (!string.IsNullOrEmpty(keyVaultName))
    {
        var keyVaultUri = new Uri($"https://{keyVaultName}.vault.azure.net/");
        builder.Configuration.AddAzureKeyVault(
            keyVaultUri,
            new DefaultAzureCredential());
    }
}

// Now builder.Configuration["ConnectionStrings:DefaultConnection"]
// automatically reads from Key Vault in Azure, or appsettings locally!
```

```powershell
# Required NuGet packages:
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Identity
```

---

## Step 4: Deploy Your App

### Option A: Deploy from Visual Studio (Easiest)

```
1. Right-click your project → Publish
2. Target: Azure → Azure App Service (Linux)
3. Select your subscription and App Service
4. Click Publish
```

### Option B: Deploy from CLI (Recommended)

```powershell
# From your project directory:

# Build and create a ZIP package
dotnet publish -c Release -o ./publish
Compress-Archive -Path ./publish/* -DestinationPath ./deploy.zip -Force

# Deploy the ZIP to App Service
az webapp deploy `
  --name $APP_NAME `
  --resource-group $RG `
  --src-path ./deploy.zip `
  --type zip
```

### Option C: Deploy from Git (Best for Teams)

```powershell
# Configure App Service for local Git deployment
az webapp deployment source config-local-git `
  --name $APP_NAME `
  --resource-group $RG

# Get the Git URL
$GIT_URL = az webapp deployment source config-local-git `
  --name $APP_NAME `
  --resource-group $RG `
  --query url `
  --output tsv

# Add as a Git remote and push
git remote add azure $GIT_URL
git push azure main
```

---

## Step 5: Verify Your Deployment

```powershell
# Check your app is running
$APP_URL = "https://$APP_NAME.azurewebsites.net"

# Test the health endpoint
curl "$APP_URL/health/live"
# Expected: Healthy

# Test the ready endpoint (verifies DB connection)
curl "$APP_URL/health/ready"
# Expected: Healthy

# View application logs (streaming)
az webapp log tail `
  --name $APP_NAME `
  --resource-group $RG

# Open the app in your browser
Start-Process $APP_URL
```

---

## Step 6: Set Up Application Insights (Monitoring)

```powershell
# Create Application Insights
az monitor app-insights component create `
  --app $APPINSIGHTS `
  --resource-group $RG `
  --location $LOCATION `
  --application-type web

# Get the instrumentation key
$AI_KEY = az monitor app-insights component show `
  --app $APPINSIGHTS `
  --resource-group $RG `
  --query instrumentationKey `
  --output tsv

$AI_CONN = az monitor app-insights component show `
  --app $APPINSIGHTS `
  --resource-group $RG `
  --query connectionString `
  --output tsv

# Connect to App Service
az webapp config appsettings set `
  --name $APP_NAME `
  --resource-group $RG `
  --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONN"

# Enable Application Insights in your .NET app:
# dotnet add package Microsoft.ApplicationInsights.AspNetCore
# In Program.cs: builder.Services.AddApplicationInsightsTelemetry();
```

---

## Step 7: Deployment Slots (Zero-Downtime Deployments)

### What Are Slots?

Deployment slots let you deploy and test a new version **before** swapping it to production. Zero downtime!

```
App Service: myproject-dev-app
├── Production slot    ← https://myproject-dev-app.azurewebsites.net (live users)
└── Staging slot       ← https://myproject-dev-app-staging.azurewebsites.net (testing)

Deployment process:
1. Deploy new code → Staging slot
2. Test on staging URL
3. "Swap" staging ↔ production (instant, zero downtime)
4. If something's wrong → Swap back (instant rollback)
```

```powershell
# Requires Standard (S1) or higher plan

# Create a staging slot
az webapp deployment slot create `
  --name $APP_NAME `
  --resource-group $RG `
  --slot staging

# Deploy to staging (not production!)
az webapp deploy `
  --name $APP_NAME `
  --resource-group $RG `
  --slot staging `
  --src-path ./deploy.zip `
  --type zip

# Test staging
curl "https://$APP_NAME-staging.azurewebsites.net/health/live"

# Swap staging → production (zero downtime!)
az webapp deployment slot swap `
  --name $APP_NAME `
  --resource-group $RG `
  --slot staging `
  --target-slot production

# If something's wrong, swap back:
az webapp deployment slot swap `
  --name $APP_NAME `
  --resource-group $RG `
  --slot staging `
  --target-slot production
```

---

## Step 8: Configure Auto-Scaling

```powershell
# Requires Standard (S1) or higher plan

# Scale out based on CPU usage
az monitor autoscale create `
  --name "$APP_NAME-autoscale" `
  --resource-group $RG `
  --resource $APP_NAME `
  --resource-type "Microsoft.Web/serverFarms" `
  --min-count 1 `
  --max-count 5 `
  --count 1

# Add rule: Scale OUT when CPU > 70% for 5 minutes
az monitor autoscale rule create `
  --resource-group $RG `
  --autoscale-name "$APP_NAME-autoscale" `
  --condition "CpuPercentage > 70 avg 5m" `
  --scale out 1

# Add rule: Scale IN when CPU < 30% for 10 minutes
az monitor autoscale rule create `
  --resource-group $RG `
  --autoscale-name "$APP_NAME-autoscale" `
  --condition "CpuPercentage < 30 avg 10m" `
  --scale in 1
```

---

## Step 9: Custom Domain (Optional)

```powershell
# 1. Add custom domain
az webapp config hostname add `
  --webapp-name $APP_NAME `
  --resource-group $RG `
  --hostname "api.yourcompany.com"

# 2. Create free managed SSL certificate
az webapp config ssl create `
  --name $APP_NAME `
  --resource-group $RG `
  --hostname "api.yourcompany.com"

# 3. Bind SSL certificate
az webapp config ssl bind `
  --name $APP_NAME `
  --resource-group $RG `
  --certificate-thumbprint "<thumbprint-from-step-2>" `
  --ssl-type SNI

# Don't forget to add a CNAME record in your DNS:
# CNAME: api.yourcompany.com → myproject-dev-app.azurewebsites.net
```

---

## ✅ Deployment Checklist

- [ ] App Service created and running
- [ ] SQL Database created and firewall configured
- [ ] Key Vault created with secrets stored
- [ ] App Service has Managed Identity with Key Vault access
- [ ] App deployed and health endpoints responding 200
- [ ] Application Insights connected
- [ ] Staging slot created (production only)
- [ ] Auto-scaling configured (production only)

---

## 📊 Cost Estimate (Dev Environment)

| Resource | SKU | Monthly Cost |
|----------|-----|-------------|
| App Service Plan | B1 | ~$13 |
| SQL Database | S0 | ~$15 |
| Key Vault | Standard | ~$0.03/10K ops |
| Application Insights | Free tier | $0 (first 5 GB) |
| **Total (Dev)** | | **~$28/month** |

---

> **Next Step:**
> - Set up CI/CD → [05-CI-CD-PIPELINE-SETUP.md](05-CI-CD-PIPELINE-SETUP.md)
> - Or also deploy to AKS → [04-DEPLOY-TO-AKS-STEP-BY-STEP.md](04-DEPLOY-TO-AKS-STEP-BY-STEP.md)
