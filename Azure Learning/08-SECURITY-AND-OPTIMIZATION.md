# Security, Optimization, and Best Practices
## Document 8: Hardening, Performance, and Production Excellence

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Security implementation, performance optimization, and production readiness

---

## TABLE OF CONTENTS

1. [Security Foundations](#security)
2. [Authentication & Authorization](#auth)
3. [Network Security](#network-security)
4. [Data Protection](#data-protection)
5. [Performance Optimization](#optimization)
6. [Scalability](#scalability)
7. [Production Checklist](#checklist)

---

## Security Foundations {#security}

### Security Layers

```
            ┌─────────────────────────┐
            │  Application Security   │
            │  (Input validation,     │
            │   Authorization)        │
            └────────────┬────────────┘
                         │
            ┌────────────┴────────────┐
            │  Network Security       │
            │  (NSG, WAF, Firewall)   │
            └────────────┬────────────┘
                         │
            ┌────────────┴────────────┐
            │  Infrastructure Security│
            │  (RBAC, Encryption)     │
            └────────────┬────────────┘
                         │
            ┌────────────┴────────────┐
            │  Data Security          │
            │  (Encryption at rest,   │
            │   in transit)           │
            └─────────────────────────┘
```

### Threat Model

```
Potential Threats:

1. Unauthorized Access
   ├─ Attacker gains credentials
   ├─ Accesses Key Vault
   └─ Exfiltrates data
   
   Mitigation:
   ├─ Managed Identity (no secrets!)
   ├─ RBAC (least privilege)
   └─ Multi-factor authentication

2. Network Attack
   ├─ Attacker intercepts traffic
   └─ Reads sensitive data
   
   Mitigation:
   ├─ TLS/SSL encryption
   ├─ Network policies
   └─ Private endpoints

3. Container Escape
   ├─ Attacker breaks out of container
   └─ Accesses host
   
   Mitigation:
   ├─ Non-root user
   ├─ Read-only filesystem
   └─ Security context

4. Supply Chain Attack
   ├─ Attacker modifies base image
   └─ Injects malware
   
   Mitigation:
   ├─ Image scanning
   ├─ Signed images
   └─ Vulnerability management

5. Data Breach
   ├─ Attacker accesses database
   └─ Steals customer data
   
   Mitigation:
   ├─ Encryption at rest
   ├─ Access control
   └─ Audit logging
```

---

## Authentication & Authorization {#auth}

### Managed Identity (Passwordless)

```
Traditional Approach ❌:
┌──────────────────────────────┐
│ Pod needs database access    │
├──────────────────────────────┤
│ 1. Store password in secret  │
│ 2. Load secret in pod        │
│ 3. Connect using password    │
│                              │
│ Problem:                     │
│ - Secret in YAML file       │
│ - Secret in Kubernetes      │
│ - Secret in pod memory      │
│ - Someone can find it!      │
└──────────────────────────────┘

Managed Identity Approach ✅:
┌──────────────────────────────┐
│ Pod (azure-learn-app-sa)     │
├──────────────────────────────┤
│ 1. Kubernetes pod identity   │
│ 2. Auto token refresh        │
│ 3. No passwords stored       │
│                              │
│ Benefit:                     │
│ - Zero secrets stored        │
│ - Auto credential rotation   │
│ - Audit trail                │
│ - Least privilege RBAC       │
└──────────────────────────────┘
```

### RBAC Implementation

```yaml
# ServiceAccount for application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-learn-app-sa
  namespace: azure-learn-app
  annotations:
    # Link to Managed Identity
    azure.workload.identity/client-id: "bf9ea53d-6351-42bf-aa45-3328a0bd296f"

---
# Role: What can do what
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: azure-learn-app-role
  namespace: azure-learn-app
rules:
# Can read ConfigMaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# Can read secrets
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

# CANNOT delete pods, create resources, etc.

---
# RoleBinding: Connect SA to Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: azure-learn-app-binding
  namespace: azure-learn-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: azure-learn-app-role
subjects:
- kind: ServiceAccount
  name: azure-learn-app-sa
  namespace: azure-learn-app
```

### Azure RBAC (Control Plane)

```bash
# Who can do what in Azure (not Kubernetes)

# See current roles
az role assignment list \
  --scope /subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev

# Assign role
az role assignment create \
  --assignee "developer@company.com" \
  --role "Contributor" \
  --scope /subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev

# Built-in roles
# - Owner: Full access
# - Contributor: Create/manage resources
# - Reader: Read-only access
# - Operator: Manage resources (no create/delete)

# Custom role (principle of least privilege)
az role definition create --role-definition '{
  "Name": "AKS Deployer",
  "Description": "Can deploy to AKS only",
  "AssignableScopes": ["/subscriptions/subscription-id"],
  "Permissions": [{
    "Actions": [
      "Microsoft.ContainerService/managedClusters/*/read",
      "Microsoft.ContainerService/managedClusters/*/write"
    ],
    "NotActions": [
      "Microsoft.ContainerService/managedClusters/delete"
    ]
  }]
}'
```

---

## Network Security {#network-security}

### Network Policies

```yaml
# Deny all traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: azure-learn-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow ingress from LoadBalancer
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-loadbalancer
  namespace: azure-learn-app
spec:
  podSelector:
    matchLabels:
      app: azure-learn-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 8080

---
# Allow egress to Cosmos DB
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cosmosdb
  namespace: azure-learn-app
spec:
  podSelector:
    matchLabels:
      app: azure-learn-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443  # HTTPS

---
# Allow DNS
- apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: azure-learn-app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Private Endpoints

```bash
# Prevent database access from internet
# Only accessible from VNet

# Create private endpoint for Cosmos DB
az network private-endpoint create \
  --name "cosmos-db-endpoint" \
  --resource-group "azure-learn-rg-dev" \
  --vnet-name "azurelearn-dev-vnet" \
  --subnet "aks-subnet" \
  --private-connection-resource-id "/subscriptions/.../azurelearndbhof7rpcc" \
  --group-ids "Sql"

# Create private endpoint for Key Vault
az network private-endpoint create \
  --name "keyvault-endpoint" \
  --resource-group "azure-learn-rg-dev" \
  --vnet-name "azurelearn-dev-vnet" \
  --subnet "aks-subnet" \
  --private-connection-resource-id "/subscriptions/.../azurelearnkvhof7rpcc" \
  --group-ids "vault"

# Result: Services only accessible from VNet (10.x.x.x)
# Not accessible from internet (20.x.x.x)
```

### Web Application Firewall (WAF)

```bicep
resource appGateway 'Microsoft.Network/applicationGateways@2023-06-01' = {
  name: 'app-gateway'
  location: location
  properties: {
    // ... gateway configuration
    webApplicationFirewallConfiguration: {
      enabled: true
      firewallMode: 'Prevention'  // Block malicious traffic
      ruleSetType: 'OWASP'
      ruleSetVersion: '3.2'
      disabledRuleGroups: []
      requestBodyCheck: true
      maxRequestBodySizeInKb: 128
      fileUploadLimitInMb: 100
      exclusions: []
    }
  }
}

// WAF protects against:
// - SQL injection
// - Cross-site scripting (XSS)
// - CSRF attacks
// - Large payloads
// - Malformed requests
```

---

## Data Protection {#data-protection}

### Encryption at Rest

```bicep
// Cosmos DB - automatically encrypted at rest
resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  properties: {
    // Microsoft-managed keys (default)
    // All data encrypted with AES-256
    
    // Optionally: Customer-managed keys (CMK)
    keyVaultKeyUri: 'https://keyvault.vault.azure.net/keys/mykey/version'
  }
}

// Storage account - enable customer-managed keys
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  properties: {
    encryption: {
      services: {
        blob: {
          enabled: true
          keyType: 'Account'  // Or 'Service' for CMK
        }
        file: {
          enabled: true
        }
      }
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
        keyname: 'storagekey'
        keyvaulturi: 'https://vault.azure.net/'
        keyversion: 'version'
      }
    }
  }
}
```

### Encryption in Transit

```csharp
// All connections use TLS 1.2+

// Cosmos DB
// Automatically uses TLS 1.2
var cosmosClient = new CosmosClient(
    accountEndpoint: endpoint,
    authKeyOrResourceToken: primaryKey,
    clientOptions: new CosmosClientOptions()
    {
        ConnectionMode = ConnectionMode.Direct,
        // TLS 1.2 enforced automatically
    }
);

// Key Vault
// Automatically uses HTTPS
var keyVaultUri = new Uri("https://keyvault.vault.azure.net/");
var client = new SecretClient(keyVaultUri, new DefaultAzureCredential());

// Application
// Use HTTPS in production
app.UseHttpsRedirection();  // In Program.cs
```

### Secret Rotation

```bash
# Set up automatic secret rotation

# Cosmos DB key rotation (90 days)
az cosmosdb keys regenerate \
  --type "keys" \
  --name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev

# Key Vault secret version management
# 1. Create new version
az keyvault secret set \
  --vault-name azurelearnkvhof7rpcc \
  --name CosmosDbConnectionString \
  --value "New value"

# 2. Old version still accessible
az keyvault secret list-versions \
  --vault-name azurelearnkvhof7rpcc \
  --name CosmosDbConnectionString

# 3. Kubernetes automatically uses latest
# (uses Secrets CSI driver to sync)

# Automate with Logic App or Function
# ├─ Trigger: Weekly
# ├─ Action: Regenerate secret
# └─ Action: Update Kubernetes secret
```

---

## Performance Optimization {#optimization}

### Caching Strategy

```
Caching Hierarchy:

User's Browser
    ↓ (Cache-Control: max-age=300)
CDN (Azure Front Door)
    ↓ (Cache static assets)
App Server (In-Memory)
    ↓ (Redis cache)
Database (Cosmos DB)
    ↓ (Materialized views)
    ↓ (Query optimization)
```

### Cosmos DB Optimization

```csharp
// 1. Use partition key in queries
// ❌ Inefficient (queries all partitions)
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM Products WHERE Category = 'Electronics'"
);

// ✅ Efficient (queries one partition)
var query = container.GetItemQueryIterator<Product>(
    "SELECT * FROM Products WHERE Category = 'Electronics'",
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKey("Electronics")
    }
);

// 2. Batch operations
// ❌ Slow (10 calls, 10 RUs)
for (int i = 0; i < 10; i++)
{
    await container.CreateItemAsync(products[i]);  // 1 RU each
}

// ✅ Fast (1 call, 10 RUs)
var batch = container.CreateBatch(new PartitionKey("category"));
foreach (var product in products)
{
    batch.CreateItemStream(product);
}
var result = await batch.ExecuteAsync();

// 3. Index optimization
// Adjust indexing policy for your queries
container.IndexingPolicy = new IndexingPolicy()
{
    IncludedPaths = 
    {
        new IncludedPath { Path = "/id/?" },        // Always index
        new IncludedPath { Path = "/Category/?" }   // Query filter
    },
    ExcludedPaths = 
    {
        new ExcludedPath { Path = "/Description/*" } // Not needed
    }
};
```

### Request Unit (RU) Optimization

```
Cost Model:

1 RU = 1 KB read (point lookup)

Cosmos DB Serverless:
├─ $0.25 per 1 million RUs
├─ $0.50 per 1 GB stored
├─ Billed per RU consumed
└─ Ideal for: Variable workloads, dev/test

Cosmos DB Provisioned:
├─ $0.12 per 100 RUs per hour
├─ Autoscale: $0.18-1.44 per 100 RUs
└─ Ideal for: Predictable workloads, production

Optimization Tips:
1. Use partition key in queries (1x RU cost)
2. Avoid cross-partition queries (10x RU cost)
3. Index only needed fields
4. Use pagination (limit results)
5. Cache frequently accessed data
```

### API Response Optimization

```csharp
public class ProductsController
{
    [HttpGet("api/products")]
    [ResponseCache(Duration = 60)]  // Cache 60 seconds
    public async Task<IEnumerable<Product>> GetProducts(
        [FromQuery] int pageSize = 20,
        [FromQuery] int pageNumber = 1)
    {
        // 1. Pagination (instead of returning all)
        var skip = (pageNumber - 1) * pageSize;
        var query = container.GetItemQueryIterator<Product>(
            $"SELECT * FROM Products OFFSET {skip} LIMIT {pageSize}"
        );
        
        // 2. Select only needed fields
        // ❌ SELECT * (returns description, images, etc.)
        // ✅ SELECT id, name, price
        
        // 3. Compress response
        [Produces("application/json")]
        [Header("Content-Encoding", "gzip")]
        
        // 4. Async all the way
        var results = new List<Product>();
        while (query.HasMoreResults)
        {
            var batch = await query.ReadNextAsync();  // Async
            results.AddRange(batch);
        }
        
        return results;
    }
}
```

---

## Scalability {#scalability}

### Horizontal Scaling

```
Initial Load: 10 requests/second

┌─────────────────────────────────┐
│ HPA Policy                      │
├─────────────────────────────────┤
│ Scale UP when:                  │
│ - CPU > 70%                     │
│ - Add 50% more pods             │
│                                 │
│ Scale DOWN when:                │
│ - CPU < 30%                     │
│ - Wait 5 mins                   │
│ - Remove 50% pods               │
└─────────────────────────────────┘

Time    Requests/s    CPU    Pods
────────────────────────────────────
00:00   10            20%    2 pods
00:05   100           75%    ↑ Scale up
00:06   100           45%    3 pods
00:10   300           80%    ↑ Scale up  
00:11   300           45%    4-5 pods
00:30   50            25%    Wait...
05:30   50            25%    ↓ Scale down
05:31   50            30%    3 pods
```

### Vertical Scaling

```
Pod Resource Limits:

Current:
├─ requests:
│  ├─ CPU: 100m
│  └─ Memory: 256Mi
├─ limits:
│  ├─ CPU: 500m
│  └─ Memory: 512Mi
└─ Result: Can handle 1000 req/s

If bottlenecked:
├─ Increase to:
│  ├─ CPU: 1000m (1 core)
│  └─ Memory: 1Gi
└─ Result: Can handle 5000 req/s

Trade-off:
├─ Pros: Simpler (fewer pods)
├─ Cons: Cost, latency (fewer failures)
└─ Best: Mix horizontal + vertical
```

### Connection Pooling

```csharp
// Reuse database connections

// ❌ Wrong: Create new connection each time
public async Task<Product> GetProduct(string id)
{
    using var client = new CosmosClient(connectionString);
    var container = client.GetContainer("db", "container");
    return await container.ReadItemAsync<Product>(id, partitionKey);
}
// Problem: New connection = 1 second overhead!

// ✅ Right: Singleton pattern
builder.Services.AddSingleton(sp => 
    new CosmosClient(connectionString)
);

public async Task<Product> GetProduct(string id)
{
    var container = _cosmosClient.GetContainer("db", "container");
    return await container.ReadItemAsync<Product>(id, partitionKey);
}
// Benefit: Reuse connection, 10ms overhead!
```

---

## Production Checklist {#checklist}

```
Before deploying to production:

SECURITY ✅
├─ [ ] Managed Identity configured
├─ [ ] RBAC roles assigned (least privilege)
├─ [ ] Network policies enabled
├─ [ ] Private endpoints created
├─ [ ] Secrets in Key Vault (not in code)
├─ [ ] TLS/SSL enabled
├─ [ ] Data encryption at rest
├─ [ ] WAF enabled
└─ [ ] Security scanning passed

PERFORMANCE ✅
├─ [ ] Database indexes optimized
├─ [ ] Response caching enabled
├─ [ ] Connection pooling configured
├─ [ ] Pagination implemented
├─ [ ] Load testing completed (1000 req/s)
├─ [ ] P95 latency < 500ms
├─ [ ] Error rate < 0.1%
└─ [ ] Cost per request calculated

RELIABILITY ✅
├─ [ ] Health checks implemented
├─ [ ] Liveness probe configured
├─ [ ] Readiness probe configured
├─ [ ] HPA configured (2-5 replicas)
├─ [ ] Pod disruption budgets set
├─ [ ] Backups automated
├─ [ ] Disaster recovery tested
└─ [ ] RTO < 4 hours, RPO < 1 hour

MONITORING ✅
├─ [ ] Application Insights enabled
├─ [ ] Structured logging configured
├─ [ ] Custom metrics defined
├─ [ ] Alerts configured
├─ [ ] Dashboards created
├─ [ ] On-call rotation set up
├─ [ ] Runbooks created
└─ [ ] Incident response plan ready

COMPLIANCE ✅
├─ [ ] Data residency verified
├─ [ ] GDPR compliance checked
├─ [ ] PII data encrypted
├─ [ ] Audit logging enabled
├─ [ ] Compliance documentation prepared
├─ [ ] Penetration testing completed
├─ [ ] Code security scan passed
└─ [ ] Dependencies vulnerability checked

OPERATIONS ✅
├─ [ ] Runbook created
├─ [ ] Escalation procedure defined
├─ [ ] Communication plan ready
├─ [ ] Deployment procedure documented
├─ [ ] Rollback procedure tested
├─ [ ] Team trained
├─ [ ] On-call schedule published
└─ [ ] SLOs communicated
```

---

## Key Takeaways

✅ **Security:** Multiple layers (app, network, infrastructure, data)  
✅ **Authentication:** Managed Identity (no passwords!)  
✅ **Authorization:** RBAC with least privilege  
✅ **Encryption:** At rest AND in transit  
✅ **Performance:** Caching, indexing, connection pooling  
✅ **Scalability:** HPA for auto-scaling  
✅ **Production Ready:** Use the complete checklist  

---

## Complete Documentation Summary

**You now have 8 comprehensive documents covering:**

- **Document 1:** Azure Fundamentals (subscriptions, regions, resource groups)
- **Document 2:** Azure Services (AKS, ACR, Cosmos DB, Key Vault, VNet, Managed Identity)
- **Document 3:** Application Code Architecture (layered architecture, DI, configuration)
- **Document 4:** Infrastructure & Docker (Bicep templates, containerization, Kubernetes)
- **Document 5:** CI/CD Pipelines (Azure DevOps, GitHub Actions, testing)
- **Document 6:** Commands Reference (Azure CLI, kubectl, Docker commands)
- **Document 7:** Monitoring & Logging (Application Insights, structured logs, alerts)
- **Document 8:** Security & Optimization (RBAC, encryption, performance tuning)

**Total Content:** 75,000+ words with diagrams, code examples, and practical guidance

**Next Steps:**
1. Review each document thoroughly
2. Implement security checklist
3. Set up monitoring and alerts
4. Run load tests
5. Deploy to production with confidence

✅ You have everything needed to understand, deploy, and operate this application in Azure!
