# Monitoring and Debugging in Azure
## File 06: Know What's Happening in Production

---

## What You'll Do in This File

By the end of this guide, you'll have:
- ✅ Application Insights collecting telemetry
- ✅ Alerts on errors and performance issues
- ✅ A dashboard for your team
- ✅ Skills to debug live production issues

**Time Required:** 1 hour

---

## What is Application Insights?

Application Insights automatically collects:

```
Your .NET App → Application Insights
                ├── Every HTTP request (URL, duration, status code)
                ├── Every exception (stack trace, context)
                ├── Every dependency call (SQL queries, HTTP calls, Redis)
                ├── Custom events you log
                ├── Performance counters (CPU, memory, GC)
                └── Live metrics (real-time dashboard)
```

---

## Step 1: Add Application Insights to Your .NET App

```powershell
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Program.cs — Add ONE line
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationInsightsTelemetry();  // ← This line!

// That's it. Application Insights will:
// - Auto-track all HTTP requests
// - Auto-track all SQL/HTTP dependencies
// - Auto-track all exceptions
// - Auto-track performance counters
```

Set the connection string in your app settings:
```json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=xxx;IngestionEndpoint=https://..."
  }
}
```

In Azure, set it as an environment variable:
```powershell
# App Service
az webapp config appsettings set --name myapp --resource-group myRG `
  --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=<your-connection-string>"

# AKS (add to ConfigMap)
# APPLICATIONINSIGHTS_CONNECTION_STRING: "<your-connection-string>"
```

---

## Step 2: Set Up Alerts

### Alert on High Error Rate

```powershell
# Alert when error rate exceeds 5% in 5 minutes
az monitor metrics alert create `
  --name "high-error-rate" `
  --resource-group $RG `
  --scopes (az monitor app-insights component show --app $APPINSIGHTS -g $RG --query id -o tsv) `
  --condition "avg requests/failed > 5" `
  --window-size 5m `
  --evaluation-frequency 1m `
  --severity 2 `
  --description "Error rate is above 5%"
```

### Alert on Slow Responses

```powershell
# Alert when average response time exceeds 2 seconds
az monitor metrics alert create `
  --name "slow-responses" `
  --resource-group $RG `
  --scopes (az monitor app-insights component show --app $APPINSIGHTS -g $RG --query id -o tsv) `
  --condition "avg requests/duration > 2000" `
  --window-size 5m `
  --severity 3 `
  --description "Average response time exceeds 2 seconds"
```

---

## Step 3: Useful Queries (KQL)

In Azure Portal → Application Insights → Logs, paste these queries:

### Find All Errors in the Last Hour

```kusto
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by name, resultCode
| order by count_ desc
```

### Slowest Endpoints

```kusto
requests
| where timestamp > ago(24h)
| summarize avg(duration), percentile(duration, 95), count() by name
| where count_ > 10
| order by percentile_duration_95 desc
| take 10
```

### Recent Exceptions with Stack Traces

```kusto
exceptions
| where timestamp > ago(4h)
| project timestamp, type, outerMessage, details = details[0].rawStack
| order by timestamp desc
| take 20
```

### Dependency Failures (DB, APIs, Redis)

```kusto
dependencies
| where timestamp > ago(1h)
| where success == false
| summarize count() by name, type, resultCode
| order by count_ desc
```

---

## Step 4: Debug Live Issues

### For App Service

```powershell
# Stream live logs
az webapp log tail --name $APP_NAME --resource-group $RG

# Download logs
az webapp log download --name $APP_NAME --resource-group $RG --log-file ./logs.zip

# Open the Kudu console (advanced debugging)
Start-Process "https://$APP_NAME.scm.azurewebsites.net"
```

### For AKS

```powershell
# View pod logs
kubectl logs -n myproject <pod-name> -f

# View logs from ALL pods of a deployment
kubectl logs -n myproject -l app=myapi --all-containers=true

# Check pod events (restarts, OOM kills, pull errors)
kubectl describe pod -n myproject <pod-name>

# Check if pods are restarting
kubectl get pods -n myproject -w    # Watch mode

# Shell into a running container
kubectl exec -it -n myproject <pod-name> -- sh
```

---

## Step 5: Create a Team Dashboard

In Azure Portal → Dashboard → New Dashboard:

**Recommended widgets:**

| Widget | Shows | Source |
|--------|-------|--------|
| Server response time | Avg response time trend | App Insights |
| Failed requests | Error count trend | App Insights |
| Server requests | Request volume | App Insights |
| Availability | Uptime percentage | App Insights |
| CPU % (App Service) | CPU utilization | App Service |
| Memory % (App Service) | Memory utilization | App Service |
| Pod count (AKS) | Running pods | AKS |
| Node CPU (AKS) | Node utilization | AKS |

---

## ✅ Monitoring Checklist

- [ ] Application Insights SDK added to .NET app
- [ ] Connection string configured in Azure
- [ ] Alert on error rate > 5%
- [ ] Alert on response time > 2 seconds
- [ ] Team dashboard created
- [ ] Team knows how to stream logs

---

> **Next Step:** Secure your app → [07-SECURITY-CHECKLIST.md](07-SECURITY-CHECKLIST.md)
