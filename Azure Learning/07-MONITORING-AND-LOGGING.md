# Monitoring, Logging, and Observability
## Document 7: Production Visibility and Troubleshooting

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Monitoring metrics, structured logging, and observability

---

## TABLE OF CONTENTS

1. [Monitoring Fundamentals](#monitoring)
2. [Azure Monitor Integration](#azure-monitor)
3. [Application Insights](#app-insights)
4. [Structured Logging](#logging)
5. [Kubernetes Monitoring](#k8s-monitoring)
6. [Alerts and Notifications](#alerts)
7. [Cost Monitoring](#cost-monitoring)

---

## Monitoring Fundamentals {#monitoring}

### What is Observability?

**Observability** = Ability to understand application state from its outputs

```
Traditional Monitoring (Reactive):
├─ Application crashes
├─ Alert triggers
├─ On-call engineer wakes up
├─ Debug to find root cause
└─ 30 minutes down

Observability (Proactive):
├─ Error rate increases 1%
├─ Alert before crash
├─ Engineer investigates
├─ Issue fixed before users notice
└─ Zero downtime
```

### Three Pillars of Observability

```
        ┌─────────────────────────────┐
        │    Observability            │
        └──────────┬──────────────────┘
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
    ┌────────┐ ┌──────┐ ┌─────────┐
    │ Logs   │ │Traces│ │Metrics  │
    └────────┘ └──────┘ └─────────┘
        │          │          │
    ┌─────────┬─────────┬─────────┐
    ↓ Event   ↓ Request ↓ Number  │
    │ logs    │ flows   │ values  │
    │ What    │ How     │ When    │
    │ happened│ it      │ things  │
    │         │ got     │ changed │
    │         │ there   │         │
    └─────────┴─────────┴─────────┘
```

---

## Azure Monitor Integration {#azure-monitor}

### Setting Up Azure Monitor

```bash
# Enable monitoring on AKS cluster (if not done)
az aks enable-addons \
  --addon monitoring \
  --name azurelearn-dev-aks \
  --resource-group azure-learn-rg-dev

# Create Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group azure-learn-rg-dev \
  --workspace-name azurelearn-logs

# Verify
az monitor log-analytics workspace show \
  --resource-group azure-learn-rg-dev \
  --workspace-name azurelearn-logs
```

### Monitor Metrics

```kusto
// KQL (Kusto Query Language) - Query metrics in Azure Monitor

// 1. CPU Usage Over Time
Perf
| where ObjectName == "K8SContainer"
| where CounterName == "cpuUsageNanoCores"
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 1m)
| render timechart

// 2. Memory Usage
Perf
| where ObjectName == "K8SContainer"
| where CounterName == "memoryWorkingSetBytes"
| summarize AvgMemory = avg(CounterValue) by bin(TimeGenerated, 1m)
| render timechart

// 3. Pod Restart Count
KubePodInventory
| where Namespace == "azure-learn-app"
| summarize TotalRestarts = sum(RestartCount) by PodName

// 4. Node Status
KubeNodeInventory
| where ClusterName == "azurelearn-dev-aks"
| project TimeGenerated, Computer, Status, Cpu = tostring(CpuCapacity), Memory = tostring(MemoryCapacity)
| sort by TimeGenerated desc

// 5. Failed Pods
KubePodInventory
| where Namespace == "azure-learn-app"
| where PodStatus != "Running"
| project TimeGenerated, PodName, PodStatus, ContainerStatus
```

### Alert Rules

```bash
# Create alert for high CPU (>80%)
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group azure-learn-rg-dev \
  --resource-type "Microsoft.ContainerService/managedClusters" \
  --resource "azurelearn-dev-aks" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action create-or-update \
  --action-group-name "azure-learn-alerts" \
  --description "Alert when CPU > 80%"

# Create alert for pod failures
az monitor log-search alert create \
  --resource-group azure-learn-rg-dev \
  --name "Pod Failure Alert" \
  --description "Alert when pod fails" \
  --scopes "/subscriptions/subscription-id/resourcegroups/azure-learn-rg-dev" \
  --criteria-query "KubePodInventory | where PodStatus != 'Running'" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group "/subscriptions/.../resourceGroups/.../providers/microsoft.insights/actiongroups/..."
```

---

## Application Insights {#app-insights}

### Configuration in Code

```csharp
// In Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
});

// Custom instrumentation
builder.Services.AddScoped<ITelemetryClient>(sp =>
{
    var client = sp.GetRequiredService<TelemetryClient>();
    return new AppInsightsTelemetry(client);
});
```

### Custom Events Logging

```csharp
public class ProductsController : ControllerBase
{
    private readonly ITelemetryClient _telemetry;
    
    [HttpPost("api/products")]
    public async Task<IActionResult> CreateProduct(Product product)
    {
        try
        {
            // Track event
            _telemetry.TrackEvent("ProductCreatedStarted", new Dictionary<string, string>
            {
                ["ProductId"] = product.Id,
                ["UserId"] = User.FindFirst("sub")?.Value,
                ["Timestamp"] = DateTime.UtcNow.ToString()
            });
            
            // Create product
            await _cosmosDbService.CreateItemAsync(product);
            
            // Track success
            _telemetry.TrackEvent("ProductCreatedSuccess", new Dictionary<string, string>
            {
                ["ProductId"] = product.Id,
                ["Price"] = product.Price.ToString()
            });
            
            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
        }
        catch (Exception ex)
        {
            // Track exception
            _telemetry.TrackException(ex, new Dictionary<string, string>
            {
                ["ProductId"] = product.Id,
                ["Operation"] = "CreateProduct"
            });
            
            return StatusCode(500);
        }
    }
}
```

### Application Map

```
Application Map visualizes dependencies:

┌──────────────────────────────┐
│   Client/Browser             │
│   (Requests monitored)       │
└──────────────┬───────────────┘
               │ ~100ms latency
               ↓
┌──────────────────────────────┐
│   Azure App Service          │
│   (Your API)                 │
│   - Response time: 45ms      │
│   - Success rate: 99.8%      │
│   - Request rate: 1000/min   │
└──────────┬───────────┬───────┘
           │           │
      ~20ms│           │~30ms
           ↓           ↓
    ┌────────────┐  ┌──────────────┐
    │ Cosmos DB  │  │ Key Vault    │
    │ Success:   │  │ Success:     │
    │ 100%       │  │ 100%         │
    │ Latency:   │  │ Latency:     │
    │ 15ms       │  │ 5ms          │
    └────────────┘  └──────────────┘
```

### Performance Metrics Query

```kusto
// Analyze API performance

requests
| where customDimensions.["http.method"] == "GET"
| where url contains "api/products"
| summarize
    TotalRequests = count(),
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 0.95),
    P99Duration = percentile(duration, 0.99),
    SuccessRate = (todouble(sumif(1, success == true)) / count()) * 100
    by bin(timestamp, 1m)
| render timechart
```

### Exception Analysis

```kusto
// Find most common exceptions

exceptions
| where cloud_RoleName == "azure-learn-app"
| summarize
    Count = count(),
    LastOccurrence = max(timestamp)
    by outerType, outerMessage
| sort by Count desc
| take 10
```

---

## Structured Logging {#logging}

### Serilog Configuration

```csharp
// In Program.cs
builder.Host.UseSerilog((context, loggerConfig) =>
{
    loggerConfig
        .MinimumLevel.Information()
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithThreadId()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console(new JsonFormatter())  // JSON to stdout
        .WriteTo.ApplicationInsights(
            new TelemetryConfiguration(connectionString),
            TelemetryConverter.Traces
        )
        .WriteTo.File(
            "logs/app-.txt",
            rollingInterval: RollingInterval.Day,
            outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message:lj}{NewLine}{Exception}"
        );
});
```

### Structured Log Example

```csharp
public class ProductService : IProductService
{
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(ILogger<ProductService> logger)
    {
        _logger = logger;
    }
    
    public async Task CreateProductAsync(Product product, string userId)
    {
        using var activity = new Activity("CreateProduct").Start();
        
        _logger.LogInformation(
            "Creating product for user {UserId}: {@Product}",
            userId,
            product  // Structured: logs entire object
        );
        
        try
        {
            var stopwatch = Stopwatch.StartNew();
            
            await _cosmosDbService.CreateItemAsync(product);
            
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Product created successfully. ProductId={ProductId}, Duration={ElapsedMs}ms, UserId={UserId}",
                product.Id,
                stopwatch.ElapsedMilliseconds,
                userId
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Failed to create product. ProductId={ProductId}, UserId={UserId}, Exception={ExceptionType}",
                product.Id,
                userId,
                ex.GetType().Name
            );
            
            throw;
        }
    }
}
```

### Log Analysis Query

```kusto
// Find slow database operations

traces
| where message contains "database"
| where customDimensions.["Duration"] > 1000  // > 1 second
| summarize
    Count = count(),
    AvgDuration = avg(todouble(customDimensions.["Duration"])),
    MaxDuration = max(todouble(customDimensions.["Duration"]))
    by operation_Name
| sort by AvgDuration desc
```

---

## Kubernetes Monitoring {#k8s-monitoring}

### Metrics Server

```bash
# Check if metrics server installed
kubectl get deployment metrics-server -n kube-system

# View pod metrics
kubectl top pods -n azure-learn-app

# Output:
# NAME                                   CPU(cores)   MEMORY(Mi)
# azure-learn-app-5cf6697495-dc45l       45m          256Mi
# azure-learn-app-5cf6697495-nlg4x       42m          243Mi

# View node metrics
kubectl top nodes

# Output:
# NAME                                STATUS   ROLES   CPU(cores)   CPU%      MEMORY(Mi)   MEMORY%
# aks-nodepool1-12345678-vmss000000   Ready    agent   234m         11%       1024Mi       40%
# aks-nodepool1-12345678-vmss000001   Ready    agent   198m         9%        987Mi        38%
```

### Dashboard Deployment

```bash
# Install Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin user
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Access dashboard
kubectl proxy

# Navigate to:
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# Get token for login
kubectl -n kubernetes-dashboard create token admin-user
```

### Prometheus and Grafana (Advanced)

```bash
# Install Prometheus Helm chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Install Grafana dashboard
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring

# Access Grafana at http://localhost:3000
# Default credentials: admin / prom-operator
```

---

## Alerts and Notifications {#alerts}

### Action Groups (Slack Integration)

```bash
# Create Slack webhook
# In Slack: Apps → Search "Incoming Webhooks" → Add → Copy webhook URL

# Create Action Group
az monitor action-group create \
  --name "azure-learn-alerts" \
  --resource-group azure-learn-rg-dev

# Add Slack notification
az monitor action-group webhook-receiver add \
  --action-group-name "azure-learn-alerts" \
  --resource-group azure-learn-rg-dev \
  --name "SlackNotification" \
  --service-uri "https://hooks.slack.com/services/YOUR/WEBHOOK/URL" \
  --use-common-alert-schema true
```

### Alert Types

```
1. Availability Alerts
   - Application down
   - Health check failing
   - Response time > threshold
   
2. Performance Alerts
   - High CPU (>80%)
   - High memory (>90%)
   - Request duration spike
   
3. Error Alerts
   - Exception rate > threshold
   - Failed dependencies
   - Database connection errors
   
4. Resource Alerts
   - Storage near limit
   - Quota exceeded
   - Cost threshold exceeded

5. Custom Alerts
   - Business metrics
   - SLA violations
   - User-defined thresholds
```

### Sample Alert Query

```kusto
// Alert: Error rate > 5% in last 5 minutes

requests
| where timestamp > ago(5m)
| summarize
    TotalRequests = count(),
    FailedRequests = sumif(1, success == false),
    ErrorRate = (todouble(sumif(1, success == false)) / count()) * 100
| where ErrorRate > 5
```

---

## Cost Monitoring {#cost-monitoring}

### Azure Cost Management

```bash
# Get current month spending
az costmanagement query \
  --timeframe MonthToDate \
  --type "Usage" \
  --dataset dataset[
    {name: "PreTaxCost"}
  ]

# Monitor specific resource group
az costmanagement query \
  --scope "subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev" \
  --timeframe MonthToDate \
  --type Usage \
  --dataset '{
    "granularity": "Daily",
    "aggregation": {
      "totalCost": {"name": "PreTaxCost", "function": "Sum"}
    },
    "grouping": [
      {"type": "Dimension", "name": "ResourceType"}
    ]
  }'
```

### Cost Breakdown by Service

```
Azure Cost Analysis (Portal):

AKS Cluster:              $120/month
├─ 2 nodes (D2s_v3)       $80
├─ Load balancer          $20
├─ Storage                $10
└─ Bandwidth              $10

Cosmos DB:                $50/month (serverless)
├─ Request Units (RUs)    $30
├─ Storage (1 GB)         $10
└─ Backups                $10

Container Registry:       $10/month
├─ Standard tier          $10

Key Vault:                $1/month
├─ Operations             $1

Virtual Network:          Free
├─ VNet itself            Free
├─ NAT Gateway            $30 (if used)

Total: ~$180/month

Ways to reduce costs:
1. Use reserved instances for AKS (-30%)
2. Use spot VMs for non-critical (-70%)
3. Optimize Cosmos DB RUs
4. Clean up unused resources
5. Use dev/prod tiers strategically
```

---

## Observability Checklist

```
✅ Monitoring
├─ Azure Monitor configured
├─ Metrics being collected
├─ Alerts defined
└─ Action groups connected

✅ Logging
├─ Structured logs implemented
├─ Logs indexed in Log Analytics
├─ Retention policy set
└─ Sensitive data redacted

✅ Tracing
├─ Application Insights enabled
├─ Custom instrumentation added
├─ Distributed tracing working
└─ Performance tracked

✅ Dashboards
├─ Custom dashboards created
├─ Key metrics visible
├─ Alerts displayed
└─ Team has access

✅ Alerting
├─ Critical alerts defined
├─ Escalation policy set
├─ On-call rotation configured
└─ Runbooks available

✅ Cost Monitoring
├─ Budget alerts configured
├─ Cost analysis reviewed
├─ Anomaly detection enabled
└─ Optimization recommendations reviewed
```

---

## Key Takeaways

✅ **Observability:** Logs + Traces + Metrics = Understanding  
✅ **Structured Logging:** Use JSON format with context  
✅ **Metrics:** Track performance, capacity, errors  
✅ **Alerts:** Proactive notifications before users notice  
✅ **Dashboards:** Visibility into system health  
✅ **Cost Monitoring:** Track spending and optimize  

**Next Document:** Document 8 covers security best practices and optimization strategies.
