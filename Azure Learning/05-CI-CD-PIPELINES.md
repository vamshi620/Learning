# CI/CD Pipelines and Complete Deployment Workflow
## Document 5: Automation, Testing, and Production Deployment

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** CI/CD pipeline, automation, testing, and deployment strategies

---

## TABLE OF CONTENTS

1. [CI/CD Concepts](#cicd-concepts)
2. [Azure DevOps Pipeline](#pipeline)
3. [Pipeline Stages](#stages)
4. [GitHub Actions Alternative](#github-actions)
5. [Testing Strategy](#testing)
6. [Monitoring and Observability](#monitoring)
7. [Disaster Recovery](#disaster-recovery)

---

## CI/CD Concepts {#cicd-concepts}

### What is CI/CD?

**CI** (Continuous Integration) = Automatically build, test on every code change  
**CD** (Continuous Deployment) = Automatically deploy to production

### Traditional Deployment (Manual) ❌

```
1. Developer: "My feature is ready"
2. Manager: "Send to QA"
3. QA: Manually tests for 2 weeks
4. QA: "Found 5 bugs"
5. Developer: Fixes bugs
6. Repeat steps 2-5 (weeks go by)
7. QA: "Approved!"
8. DevOps: Manual deployment
   ├─ SSH into server
   ├─ Stop application
   ├─ Backup database
   ├─ Pull new code
   ├─ Restart application
   └─ Pray nothing breaks
9. Deployment takes 4 hours
10. Breaks in production
11. Rollback (manual process)
12. Everyone blames everyone

Result: Slow, error-prone, painful
Deployment frequency: Once per month
Time to fix: 4 hours
```

### CI/CD Automated ✅

```
1. Developer: Commit code
2. Git hook: Triggers pipeline
3. Pipeline stage 1: Build
   ├─ Compile code
   ├─ Run unit tests
   ├─ Build Docker image
   └─ Push to registry (5 mins)
4. Pipeline stage 2: Validate
   ├─ Check infrastructure templates
   ├─ Security scanning
   └─ Compliance checks (2 mins)
5. Pipeline stage 3: Deploy to Dev
   ├─ Deploy automatically
   ├─ Run integration tests
   ├─ Smoke tests
   └─ Automated checks (10 mins)
6. Pipeline stage 4: Approval Gate
   ├─ Human review (Slack notification)
   └─ Approval needed
7. Pipeline stage 5: Deploy to Prod
   ├─ Deploy automatically
   ├─ Health checks
   ├─ Automated verification
   └─ Done! (5 mins)

Total time: 25 minutes end-to-end
Deployment frequency: Multiple times per day
Time to fix: 30 minutes (automated rollback)
Consistency: Same process every time

Result: Fast, reliable, consistent
```

### Pipeline Flow Diagram

```
┌──────────────────────────────────────────────────────┐
│       Developer Makes Change and Commits            │
│     $ git commit -m "Fix bug in API"                │
└────────────────┬─────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────┐
│    Git Repository (GitHub / Azure DevOps)           │
│    ├─ Webhook triggers pipeline                     │
│    └─ Code pushed                                   │
└────────────────┬─────────────────────────────────────┘
                 ↓
╔══════════════════════════════════════════════════════╗
║           PIPELINE STAGE 1: BUILD                    ║
║  ├─ Download source code                            ║
║  ├─ Restore NuGet packages                          ║
║  ├─ Compile .NET code                               ║
║  ├─ Run unit tests                                  ║
║  │  └─ If fail: Stop here, notify developer        ║
║  ├─ Build Docker image                              ║
║  └─ Push to Azure Container Registry                ║
║     Result: azurelearn:v3.0 (143 MB)                ║
╚════════════════┬═════════════════════════════════════╝
                 ↓
╔══════════════════════════════════════════════════════╗
║    PIPELINE STAGE 2: VALIDATE INFRASTRUCTURE        ║
║  ├─ Validate main.bicep template                    ║
║  ├─ Validate aks.bicep template                     ║
║  ├─ Check for errors                                ║
║  │  └─ If fail: Stop here, notify developer        ║
║  └─ Security scanning                               ║
║     Result: Infrastructure validated                 ║
╚════════════════┬═════════════════════════════════════╝
                 ↓
╔══════════════════════════════════════════════════════╗
║    PIPELINE STAGE 3: DEPLOY TO DEVELOPMENT          ║
║  ├─ Deploy infrastructure (Bicep)                   ║
║  ├─ Deploy pods to dev AKS                          ║
║  │  └─ New image: azurelearn:v3.0                   ║
║  ├─ Wait for health checks                          ║
║  ├─ Run integration tests                           ║
║  │  └─ API smoke tests                              ║
║  │  └─ Database connectivity tests                  ║
║  ├─ Automated verification                          ║
║  │  └─ If fail: Stop here, notify team              ║
║  └─ Dev environment ready                           ║
║     URL: dev-api.azurelearn.com                      ║
╚════════════════┬═════════════════════════════════════╝
                 ↓
╔══════════════════════════════════════════════════════╗
║         APPROVAL GATE (MANUAL)                       ║
║  ├─ Slack message to #deployments                   ║
║  │  "Ready to deploy to production?"                ║
║  │  [Approve] [Reject]                              ║
║  └─ Wait for human decision                         ║
╚════════════════┬═════════════════════════════════════╝
                 ↓
         [If Approved]
                 ↓
╔══════════════════════════════════════════════════════╗
║    PIPELINE STAGE 4: DEPLOY TO PRODUCTION           ║
║  ├─ Deploy to prod AKS                              ║
║  │  └─ New image: azurelearn:v3.0                   ║
║  ├─ Health checks                                   ║
║  ├─ Smoke tests against production                  ║
║  ├─ Monitor for errors (first 5 mins)               ║
║  │  └─ If error rate > 5%: Automated rollback       ║
║  └─ Production updated                              ║
║     URL: api.azurelearn.com                         ║
║     Users: Using new version automatically           ║
╚════════════════┬═════════════════════════════════════╝
                 ↓
┌──────────────────────────────────────────────────────┐
│           DEPLOYMENT COMPLETE                       │
│  Total time: ~25 minutes                            │
│  Changes live to users automatically                 │
│  Automatic rollback if issues detected               │
└──────────────────────────────────────────────────────┘
```

---

## Azure DevOps Pipeline {#pipeline}

### Pipeline YAML Structure

```yaml
# File: pipelines/azure-pipelines.yml
# This file defines the CI/CD workflow

trigger:
  branches:
    include:
    - main              # Trigger on main branch
    - Feature/*         # Trigger on Feature branches
    - develop          
  paths:
    exclude:
    - README.md         # Don't trigger on docs changes
    - '**/*.md'

pool:
  vmImage: 'ubuntu-latest'  # Build machine type

# Variables shared across pipeline
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'
  dockerImageName: 'azurelearn'
  acrRegistryName: 'azurelearnacrhof7rpcc'

stages:
# Stage 1: Build
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: BuildJob
    displayName: 'Build Docker Image'
    steps:
    
    # Step 1: Checkout code
    - checkout: self
      displayName: 'Checkout code'
    
    # Step 2: Setup .NET
    - task: UseDotNet@2
      displayName: 'Install .NET SDK'
      inputs:
        version: $(dotnetVersion)
        includePreviewVersions: false
    
    # Step 3: Restore packages
    - task: DotNetCoreCLI@2
      displayName: 'Restore NuGet packages'
      inputs:
        command: 'restore'
        projects: '**/AzureLearnApp.csproj'
    
    # Step 4: Build
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        projects: '**/AzureLearnApp.csproj'
        arguments: '--configuration $(buildConfiguration) --no-restore'
    
    # Step 5: Run unit tests
    - task: DotNetCoreCLI@2
      displayName: 'Run unit tests'
      inputs:
        command: 'test'
        projects: '**/AzureLearnApp.Tests.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --logger trx'
      condition: succeededOrFailed()
    
    # Step 6: Publish test results
    - task: PublishTestResults@2
      displayName: 'Publish test results'
      inputs:
        testResultsFormat: 'VSTest'
        testResultsFiles: '**/*.trx'
      condition: succeededOrFailed()
    
    # Step 7: Build Docker image
    - task: Docker@2
      displayName: 'Build Docker image'
      inputs:
        command: 'build'
        repository: $(dockerImageName)
        dockerfile: 'Dockerfile'
        tags: |
          $(Build.BuildId)
          latest
          v3.0
    
    # Step 8: Push to ACR
    - task: AzureCLI@2
      displayName: 'Push image to Azure Container Registry'
      inputs:
        azureSubscription: 'AzureLearnConnection'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az acr build \
            --registry $(acrRegistryName) \
            --image $(dockerImageName):v3.0 \
            --image $(dockerImageName):latest \
            .
    
    # Step 9: Publish build artifacts
    - task: PublishBuildArtifacts@1
      displayName: 'Publish artifacts'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'

# Stage 2: Validate Infrastructure
- stage: ValidateInfrastructure
  displayName: 'Validate Infrastructure'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - job: ValidateBicep
    displayName: 'Validate Bicep Templates'
    steps:
    
    - checkout: self
    
    # Validate main.bicep
    - task: AzureCLI@2
      displayName: 'Validate main.bicep'
      inputs:
        azureSubscription: 'AzureLearnConnection'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az bicep build --file infra/main.bicep --outdir ./output
    
    # Validate aks.bicep
    - task: AzureCLI@2
      displayName: 'Validate aks.bicep'
      inputs:
        azureSubscription: 'AzureLearnConnection'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az bicep build --file infra/aks.bicep --outdir ./output

# Stage 3: Deploy to Development
- stage: DeployToDev
  displayName: 'Deploy to Development'
  dependsOn: ValidateInfrastructure
  condition: succeeded()
  jobs:
  - deployment: DeployDev
    displayName: 'Deploy to Dev Environment'
    environment:
      name: Azure-Learn-Dev
    strategy:
      runOnce:
        deploy:
          steps:
          
          - checkout: self
          
          # Deploy infrastructure
          - task: AzureResourceGroupDeployment@2
            displayName: 'Deploy infrastructure'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              action: 'Create Update'
              resourceGroupName: 'azure-learn-rg-dev'
              location: 'westus'
              templateLocation: 'Linked artifact'
              csmFile: 'infra/main.bicep'
              csmParametersFile: 'infra/main.parameters.json'
              deploymentMode: 'Incremental'
          
          # Deploy to AKS
          - task: AzureCLI@2
            displayName: 'Deploy to AKS'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az aks get-credentials \
                  --name azurelearn-dev-aks \
                  --resource-group azure-learn-rg-dev \
                  --overwrite-existing
                
                kubectl apply -f k8s/deployment.yaml
                kubectl rollout restart deployment/azure-learn-app \
                  -n azure-learn-app
          
          # Wait for deployment
          - task: AzureCLI@2
            displayName: 'Wait for pods to be ready'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                kubectl wait --for=condition=ready pod \
                  -l app=azure-learn-app \
                  -n azure-learn-app \
                  --timeout=300s
          
          # Run smoke tests
          - task: AzureCLI@2
            displayName: 'Run smoke tests'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                EXTERNAL_IP=$(kubectl get svc \
                  azure-learn-app-service \
                  -n azure-learn-app \
                  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                
                # Test health check
                curl -f http://$EXTERNAL_IP/health/live || exit 1
                
                # Test API
                curl -f http://$EXTERNAL_IP/api/products || exit 1
                
                echo "Smoke tests passed!"

# Stage 4: Deploy to Production
- stage: DeployToProd
  displayName: 'Deploy to Production'
  dependsOn: DeployToDev
  condition: succeeded()
  jobs:
  - deployment: DeployProd
    displayName: 'Deploy to Prod Environment'
    environment:
      name: Azure-Learn-Prod
      resourceId: /subscriptions/$(subscriptionId)/resourceGroups/azure-learn-rg-prod
    strategy:
      runOnce:
        preDeployment:
          steps:
          - task: AzureCLI@2
            displayName: 'Pre-deployment checks'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Verify image exists
                az acr repository show \
                  --name azurelearnacrhof7rpcc \
                  --image azurelearn:v3.0
        
        deploy:
          steps:
          - checkout: self
          
          - task: AzureCLI@2
            displayName: 'Deploy to production AKS'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az aks get-credentials \
                  --name azurelearn-prod-aks \
                  --resource-group azure-learn-rg-prod \
                  --overwrite-existing
                
                kubectl apply -f k8s/deployment.yaml
                kubectl set image deployment/azure-learn-app \
                  app=azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0 \
                  -n azure-learn-app
        
        postDeployment:
          steps:
          - task: AzureCLI@2
            displayName: 'Post-deployment verification'
            inputs:
              azureSubscription: 'AzureLearnConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Monitor for 5 minutes
                # If error rate > 5%, trigger automatic rollback
                # Check metrics from Application Insights
                
                echo "Production deployment verified!"
```

---

## Pipeline Stages {#stages}

### Stage 1: Build

**Duration:** 5-10 minutes

**What happens:**
```
Code → Compile → Test → Docker Build → Push to ACR

Steps:
1. Get latest code from Git
2. Install .NET 8 SDK
3. Restore NuGet packages (~200 MB downloaded)
4. Compile to DLL
5. Run unit tests (should be fast <1min)
6. Build Docker image (multi-stage)
7. Push to Azure Container Registry

Success Criteria:
- ✅ Code compiles without errors
- ✅ All tests pass
- ✅ Docker image created (143 MB)
- ✅ Image pushed to registry

Failure Criteria:
- ❌ Compilation errors
- ❌ Unit test failures
- ❌ Docker build fails
- If fails: Stop pipeline, notify developer
```

### Stage 2: Validate Infrastructure

**Duration:** 2-3 minutes

**What happens:**
```
Bicep Templates → Validation → Check for Errors

Steps:
1. Build main.bicep (creates ARM template JSON)
2. Build aks.bicep (creates ARM template JSON)
3. Validate Bicep syntax
4. Check for undefined variables
5. Verify parameter types

Success Criteria:
- ✅ main.bicep valid
- ✅ aks.bicep valid
- ✅ No syntax errors

Failure Criteria:
- ❌ Bicep syntax error
- ❌ Undefined variables
- ❌ Invalid parameter types
- If fails: Stop pipeline, fix Bicep
```

### Stage 3: Deploy to Development

**Duration:** 15-20 minutes

**What happens:**
```
Infrastructure Deploy → App Deploy → Tests → Verification

Steps:
1. Deploy Bicep templates to dev resource group
   ├─ Create/update AKS cluster
   ├─ Create/update databases
   ├─ Create/update Key Vault
   └─ Create/update networking
2. Get AKS credentials (kubeconfig)
3. Apply Kubernetes manifests
4. Restart deployment (pull new image)
5. Wait for pods to be ready
   ├─ Check readiness probe passes
   └─ Wait max 5 minutes
6. Run smoke tests
   ├─ Health check endpoint: GET /health/live
   ├─ API test: GET /api/products
   └─ Database connectivity test
7. Verify resources created

Success Criteria:
- ✅ Infrastructure deployed
- ✅ All pods running
- ✅ Health checks pass
- ✅ Smoke tests pass

Failure Criteria:
- ❌ Infrastructure deployment fails
- ❌ Pods not ready after 5 minutes
- ❌ Health checks fail
- ❌ Smoke tests fail
- If fails: Stop pipeline, investigate
```

### Stage 4: Deploy to Production

**Duration:** 10-15 minutes (requires manual approval)

**What happens:**
```
Pre-checks → Approval Gate → Production Deploy → Post Verification

Steps:
1. Pre-deployment checks
   ├─ Verify Docker image exists in registry
   ├─ Verify development tests passed
   └─ Check for conflicts
2. Approval Gate
   ├─ Pipeline waits (pauses)
   ├─ Slack notification: "Ready to deploy production?"
   ├─ Team lead reviews and approves
   └─ Resume pipeline
3. Deploy to production
   ├─ Get production AKS credentials
   ├─ Apply Kubernetes manifests
   ├─ Update image to new version (v3.0)
   └─ Rolling update (no downtime)
4. Post-deployment verification
   ├─ Health checks
   ├─ API smoke tests
   ├─ Monitor for 5 minutes
   │  └─ If error rate > 5%: Automatic rollback
   └─ Notify team: "Deployment successful"

Success Criteria:
- ✅ Approved by team
- ✅ Production pods running
- ✅ Health checks pass
- ✅ No error spike
- ✅ Users can access app

Failure Criteria:
- ❌ Approval denied
- ❌ Pods not ready
- ❌ Health checks fail
- ❌ Error rate spike
- If fails: Automatic rollback to previous version
```

---

## GitHub Actions Alternative {#github-actions}

If using GitHub instead of Azure DevOps:

```yaml
# File: .github/workflows/deploy.yml

name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Test
      run: dotnet test --configuration Release --no-build
    
    - name: Build Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: azurelearnacrhof7rpcc.azurecr.io/azurelearn:${{ github.sha }}
        registry: azurelearnacrhof7rpcc.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
  
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to AKS
      run: |
        az aks get-credentials \
          --name azurelearn-prod-aks \
          --resource-group azure-learn-rg-prod \
          --admin
        
        kubectl set image deployment/azure-learn-app \
          app=azurelearnacrhof7rpcc.azurecr.io/azurelearn:${{ github.sha }} \
          -n azure-learn-app
      env:
        AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
        AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
        AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

---

## Testing Strategy {#testing}

### Test Pyramid

```
          ▲
         /│\
        / │ \
       /  │  \         E2E Tests (UI) - 5%
      /   │   \        - Full user workflows
     /    │    \       - Slow (2-5 minutes)
    /─────┼─────\      - Expensive (test infrastructure)
   /      │      \
  /       │       \    Integration Tests - 15%
 /        │        \   - API calls
/─────────┼─────────\  - Database queries
 \       │       /    - Medium speed (30-60s)
  \      │      /
   \─────┼─────/      Unit Tests - 80%
    \    │    /       - Single functions
     \   │   /        - Fast (<1s)
      \  │  /         - Cheap (no external resources)
       \ │ /
        \│/
         ▼
```

### Unit Tests

```csharp
[TestClass]
public class ProductServiceTests
{
    private ICosmosDbService _cosmosDbService;
    
    [TestInitialize]
    public void Setup()
    {
        // Mock the Cosmos DB service
        _cosmosDbService = new Mock<ICosmosDbService>().Object;
    }
    
    [TestMethod]
    public async Task GetProduct_WithValidId_ReturnsProduct()
    {
        // Arrange
        var product = new Product 
        { 
            Id = "123", 
            Name = "Test" 
        };
        
        // Act
        var result = await _cosmosDbService.GetItemAsync<Product>("123");
        
        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual("123", result.Id);
    }
    
    [TestMethod]
    public async Task GetProduct_WithInvalidId_ReturnsNull()
    {
        // Act
        var result = await _cosmosDbService.GetItemAsync<Product>("invalid");
        
        // Assert
        Assert.IsNull(result);
    }
}
```

### Integration Tests

```csharp
[TestClass]
public class ApiIntegrationTests
{
    private readonly HttpClient _client;
    
    public ApiIntegrationTests()
    {
        // Create test server
        var factory = new WebApplicationFactory<Program>();
        _client = factory.CreateClient();
    }
    
    [TestMethod]
    public async Task CreateProduct_ReturnsCreated()
    {
        // Arrange
        var product = new { Name = "Test", Price = 99.99 };
        var content = new StringContent(
            JsonSerializer.Serialize(product),
            Encoding.UTF8,
            "application/json"
        );
        
        // Act
        var response = await _client.PostAsync("/api/products", content);
        
        // Assert
        Assert.AreEqual(HttpStatusCode.Created, response.StatusCode);
    }
}
```

---

## Monitoring and Observability {#monitoring}

### Application Insights Integration

```csharp
// In Program.cs
builder.Services.AddApplicationInsightsTelemetry();

// Automatic tracking:
// - HTTP requests (status codes, duration)
// - Exceptions (stack traces)
// - Dependencies (SQL, HTTP calls)
// - Performance metrics (CPU, memory)

// Custom events
telemetryClient.TrackEvent("ProductCreated", 
    new Dictionary<string, string>
    {
        ["ProductId"] = product.Id,
        ["UserId"] = userId
    }
);

// Custom metrics
telemetryClient.GetMetric("ProductPrice").TrackValue(price);
```

### Kubernetes Monitoring

```bash
# Pod resource usage
kubectl top pods -n azure-learn-app

# Node resource usage
kubectl top nodes

# Logs from pod
kubectl logs -n azure-learn-app deployment/azure-learn-app --tail 100

# Real-time logs
kubectl logs -n azure-learn-app deployment/azure-learn-app -f

# Events (errors, warnings)
kubectl get events -n azure-learn-app --sort-by='.lastTimestamp'

# Pod status
kubectl describe pod <pod-name> -n azure-learn-app
```

### Azure Monitor Queries

```kusto
// Find error rates
traces
| where severityLevel >= 2  // Warning and above
| summarize count() by bin(timestamp, 5m)
| render timechart

// Slow API calls
requests
| where duration > 5000  // > 5 seconds
| project timestamp, url, duration, resultCode
| order by duration desc

// Database errors
dependencies
| where type == "SQL"
| where success == false
| summarize count() by bin(timestamp, 1m)
```

---

## Disaster Recovery {#disaster-recovery}

### Backup Strategy

```
Cosmos DB:
├─ Automatic backup every 4 hours
├─ 30-day retention
├─ Point-in-time restore available
└─ Geo-replicated (manual setup)

Key Vault:
├─ Soft delete (90 days)
├─ Purge protection enabled
└─ Audit logs of all access

Container Registry:
├─ Image tags (multiple versions)
├─ Geo-replication (optional)
└─ Webhook backups

AKS:
├─ Persistent volume snapshots
├─ etcd backup (managed by Microsoft)
└─ Node image backups
```

### Disaster Recovery Plan

```
Scenario 1: Pod Crash
├─ Kubernetes detects
├─ Automatic restart (liveness probe)
└─ No action needed

Scenario 2: Node Failure
├─ Kubernetes detects
├─ Reschedules pods on other nodes
└─ Auto-scaling adds new node

Scenario 3: Application Bug in Production
├─ Error rate spike detected (>5%)
├─ Automatic rollback to previous version
├─ Alerts sent to team
└─ Investigate and fix

Scenario 4: Database Corruption
├─ Stop application
├─ Restore from backup
├─ Verify data integrity
├─ Resume application

Scenario 5: Complete Region Failure
├─ Primary region unavailable
├─ Manual failover to secondary region
├─ Update DNS to point to secondary
└─ Restore from last backup
```

### Recovery Testing

```bash
# Test backup restoration (monthly)
1. Restore database from backup
2. Verify data integrity
3. Run smoke tests
4. Compare with live (checksums)
5. Document recovery time

# Test failover (quarterly)
1. Simulate region failure
2. Trigger failover
3. Measure RPO (data loss)
4. Measure RTO (recovery time)
5. Document and improve

# Chaos engineering (optional advanced)
1. Randomly kill pods
2. Inject network latency
3. Simulate resource exhaustion
4. Observe and improve
```

---

## Key Takeaways

✅ **CI/CD:** Automated build, test, deploy on every change  
✅ **Pipelines:** Consistent, repeatable deployment process  
✅ **Testing:** Unit, integration, E2E at all levels  
✅ **Monitoring:** Continuous visibility into production  
✅ **Disaster Recovery:** Automated failover and rollback  
✅ **Approval Gates:** Human review before production  
✅ **Rollback:** Automatic on error detection  

**Next Document:** Document 6 will cover complete commands reference and troubleshooting guide.
