# Infrastructure as Code (Bicep) and Containerization
## Document 4: Deployment Infrastructure, Docker, and Kubernetes Manifests

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Infrastructure templates, containerization, and deployment specifications

---

## TABLE OF CONTENTS

1. [Infrastructure as Code (IaC) Concepts](#iac-concepts)
2. [Bicep Language](#bicep)
3. [Docker and Containerization](#docker)
4. [Kubernetes Manifests](#k8s-manifests)
5. [Deployment Strategy](#deployment-strategy)
6. [Networking Architecture](#networking)
7. [Security Best Practices](#security)

---

## Infrastructure as Code (IaC) Concepts {#iac-concepts}

### What is Infrastructure as Code?

**IaC** = Define infrastructure using code/text files instead of clicking UI

### Traditional (Manual) Approach ❌

```
1. Login to Azure Portal
2. Click "Create resource"
3. Fill 50 forms
4. Wait for deployment
5. Configure networking (manually)
6. Set up security (manually)
7. Create Kubernetes cluster (manually)
8. Deploy pods (manually)
9. Create databases (manually)

Problems:
- Slow and error-prone
- No version control
- Can't reproduce easily
- Mistakes hard to rollback
- Team coordination difficult
```

### IaC Approach (Bicep) ✅

```
1. Write deployment.bicep file
2. Run: az deployment group create ...
3. Everything created automatically:
   ├─ Resource groups
   ├─ Virtual networks
   ├─ Kubernetes clusters
   ├─ Databases
   ├─ Security policies
   └─ All connections configured

Benefits:
- Reproducible (same result every time)
- Version controlled (track changes)
- Auditable (who changed what)
- Repeatable (deploy 100x identically)
- Easy rollback (previous version)
- Documentation (code is documentation)
```

### IaC Workflow

```
┌─────────────────────────────────────────┐
│   Developer Writes Bicep Template       │
│   (infrastructure.bicep)                │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Commit to Git Repository               │
│  (version control)                      │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  CI/CD Pipeline Triggers                │
│  (Azure DevOps)                         │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Validate Template                      │
│  (Check for errors)                     │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Deploy to Development                  │
│  (Automated)                            │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Tests Run                              │
│  (Verify everything works)              │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Manual Approval                        │
│  (Human review)                         │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Deploy to Production                   │
│  (Automated)                            │
└────────────────┴────────────────────────┘
```

---

## Bicep Language {#bicep}

### What is Bicep?

**Bicep** = Azure's infrastructure language (cleaner than ARM JSON)

### Bicep vs ARM Template

**ARM Template (JSON - verbose):**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "location": {
      "type": "string",
      "defaultValue": "eastus"
    }
  },
  "variables": {
    "cosmosDbAccountName": "[concat('cosmos', uniqueString(resourceGroup().id))]"
  },
  "resources": [
    {
      "type": "Microsoft.DocumentDB/databaseAccounts",
      "apiVersion": "2023-11-15",
      "name": "[variables('cosmosDbAccountName')]",
      "location": "[parameters('location')]",
      "properties": { ... }
    }
  ]
}
```

**Bicep (Declarative - clean):**
```bicep
param location string = 'eastus'

var cosmosDbAccountName = 'cosmos${uniqueString(resourceGroup().id)}'

resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: cosmosDbAccountName
  location: location
  properties: { ... }
}
```

### Our Bicep Files

#### 1. main.bicep (Core Infrastructure)

```bicep
// ============================================================================
// PARAMETERS: Input values (can override at deployment time)
// ============================================================================
param location string = resourceGroup().location
param environment string = 'dev'
param projectName string = 'azurelearn'
param deploymentId string = uniqueString(resourceGroup().id, deployment().name)

// ============================================================================
// VARIABLES: Computed values (naming conventions)
// ============================================================================
var appName = '${projectName}-${environment}'
var cosmosDbAccountName = '${projectName}db${take(deploymentId, 8)}'
var keyVaultName = '${projectName}kv${take(deploymentId, 8)}'
var containerRegistryName = '${projectName}acr${take(deploymentId, 8)}'

// ============================================================================
// OUTPUTS: Values returned after deployment
// ============================================================================
output cosmosDbEndpoint string = cosmosDbAccount.properties.documentEndpoint
output keyVaultUri string = keyVault.properties.vaultUri
output containerRegistryLoginServer string = containerRegistry.properties.loginServer

// ============================================================================
// RESOURCES: Actual Azure resources
// ============================================================================

// Cosmos DB Account
resource cosmosDbAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: cosmosDbAccountName
  location: location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
      maxIntervalInSeconds: 5
      maxStalenessPrefix: 100
    }
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: false
      }
    ]
    capabilities: [
      {
        name: 'EnableServerless'
      }
    ]
    backupPolicy: {
      type: 'Periodic'
      periodicModeProperties: {
        backupIntervalInMinutes: 240
        backupRetentionIntervalInHours: 8
        backupStorageRedundancy: 'Local'
      }
    }
  }
}

