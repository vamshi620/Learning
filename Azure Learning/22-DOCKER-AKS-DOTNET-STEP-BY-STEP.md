# Docker & AKS Implementation Guide for .NET Applications
## Document 22: Step-by-Step Playbook — From Code to Cloud

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Reusable, step-by-step checklist to Dockerize any .NET application and deploy it to AKS

---

## TABLE OF CONTENTS

1. [Overview — When to Use This Guide](#overview)
2. [Prerequisites Checklist](#prerequisites)
3. [Phase 1 — Prepare the .NET Application](#phase1)
4. [Phase 2 — Dockerize the Application](#phase2)
5. [Phase 3 — Set Up Azure Infrastructure](#phase3)
6. [Phase 4 — Push Image to ACR](#phase4)
7. [Phase 5 — Create Kubernetes Manifests](#phase5)
8. [Phase 6 — Deploy to AKS](#phase6)
9. [Phase 7 — Configure Helm Chart (Optional)](#phase7)
10. [Phase 8 — Set Up CI/CD Pipeline](#phase8)
11. [Phase 9 — Production Hardening](#phase9)
12. [Phase 10 — Monitoring & Observability](#phase10)
13. [Quick-Reference Cheatsheet](#cheatsheet)
14. [Troubleshooting Playbook](#troubleshooting)

---

## Overview — When to Use This Guide {#overview}

Use this document whenever you need to:

- Containerize a **new** or **existing** .NET application
- Deploy a .NET API/Web App to **Azure Kubernetes Service (AKS)**
- Set up end-to-end CI/CD for Docker + AKS
- Migrate an on-premises .NET service to the cloud

### What You'll Build

```
┌─────────────────────────────────────────────────────┐
│  .NET Application (Web API / MVC / Worker Service)  │
│                     ↓                               │
│            Docker Container Image                   │
│                     ↓                               │
│         Azure Container Registry (ACR)              │
│                     ↓                               │
│       Azure Kubernetes Service (AKS Cluster)        │
│       ├─ Deployment (pods running your app)         │
│       ├─ Service (LoadBalancer / Ingress)            │
│       ├─ ConfigMap & Secrets                        │
│       ├─ HPA (auto-scaling)                         │
│       └─ Health checks (liveness + readiness)       │
│                     ↓                               │
│             End Users / API Consumers               │
└─────────────────────────────────────────────────────┘
```

---

## Prerequisites Checklist {#prerequisites}

Run through this list before starting:

```
Tools Installed:
├─ [ ] .NET SDK 8.0+        → dotnet --version
├─ [ ] Docker Desktop        → docker --version
├─ [ ] Azure CLI             → az --version
├─ [ ] kubectl               → kubectl version --client
├─ [ ] Helm (optional)       → helm version
└─ [ ] Git                   → git --version

Azure Access:
├─ [ ] Azure subscription active
├─ [ ] Logged in: az login
├─ [ ] Correct subscription set: az account show
└─ [ ] Permissions: Contributor on resource group
```

---

## Phase 1 — Prepare the .NET Application {#phase1}

### Step 1.1: Ensure the App Runs Locally

```bash
# Restore packages
dotnet restore

# Build
dotnet build --configuration Release

# Run
dotnet run --urls "http://localhost:8080"

# Test health endpoint (if available)
curl http://localhost:8080/health/live
```

### Step 1.2: Add Health Check Endpoints

If your app doesn't have them, add health checks:

```csharp
// In Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    // Add database check if applicable:
    // .AddCosmosDb(connectionString)
    // .AddSqlServer(connectionString)
    ;

var app = builder.Build();

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Basic liveness — always healthy if process is up
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => true  // Readiness — checks dependencies
});
```

### Step 1.3: Configure the Listening Port

```csharp
// In Program.cs — ensure the app listens on 8080
builder.WebHost.UseUrls("http://+:8080");

// Or via environment variable (preferred for containers):
// Set ASPNETCORE_URLS=http://+:8080
```

### Step 1.4: Externalize Configuration

Ensure secrets are NOT hardcoded. Use environment variables or Azure Key Vault:

```json
// appsettings.json — non-sensitive only
{
  "Logging": { "LogLevel": { "Default": "Information" } },
  "AllowedHosts": "*",
  "DatabaseName": "MyDb"
}
```

```csharp
// Read from environment variables (set by K8s ConfigMap/Secret)
var connString = configuration["ConnectionStrings:DefaultConnection"];
var keyVaultUrl = configuration["KeyVault:Url"];
```

---

## Phase 2 — Dockerize the Application {#phase2}

### Step 2.1: Create the Dockerfile

Place this at the **project root** (same level as `.csproj` or `src/` folder):

```dockerfile
# ============================================
# Stage 1: Build
# ============================================
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore (layer caching)
COPY ["src/MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# Copy everything else and publish
COPY src/ .
WORKDIR /src/MyApp
RUN dotnet publish -c Release -o /app/publish --no-restore

# ============================================
# Stage 2: Runtime
# ============================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Security: create non-root user
RUN adduser --disabled-password --gecos "" --uid 1001 appuser

# Copy published output
COPY --from=build /app/publish .

# Set ownership
RUN chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

> **Adapt:** Replace `MyApp` and paths to match your actual project name and folder structure.

### Step 2.2: Create .dockerignore

```
**/.git
**/.vs
**/bin
**/obj
**/node_modules
**/.env
**/Dockerfile*
**/.dockerignore
**/docker-compose*
*.md
```

### Step 2.3: Build and Test Locally

```bash
# Build image
docker build -t myapp:local .

# Run container
docker run -d --name myapp-test \
  -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Development \
  myapp:local

# Test
curl http://localhost:8080/health/live

# Check logs
docker logs myapp-test

# Clean up
docker stop myapp-test && docker rm myapp-test
```

---

## Phase 3 — Set Up Azure Infrastructure {#phase3}

### Step 3.1: Create Resource Group

```bash
RESOURCE_GROUP="myapp-rg-dev"
LOCATION="eastus"

az group create --name $RESOURCE_GROUP --location $LOCATION
```

### Step 3.2: Create Azure Container Registry

```bash
ACR_NAME="myappacr$(openssl rand -hex 4)"

az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard \
  --admin-enabled true

# Save login server
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
echo "ACR: $ACR_LOGIN_SERVER"
```

### Step 3.3: Create AKS Cluster

```bash
AKS_NAME="myapp-dev-aks"

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --attach-acr $ACR_NAME \
  --network-plugin azure \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --overwrite-existing

# Verify
kubectl get nodes
```

### Step 3.4: Create Supporting Resources (as needed)

```bash
# Key Vault (for secrets)
KV_NAME="myappkv$(openssl rand -hex 4)"
az keyvault create \
  --name $KV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Add a secret
az keyvault secret set \
  --vault-name $KV_NAME \
  --name "ConnectionString" \
  --value "Server=...;Database=...;"

# Managed Identity (for workload identity)
MI_NAME="myapp-pod-identity"
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $MI_NAME

MI_CLIENT_ID=$(az identity show -g $RESOURCE_GROUP -n $MI_NAME --query clientId -o tsv)
MI_PRINCIPAL_ID=$(az identity show -g $RESOURCE_GROUP -n $MI_NAME --query principalId -o tsv)

# Grant Key Vault access
az keyvault set-policy \
  --name $KV_NAME \
  --object-id $MI_PRINCIPAL_ID \
  --secret-permissions get list
```

---

## Phase 4 — Push Image to ACR {#phase4}

### Option A: Build in ACR (Recommended)

```bash
# ACR builds the image in the cloud — no local Docker needed
az acr build \
  --registry $ACR_NAME \
  --image myapp:v1.0 \
  --image myapp:latest \
  .
```

### Option B: Build Locally, Push to ACR

```bash
# Login to ACR
az acr login --name $ACR_NAME

# Tag for ACR
docker tag myapp:local $ACR_LOGIN_SERVER/myapp:v1.0

# Push
docker push $ACR_LOGIN_SERVER/myapp:v1.0
```

### Verify

```bash
az acr repository list --name $ACR_NAME
az acr repository show-tags --name $ACR_NAME --repository myapp
```

---

## Phase 5 — Create Kubernetes Manifests {#phase5}

### Step 5.1: Create Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapp
```

### Step 5.2: ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  KeyVault__Url: "https://<your-keyvault>.vault.azure.net/"
  # Add any non-sensitive config here
```

### Step 5.3: ServiceAccount (for Workload Identity)

```yaml
# k8s/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
  annotations:
    azure.workload.identity/client-id: "<MI_CLIENT_ID>"
```

### Step 5.4: Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: <ACR_LOGIN_SERVER>/myapp:v1.0   # Replace
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
      nodeSelector:
        kubernetes.io/os: linux
```

### Step 5.5: Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### Step 5.6: HPA

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Phase 6 — Deploy to AKS {#phase6}

### Step 6.1: Apply All Manifests

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
```

### Step 6.2: Verify Deployment

```bash
# Check pods
kubectl get pods -n myapp

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=myapp -n myapp --timeout=120s

# Get external IP
kubectl get svc myapp-service -n myapp

# Test endpoints
EXTERNAL_IP=$(kubectl get svc myapp-service -n myapp -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP/health/live
curl http://$EXTERNAL_IP/api/products   # or your endpoint
```

### Step 6.3: Update Image (Rolling Update)

```bash
# Build and push new version
az acr build --registry $ACR_NAME --image myapp:v2.0 .

# Update deployment
kubectl set image deployment/myapp \
  myapp=$ACR_LOGIN_SERVER/myapp:v2.0 \
  -n myapp

# Watch rollout
kubectl rollout status deployment/myapp -n myapp

# Rollback if needed
kubectl rollout undo deployment/myapp -n myapp
```

---

## Phase 7 — Configure Helm Chart (Optional) {#phase7}

If you want Helm instead of raw manifests, see **Document 21: Helm Charts Complete Guide**.

Quick start:

```bash
# Create chart
helm create myapp-chart

# Edit values.yaml with your ACR, image, ports, etc.

# Deploy
helm upgrade --install myapp ./myapp-chart \
  --namespace myapp --create-namespace \
  --set image.repository=$ACR_LOGIN_SERVER/myapp \
  --set image.tag=v1.0 \
  --wait
```

---

## Phase 8 — Set Up CI/CD Pipeline {#phase8}

### Azure DevOps (Minimal Pipeline)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

pool:
  vmImage: 'ubuntu-latest'

variables:
  acrName: 'myappacr1234'
  imageName: 'myapp'

stages:
- stage: BuildAndPush
  jobs:
  - job: Build
    steps:
    - task: AzureCLI@2
      displayName: 'Build & Push to ACR'
      inputs:
        azureSubscription: 'MyAzureConnection'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az acr build \
            --registry $(acrName) \
            --image $(imageName):$(Build.BuildId) \
            --image $(imageName):latest .

- stage: Deploy
  dependsOn: BuildAndPush
  jobs:
  - job: DeployToAKS
    steps:
    - task: AzureCLI@2
      displayName: 'Deploy to AKS'
      inputs:
        azureSubscription: 'MyAzureConnection'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az aks get-credentials -g myapp-rg-dev -n myapp-dev-aks --overwrite-existing
          kubectl set image deployment/myapp \
            myapp=$(acrName).azurecr.io/$(imageName):$(Build.BuildId) \
            -n myapp
          kubectl rollout status deployment/myapp -n myapp --timeout=300s
```

### GitHub Actions (Minimal)

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy
on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build & Push
      run: |
        az acr build \
          --registry myappacr1234 \
          --image myapp:${{ github.sha }} .

    - name: Deploy
      run: |
        az aks get-credentials -g myapp-rg-dev -n myapp-dev-aks
        kubectl set image deployment/myapp \
          myapp=myappacr1234.azurecr.io/myapp:${{ github.sha }} \
          -n myapp
        kubectl rollout status deployment/myapp -n myapp
```

---

## Phase 9 — Production Hardening {#phase9}

### Security Checklist

```
├─ [ ] Run as non-root user in Dockerfile (USER appuser)
├─ [ ] Use specific base image tags, never :latest
├─ [ ] Secrets in Key Vault, NOT in ConfigMap or code
├─ [ ] Workload Identity configured (no passwords in pods)
├─ [ ] Network Policies applied (deny-all default + allow-list)
├─ [ ] Resource requests AND limits set
├─ [ ] Image vulnerability scanning enabled on ACR
├─ [ ] Pod Security Standards enforced (restricted profile)
├─ [ ] TLS/HTTPS via Ingress controller + cert-manager
└─ [ ] RBAC: least-privilege roles for ServiceAccounts
```

### Ingress with TLS (NGINX)

```bash
# Install NGINX Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Install cert-manager for Let's Encrypt
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.myapp.com]
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## Phase 10 — Monitoring & Observability {#phase10}

```bash
# Enable Container Insights on AKS
az aks enable-addons \
  --addon monitoring \
  --name $AKS_NAME \
  --resource-group $RESOURCE_GROUP

# Add Application Insights to your .NET app
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry();
```

```yaml
# Add connection string to ConfigMap
data:
  APPLICATIONINSIGHTS_CONNECTION_STRING: "InstrumentationKey=..."
```

---

## Quick-Reference Cheatsheet {#cheatsheet}

```bash
# === ONE-TIME SETUP ===
az login
az group create -n myapp-rg -l eastus
az acr create -g myapp-rg -n myappacr --sku Standard --admin-enabled true
az aks create -g myapp-rg -n myapp-aks --node-count 2 --attach-acr myappacr --enable-managed-identity
az aks get-credentials -g myapp-rg -n myapp-aks

# === BUILD & DEPLOY LOOP ===
az acr build --registry myappacr --image myapp:v1.0 .
kubectl apply -f k8s/
kubectl get pods -n myapp
kubectl get svc -n myapp

# === UPDATE ===
az acr build --registry myappacr --image myapp:v2.0 .
kubectl set image deployment/myapp myapp=myappacr.azurecr.io/myapp:v2.0 -n myapp

# === ROLLBACK ===
kubectl rollout undo deployment/myapp -n myapp

# === DEBUG ===
kubectl describe pod <pod-name> -n myapp
kubectl logs <pod-name> -n myapp
kubectl exec -it <pod-name> -n myapp -- /bin/sh
```

---

## Troubleshooting Playbook {#troubleshooting}

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `ImagePullBackOff` | ACR auth failed or image doesn't exist | `az aks check-acr --name <aks> -g <rg> --acr <acr>` — re-attach ACR |
| `CrashLoopBackOff` | App crashes on startup | `kubectl logs <pod>` — check connection strings / missing env vars |
| Pod stuck in `Pending` | No node capacity | `kubectl describe pod` — check resource requests vs node capacity |
| `0/2 nodes are available` | Insufficient resources | Scale up: `az aks scale -g <rg> -n <aks> --node-count 3` |
| `Readiness probe failed` | App not ready or wrong health path | Verify `/health/ready` path and port in deployment |
| `Service has no endpoints` | Label mismatch | Ensure `spec.selector` in Service matches Deployment labels |
| LoadBalancer stuck on `<pending>` | Quota/permissions issue | `kubectl describe svc` — check Azure events for LB errors |
| 502 Bad Gateway (Ingress) | Backend not ready or wrong port | Verify `targetPort` matches `containerPort` |

---

## Key Takeaways

✅ **Phase 1–2:** Make the app container-ready (health checks, port config, Dockerfile)  
✅ **Phase 3–4:** Create Azure infra (ACR + AKS) and push the image  
✅ **Phase 5–6:** Write K8s manifests and deploy  
✅ **Phase 7–8:** Use Helm and CI/CD for repeatable deployments  
✅ **Phase 9–10:** Harden security and add observability  

**This guide is reusable for any .NET application** — just adapt the project name, ports, and configuration values.