// Cosmos DB Database
resource cosmosDbDatabase 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-11-15' = {
  parent: cosmosDbAccount
  name: 'AzureLearnDb'
  properties: {
    resource: {
      id: 'AzureLearnDb'
    }
  }
}

// Cosmos DB Container
resource cosmosDbContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: cosmosDbDatabase
  name: 'Products'
  properties: {
    resource: {
      id: 'Products'
      partitionKey: {
        paths: [
          '/id'
        ]
        kind: 'Hash'
      }
      defaultTtl: -1
    }
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    enabledForDeployment: true
    enabledForTemplateDeployment: true
    enabledForDiskEncryption: false
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    accessPolicies: []
    softDeleteRetentionInDays: 90
    enableSoftDelete: true
    enablePurgeProtection: true
  }
}

// Container Registry
resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: containerRegistryName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    adminUserEnabled: true
    publicNetworkAccess: 'Enabled'
    networkRuleBypassOptions: 'AzureServices'
    policies: {
      quarantinePolicy: {
        status: 'disabled'
      }
      trustPolicy: {
        type: 'Notary'
        status: 'disabled'
      }
      retentionPolicy: {
        days: 30
        status: 'enabled'
      }
    }
  }
}

// Managed Identity for AKS
resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${appName}-pod-identity'
  location: location
}
```

**Key Concepts:**
- **Parameters:** Inputs to template
- **Variables:** Computed values
- **Resources:** What to create
- **Outputs:** What to return
- **Dependencies:** Implicit (parent resource)

#### 2. aks.bicep (Kubernetes Cluster)

```bicep
param location string = resourceGroup().location
param environment string = 'dev'
param projectName string = 'azurelearn'
param nodeCount int = 2
param vmSize string = 'Standard_D2s_v3'

var clusterName = '${projectName}-${environment}-aks'
var vnetName = '${projectName}-${environment}-vnet'
var subnetName = 'aks-subnet'

// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/8'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.240.0.0/16'
          serviceEndpoints: [
            {
              service: 'Microsoft.Storage'
            }
            {
              service: 'Microsoft.KeyVault'
            }
          ]
        }
      }
    ]
  }
}

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2023-10-02-preview' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: '1.29'
    resourceGroup: resourceGroup().name
    dnsPrefix: clusterName
    enableRBAC: true
    networkProfile: {
      networkPlugin: 'azure'
      serviceCidr: '10.0.0.0/16'
      dnsServiceIP: '10.0.0.10'
      dockerBridgeCidr: '172.17.0.1/16'
      networkPolicy: 'azure'
    }
    agentPoolProfiles: [
      {
        name: 'system'
        count: nodeCount
        vmSize: vmSize
        osType: 'Linux'
        mode: 'System'
        vnetSubnetID: '${vnet.id}/subnets/${subnetName}'
        type: 'VirtualMachineScaleSets'
        availabilityZones: [
          '1'
          '2'
        ]
      }
    ]
    addonProfiles: {
      httpApplicationRouting: {
        enabled: false
      }
      monitoring: {
        enabled: false
      }
    }
  }
}

output aksClusterId string = aksCluster.id
output aksControlPlaneId string = aksCluster.properties.resourceId
```

**AKS Configuration Explained:**
- **kubernetesVersion:** K8s version (1.29)
- **networkPlugin:** How pods communicate (azure networking)
- **networkPolicy:** Firewall rules for pod communication
- **agentPoolProfiles:** Node pools configuration
- **availabilityZones:** Spread across zones for resilience

---

## Docker and Containerization {#docker}

### What is Docker?

**Docker** = Package application with all dependencies into a container

### Without Docker ❌

```
Issue: "Works on my machine but not on production"

My Computer:
- OS: Windows 11
- .NET: 8.0.1
- Docker: 24.0.1
- App runs: ✅

Server:
- OS: Ubuntu 20.04
- .NET: 8.0.0
- Docker: 23.0.0
- App runs: ❌

Why different?
- Different OS
- Different versions
- Different dependencies
- Different configuration
```

### With Docker ✅

```
Container = Sealed box with:
├─ OS (Ubuntu Linux)
├─ .NET 8.0
├─ Application code
├─ Dependencies (DLLs)
├─ Configuration
└─ Everything needed

My Computer:
- Docker starts container
- App runs: ✅

Production Server:
- Docker starts same container
- App runs: ✅

Result: Always works!
```

### Dockerfile Explained

```dockerfile
# Stage 1: Build stage
# Use SDK to compile code (includes development tools)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /app

# Copy project file
COPY src/*.csproj ./

# Restore NuGet packages
RUN dotnet restore
# Downloads: Entity Framework, Cosmos DB SDK, etc.

# Copy all source code
COPY src/ ./

# Publish (compile to release binary)
RUN dotnet publish -c Release -o /app/publish
# Output: /app/publish/AzureLearnApp.dll


# Stage 2: Runtime stage
# Use ASPNET runtime (no SDK, smaller image)
FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

# Create non-root user for security
RUN useradd -m -u 1001 appuser

# Copy published app from build stage
COPY --from=build /app/publish .

# Expose port 8080
EXPOSE 8080

# Set ownership
RUN chown -R appuser:appuser /app

# Run as non-root user
USER appuser

# Health check (Docker uses this)
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1

# Start application
ENTRYPOINT ["dotnet", "AzureLearnApp.dll"]
```

### Multi-Stage Build Benefits

```
Stage 1 (Build):
├─ 800 MB (includes SDK, compiler tools)
└─ Creates executable

Stage 2 (Runtime):
├─ 127 MB (only runtime, no SDK)
├─ Takes executable from Stage 1
└─ Final image: ~130 MB

If single stage:
└─ 800 MB (includes unnecessary SDK)

Result: 6x smaller image!

Benefits:
- Faster download (2 mins vs 12 mins)
- Less storage (100 MB vs 600 MB)
- Faster startup (quicker to start container)
```

### Image Layers

```
Image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0

Layer 1: Base Image (mcr.microsoft.com/dotnet/aspnet:8.0)
├─ Linux OS
├─ .NET Runtime
└─ System libraries (~127 MB)
   ↓ Cached if unchanged

Layer 2: Application
├─ AzureLearnApp.dll (~5 MB)
├─ Dependencies (~10 MB)
└─ Configuration (~1 MB)
   ↓ Downloaded only if changed

Total: ~143 MB

Layers allow:
- Faster rebuilds (unchanged layers cached)
- Bandwidth savings (only changed layers)
- Version tracking (layer history)
```

### Building and Pushing Image

**Build Process:**
```bash
cd AzureLearnApp

# Build image locally
docker build -t azurelearn:v3.0 .

# Result:
# ├─ Stage 1: Downloads SDK 800MB, compiles code
# ├─ Stage 2: Creates minimal runtime image
# └─ azurelearn:v3.0 (143 MB) ready

# Or build and push to Azure Container Registry
az acr build --registry azurelearnacrhof7rpcc \
             --image azurelearn:v3.0 .

# ACR handles build in Azure:
# ├─ Upload code to ACR
# ├─ Build in Azure (parallel builds)
# ├─ Push to registry automatically
# └─ No need to build locally
```

### Image Tagging

```
Image tagged as:
  azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
                                    ├─ Repository (azurelearn)
                                    └─ Tag (v3.0)

Multiple tags for same image:
  azurelearn:v3.0      ← Specific version
  azurelearn:latest    ← Most recent
  azurelearn:prod      ← Production
  azurelearn:stable    ← Stable version

Benefits:
- Version tracking
- Easy rollback (deploy v2.0 if v3.0 broken)
- Rolling updates (swap tags)
- A/B testing (different versions simultaneously)
```

---

## Kubernetes Manifests {#k8s-manifests}

### Manifest Files Structure

```
k8s/
└─ deployment.yaml
   ├─ Namespace (azure-learn-app)
   ├─ Secrets (cosmos-db-secret, acr-secret)
   ├─ ConfigMap (app-config)
   ├─ ServiceAccount (azure-learn-app-sa)
   ├─ Deployment (azure-learn-app)
   ├─ Service (LoadBalancer)
   └─ HorizontalPodAutoscaler (HPA)
```

### Deployment Manifest Breakdown

```yaml
# LEARNING CONCEPT: Kubernetes Manifest
# YAML format (human-readable configuration)

---
# Create namespace
apiVersion: v1
kind: Namespace
metadata:
  name: azure-learn-app
  labels:
    environment: production

---
# Create Secret for Cosmos DB
apiVersion: v1
kind: Secret
metadata:
  name: cosmos-db-secret
  namespace: azure-learn-app
type: Opaque
stringData:
  # In production: Use Managed Identity instead of secrets
  connectionString: "AccountEndpoint=...;AccountKey=..."

---
# Create ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: azure-learn-app
data:
  KeyVault__Url: "https://azurelearnkvhof7rpcc.vault.azure.net/"
  CosmosDb__DatabaseName: "AzureLearnDb"
  CosmosDb__ContainerName: "Products"
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"

---
# Create ServiceAccount for Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-learn-app-sa
  namespace: azure-learn-app
  annotations:
    azure.workload.identity/client-id: "bf9ea53d-6351-42bf-aa45-3328a0bd296f"

---
# Create Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-learn-app
  namespace: azure-learn-app
  labels:
    app: azure-learn-app
    version: v1
spec:
  # Number of pod replicas
  replicas: 2
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # One extra pod during update
      maxUnavailable: 0     # Zero pods down (no downtime)
  
  # Pod selection
  selector:
    matchLabels:
      app: azure-learn-app
  
  # Pod template
  template:
    metadata:
      labels:
        app: azure-learn-app
        version: v1
      annotations:
        azure.workload.identity/use: "true"
    
    spec:
      # Service account for authentication
      serviceAccountName: azure-learn-app-sa
      
      # Pod containers
      containers:
      - name: app
        image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
        imagePullPolicy: IfNotPresent  # Use local if exists
        
        # Container ports
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        # Environment variables from ConfigMap
        envFrom:
        - configMapRef:
            name: app-config
        
        # Additional environment variables
        env:
        - name: LOG_LEVEL
          value: "Information"
        
        # Resource requests and limits
        resources:
          requests:
            memory: "256Mi"      # Minimum needed
            cpu: "100m"          # Minimum needed
          limits:
            memory: "512Mi"      # Maximum allowed
            cpu: "500m"          # Maximum allowed
        
        # Liveness probe (is app running?)
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10  # Wait 10s after start
          periodSeconds: 30        # Check every 30s
          failureThreshold: 3      # Restart after 3 fails
          timeoutSeconds: 5        # Timeout per request
        
        # Readiness probe (is app ready for traffic?)
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 2
          timeoutSeconds: 5
        
        # Volume mounts
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
      
      # Image pull secrets (for private ACR)
      imagePullSecrets:
      - name: acr-pull-secret
      
      # Volumes
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      
      # Node selection (optional)
      nodeSelector:
        kubernetes.io/os: linux

---
# Create Service (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: azure-learn-app-service
  namespace: azure-learn-app
  labels:
    app: azure-learn-app
spec:
  # External access via LoadBalancer
  type: LoadBalancer
  
  # Pod selection
  selector:
    app: azure-learn-app
  
  # Port mapping
  ports:
  - name: http
    protocol: TCP
    port: 80              # External port (internet)
    targetPort: 8080      # Pod port (app)
    nodePort: 30131       # Alternative access
  
  # Load balancing
  sessionAffinity: None   # Round-robin
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # Sticky session timeout

---
# Create HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: azure-learn-app-hpa
  namespace: azure-learn-app
spec:
  # Target deployment
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: azure-learn-app
  
  # Scaling limits
  minReplicas: 2     # Always keep 2 pods
  maxReplicas: 5     # Never go above 5 pods
  
  # Metrics
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when avg CPU > 70%
  
  # Scaling behavior
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300      # Wait 5 mins before scaling down
      policies:
      - type: Percent
        value: 50                           # Remove 50% of pods
        periodSeconds: 60                   # Per minute
    scaleUp:
      stabilizationWindowSeconds: 0        # Scale up immediately
      policies:
      - type: Percent
        value: 50                           # Add 50% more pods
        periodSeconds: 60                   # Per minute
```

---

## Deployment Strategy {#deployment-strategy}

### Rolling Update (What We Use)

```
Initial: 2 old pods (v2.0)

Update 1: Kill 1 old, start 1 new
  ├─ 1 old pod (v2.0) - running
  ├─ 1 new pod (v3.0) - starting

Update 2: Old pod ready
  ├─ 1 old pod (v2.0) - running
  ├─ 1 new pod (v3.0) - ready

Update 3: Kill 1 old, start 1 new
  ├─ 1 old pod (v2.0) - shutting down
  ├─ 1 new pod (v3.0) - running
  ├─ 1 new pod (v3.0) - starting

Update 4: Complete
  ├─ 2 new pods (v3.0) - running

Benefits:
- Zero downtime (always pods running)
- Automatic rollback if fails
- Gradual traffic shift
- Easy monitoring

Configuration:
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # Max 1 extra during update
    maxUnavailable: 0     # Never have downtime
```

### Blue-Green Deployment (Alternative)

```
Blue (Current - v2.0):  2 pods running, handling traffic
Green (New - v3.0):     2 pods, NOT handling traffic yet

Testing:
  └─ Verify v3.0 works

Switch:
  └─ Load balancer switches to v3.0

Rollback:
  └─ If issue, switch back to v2.0 instantly

Trade-off:
- Pro: Instant rollback
- Con: 2x resources needed temporarily
```

### Canary Deployment (Advanced)

```
Initial: 100% traffic to v2.0

Phase 1: 95% v2.0, 5% v3.0
  └─ Monitor v3.0 for errors

Phase 2: 50% v2.0, 50% v3.0
  └─ More users on v3.0

Phase 3: 0% v2.0, 100% v3.0
  └─ Full migration

Benefits:
- Risk reduction
- Gradual testing
- Early error detection
- Traffic gradually shifts
```

---

## Networking Architecture {#networking}

### Network Communication Paths

```
┌────────────────────────────────────────────────────────┐
│              Internet (External)                       │
│  User: https://example.com/api/products              │
└───────────────────────┬────────────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        ↓                               ↓
┌──────────────────┐          ┌──────────────────┐
│ Azure Front Door │          │ Azure CDN        │
│ (Optional)       │          │ (Optional)       │
│ - DDoS protect   │          │ - Cache static   │
│ - Global routing │          │ - Speed up       │
└──────────────────┘          └──────────────────┘
        ↓                               ↓
        └───────────────┬───────────────┘
                        ↓
        ┌───────────────────────────┐
        │  Azure Load Balancer      │
        │  20.245.92.146:80         │
        │  (Public IP)              │
        └───────────────┬───────────┘
                        ↓
        ┌───────────────────────────┐
        │  Virtual Network          │
        │  10.0.0.0/8               │
        │  (Private, Azure-only)    │
        │                           │
        │  ┌───────────────────┐   │
        │  │ AKS Subnet        │   │
        │  │ 10.240.0.0/16     │   │
        │  │                   │   │
        │  │  ┌─────────────┐  │   │
        │  │  │  Pod 1      │  │   │
        │  │  │ 10.224.0.19 │  │   │
        │  │  └─────────────┘  │   │
        │  │                   │   │
        │  │  ┌─────────────┐  │   │
        │  │  │  Pod 2      │  │   │
        │  │  │ 10.224.0.26 │  │   │
        │  │  └─────────────┘  │   │
        │  └───────────────────┘   │
        │                           │
        │  ┌───────────────────┐   │
        │  │ Cosmos DB         │   │
        │  │ Private Endpoint  │   │
        │  │ 10.x.x.x          │   │
        │  └───────────────────┘   │
        │                           │
        │  ┌───────────────────┐   │
        │  │ Key Vault         │   │
        │  │ Private Endpoint  │   │
        │  │ 10.x.x.x          │   │
        │  └───────────────────┘   │
        └───────────────────────────┘

Communication:
- Internet → Public IP (LoadBalancer)
- LoadBalancer → Pod (internal routing)
- Pod → Pod (internal network)
- Pod → CosmosDB (private endpoint)
- Pod → Key Vault (private endpoint)
```

### Network Policies (Firewall Rules)

```yaml
# Allow ingress only from LoadBalancer
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-ingress-policy
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
          name: azure-learn-app
    ports:
    - protocol: TCP
      port: 8080

---
# Allow egress to internet and services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-egress-policy
  namespace: azure-learn-app
spec:
  podSelector:
    matchLabels:
      app: azure-learn-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: azure-learn-app
    ports:
    - protocol: TCP
      port: 5432  # Postgres
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53   # DNS
    - protocol: UDP
      port: 53
```

---

## Security Best Practices {#security}

### Secret Management

```
❌ WRONG:
- Store secrets in code
- Store secrets in config files
- Store secrets in environment variables (insecure)
- Share passwords in chat/email

✅ RIGHT:
- Store secrets in Azure Key Vault
- Access via Managed Identity (passwordless)
- Rotate secrets regularly
- Enable audit logging
- Use network policies
- Encrypt at rest and in transit
```

### Container Security

```dockerfile
# Good practices in Dockerfile:

# 1. Use specific versions, not latest
FROM mcr.microsoft.com/dotnet/aspnet:8.0  # ✅ Specific
# FROM mcr.microsoft.com/dotnet/aspnet      # ❌ Latest (unpredictable)

# 2. Run as non-root user
RUN useradd -m -u 1001 appuser
USER appuser                   # ✅ Non-root

# 3. Read-only file system where possible
# (Set in Kubernetes securityContext)

# 4. Remove unnecessary packages
RUN apt-get remove curl wget  # ✅ Minimal

# 5. Scan for vulnerabilities
# Use: trivy image azurelearn:v3.0
```

### RBAC (Role-Based Access Control)

```
Who can do what in Kubernetes?

User: alice@company.com
├─ Can: Deploy to development
├─ Can: View logs
└─ Cannot: Delete production

User: bob@company.com
├─ Can: Deploy to production
├─ Can: Modify networking
└─ Cannot: Access other teams' namespace

ServiceAccount: azure-learn-app-sa
├─ Can: Read ConfigMap in own namespace
├─ Can: Read Secrets in own namespace
└─ Cannot: Create resources
```

### Network Security

```yaml
# Security context for pod
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true        # Must run as non-root
    runAsUser: 1001           # Specific user ID
    fsGroup: 2000             # File system group
    allowPrivilegeEscalation: false  # Can't become root
    capabilities:
      drop:
      - ALL                   # Remove all Linux capabilities
      add:
      - NET_BIND_SERVICE      # Only allow binding to ports
  
  containers:
  - name: app
    image: azurelearn:v3.0
    securityContext:
      readOnlyRootFilesystem: true  # Root FS read-only
      allowPrivilegeEscalation: false
```

---

## Key Takeaways

✅ **IaC:** Version-controlled, reproducible infrastructure  
✅ **Bicep:** Clean syntax for Azure resources  
✅ **Docker:** Containerize apps for consistency  
✅ **Multi-stage builds:** Smaller, faster images  
✅ **Kubernetes:** Orchestrate, scale, manage containers  
✅ **Rolling updates:** Zero-downtime deployments  
✅ **Security:** RBAC, network policies, Secrets management  

**Next Document:** Document 5 will cover CI/CD pipelines and complete deployment workflow.
