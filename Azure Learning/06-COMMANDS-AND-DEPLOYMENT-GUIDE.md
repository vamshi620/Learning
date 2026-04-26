# Complete Commands Reference and Deployment Guide
## Document 6: Step-by-Step Setup, Commands, and Troubleshooting

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Complete commands reference and hands-on deployment guide

---

## TABLE OF CONTENTS

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#setup)
3. [Azure CLI Commands](#azure-commands)
4. [Kubernetes Commands](#kubectl-commands)
5. [Docker Commands](#docker-commands)
6. [Bicep Commands](#bicep-commands)
7. [Complete Deployment Workflow](#deployment-workflow)
8. [Troubleshooting Common Issues](#troubleshooting)

---

## Prerequisites {#prerequisites}

### Required Tools Installation

```bash
# Windows (PowerShell as Administrator)

# 1. Install Azure CLI
# Method A: Download from https://aka.ms/installazurecliwindows
# Method B: Using winget
winget install Microsoft.AzureCLI

# Verify
az --version
# Output: azure-cli 2.55.0

# 2. Install kubectl
az aks install-cli
# or
choco install kubernetes-cli

# Verify
kubectl version --client

# 3. Install Docker Desktop
# Download from https://www.docker.com/products/docker-desktop
# Or: choco install docker-desktop

# Verify
docker --version

# 4. Install .NET 8 SDK
# Download from https://dotnet.microsoft.com/download
# Or: choco install dotnet-sdk

# Verify
dotnet --version

# 5. Install Git
choco install git

# Verify
git --version
```

### Azure Subscription Setup

```bash
# Login to Azure
az login

# If multiple subscriptions, set default
az account set --subscription "subscription-name-or-id"

# Verify
az account show
# Shows: name, id, tenantId, isDefault

# Create resource group
az group create \
  --name azure-learn-rg-dev \
  --location westus

# Verify
az group show --name azure-learn-rg-dev
```

---

## Initial Setup {#setup}

### Step 1: Clone Repository

```bash
# Clone project
git clone https://github.com/your-org/AzureLearnApp.git
cd AzureLearnApp

# Create feature branch
git checkout -b feature/setup-infrastructure
```

### Step 2: Install Dependencies

```bash
# Restore .NET packages
dotnet restore

# Output:
# Restored /path/to/AzureLearnApp/AzureLearnApp.csproj
```

### Step 3: Update Configuration

```bash
# Create local appsettings file (not committed to Git)
cp appsettings.json appsettings.Development.json

# Edit with your values
vim appsettings.Development.json

# Content:
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "KeyVault": {
    "Url": "https://azurelearnkvhof7rpcc.vault.azure.net/"
  },
  "CosmosDb": {
    "DatabaseName": "AzureLearnDb",
    "ContainerName": "Products"
  }
}
```

### Step 4: Run Locally

```bash
# Run application
dotnet run

# Output:
# info: Microsoft.Hosting.Lifetime[14]
#       Now listening on: http://localhost:5000
# info: Microsoft.Hosting.Lifetime[15]
#       Application started. Press Ctrl+C to exit.

# Test endpoint
curl http://localhost:5000/api/products

# Open Swagger UI
# Navigate to: http://localhost:5000/swagger/index.html
```

---

## Azure CLI Commands {#azure-commands}

### Resource Group Management

```bash
# Create resource group
az group create \
  --name azure-learn-rg-dev \
  --location westus

# List all resource groups
az group list --output table

# Output:
# Name                     Location    Status
# ─────────────────────────────────────────
# azure-learn-rg-dev       westus      Succeeded
# azure-learn-rg-prod      westus      Succeeded

# Delete resource group (⚠️ Deletes all resources!)
az group delete \
  --name azure-learn-rg-dev \
  --yes

# Get resource group details
az group show --name azure-learn-rg-dev
```

### Azure Container Registry (ACR) Commands

```bash
# Create container registry
az acr create \
  --resource-group azure-learn-rg-dev \
  --name azurelearnacrhof7rpcc \
  --sku Standard \
  --admin-enabled true

# Login to registry
az acr login --name azurelearnacrhof7rpcc

# List repositories
az acr repository list --name azurelearnacrhof7rpcc

# List image tags
az acr repository show-tags \
  --name azurelearnacrhof7rpcc \
  --repository azurelearn

# Output:
# [
#   "latest",
#   "v1.0",
#   "v2.0",
#   "v3.0"
# ]

# Build image in ACR (recommended)
az acr build \
  --registry azurelearnacrhof7rpcc \
  --image azurelearn:v3.0 \
  --image azurelearn:latest \
  .

# Output:
# Queued with ID: cf6
# Waiting for an agent...
# ...
# Run ID: cf6 was successful after 58s

# Delete image
az acr repository delete \
  --name azurelearnacrhof7rpcc \
  --image azurelearn:v1.0 \
  --yes
```

### Azure Kubernetes Service (AKS) Commands

```bash
# Create AKS cluster
az aks create \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --node-count 2 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard \
  --enable-managed-identity \
  --network-plugin azure

# List AKS clusters
az aks list --output table

# Get AKS credentials (kubeconfig)
az aks get-credentials \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --overwrite-existing

# Output:
# Merged "azurelearn-dev-aks" as current context in ~/.kube/config

# Get cluster info
az aks show \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks

# Scale cluster (change node count)
az aks scale \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --node-count 3

# Delete AKS cluster
az aks delete \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --yes
```

### Azure Cosmos DB Commands

```bash
# Create Cosmos DB account
az cosmosdb create \
  --name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev \
  --kind GlobalDocumentDB \
  --default-consistency-level Session

# Create database
az cosmosdb sql database create \
  --account-name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev \
  --name AzureLearnDb

# Create container (collection)
az cosmosdb sql container create \
  --account-name azurelearndbhof7rpcc \
  --database-name AzureLearnDb \
  --resource-group azure-learn-rg-dev \
  --name Products \
  --partition-key-path "/id"

# List databases
az cosmosdb sql database list \
  --account-name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev

# List containers
az cosmosdb sql container list \
  --account-name azurelearndbhof7rpcc \
  --database-name AzureLearnDb \
  --resource-group azure-learn-rg-dev

# Get connection string
az cosmosdb keys list \
  --name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev

# Delete database
az cosmosdb sql database delete \
  --account-name azurelearndbhof7rpcc \
  --database-name AzureLearnDb \
  --resource-group azure-learn-rg-dev \
  --yes
```

### Azure Key Vault Commands

```bash
# Create Key Vault
az keyvault create \
  --name azurelearnkvhof7rpcc \
  --resource-group azure-learn-rg-dev \
  --location westus

# Add secret
az keyvault secret set \
  --vault-name azurelearnkvhof7rpcc \
  --name CosmosDbConnectionString \
  --value "AccountEndpoint=...;AccountKey=..."

# Get secret
az keyvault secret show \
  --vault-name azurelearnkvhof7rpcc \
  --name CosmosDbConnectionString
  --query value -o tsv

# List secrets
az keyvault secret list \
  --vault-name azurelearnkvhof7rpcc

# Delete secret
az keyvault secret delete \
  --vault-name azurelearnkvhof7rpcc \
  --name CosmosDbConnectionString

# Set access policy
az keyvault set-policy \
  --name azurelearnkvhof7rpcc \
  --object-id <managed-identity-principal-id> \
  --secret-permissions get list
```

---

## Kubernetes Commands {#kubectl-commands}

### Cluster Information

```bash
# Get cluster info
kubectl cluster-info

# Get cluster version
kubectl version

# Get nodes
kubectl get nodes
# Output:
# NAME                                STATUS   ROLES
# aks-nodepool1-12345678-vmss000000   Ready    agent
# aks-nodepool1-12345678-vmss000001   Ready    agent

# Node details
kubectl describe nodes
```

### Namespace Management

```bash
# Create namespace
kubectl create namespace azure-learn-app

# List namespaces
kubectl get namespaces

# Set default namespace
kubectl config set-context --current --namespace=azure-learn-app

# Delete namespace (deletes all resources in it!)
kubectl delete namespace azure-learn-app
```

### Deployment Management

```bash
# Apply manifest
kubectl apply -f k8s/deployment.yaml

# List deployments
kubectl get deployments -n azure-learn-app

# Get deployment details
kubectl describe deployment azure-learn-app -n azure-learn-app

# Update image
kubectl set image deployment/azure-learn-app \
  app=azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0 \
  -n azure-learn-app

# Rollout status
kubectl rollout status deployment/azure-learn-app \
  -n azure-learn-app

# Rollout history
kubectl rollout history deployment/azure-learn-app \
  -n azure-learn-app

# Rollback to previous version
kubectl rollout undo deployment/azure-learn-app \
  -n azure-learn-app

# Restart deployment (pull new image)
kubectl rollout restart deployment/azure-learn-app \
  -n azure-learn-app

# Scale deployment
kubectl scale deployment/azure-learn-app \
  --replicas 3 \
  -n azure-learn-app
```

### Pod Management

```bash
# List pods
kubectl get pods -n azure-learn-app

# Pod details
kubectl describe pod <pod-name> -n azure-learn-app

# Logs (single pod)
kubectl logs <pod-name> -n azure-learn-app

# Logs (follow/stream)
kubectl logs -f <pod-name> -n azure-learn-app

# Logs (all pods of deployment)
kubectl logs -l app=azure-learn-app -n azure-learn-app

# Execute command in pod
kubectl exec -it <pod-name> -n azure-learn-app -- /bin/bash

# Port forward (tunnel local port to pod)
kubectl port-forward <pod-name> 8080:8080 \
  -n azure-learn-app
# Then: curl http://localhost:8080/api/products

# Delete pod (causes restart)
kubectl delete pod <pod-name> -n azure-learn-app
```

### Service Management

```bash
# List services
kubectl get svc -n azure-learn-app

# Service details
kubectl describe svc azure-learn-app-service \
  -n azure-learn-app

# Get external IP (LoadBalancer)
kubectl get svc azure-learn-app-service \
  -n azure-learn-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Port forward to service
kubectl port-forward svc/azure-learn-app-service 8080:80 \
  -n azure-learn-app
```

### ConfigMap and Secrets

```bash
# List ConfigMaps
kubectl get configmap -n azure-learn-app

# View ConfigMap
kubectl describe configmap app-config -n azure-learn-app

# Edit ConfigMap
kubectl edit configmap app-config -n azure-learn-app

# List secrets
kubectl get secrets -n azure-learn-app

# Create secret (from file)
kubectl create secret generic cosmos-db-secret \
  --from-literal=connectionString="..." \
  -n azure-learn-app

# View secret (base64 encoded)
kubectl get secret cosmos-db-secret \
  -o jsonpath='{.data.connectionString}' \
  -n azure-learn-app | base64 -d
```

### Monitoring

```bash
# Resource usage (pods)
kubectl top pods -n azure-learn-app

# Resource usage (nodes)
kubectl top nodes

# Pod events
kubectl get events -n azure-learn-app

# Recent pod events
kubectl get events -n azure-learn-app \
  --sort-by='.lastTimestamp' \
  --field-selector type!=Normal

# Pod status conditions
kubectl get pod <pod-name> \
  -o jsonpath='{.status.conditions}' \
  -n azure-learn-app | jq

# HPA status
kubectl get hpa -n azure-learn-app

# HPA details
kubectl describe hpa azure-learn-app-hpa \
  -n azure-learn-app
```

---

## Docker Commands {#docker-commands}

### Building Images

```bash
# Build image locally
docker build -t azurelearn:v3.0 .

# Build with multiple tags
docker build \
  -t azurelearn:v3.0 \
  -t azurelearn:latest \
  .

# Build specific dockerfile
docker build -f Dockerfile -t azurelearn:v3.0 .

# Show build output
docker build --progress=plain -t azurelearn:v3.0 .

# Get image size
docker images azurelearn:v3.0
```

### Running Containers

```bash
# Run container
docker run -d \
  --name myapp \
  -p 8080:8080 \
  azurelearn:v3.0

# Run with environment variables
docker run -d \
  --name myapp \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e KeyVault__Url=https://example.vault.azure.net/ \
  -p 8080:8080 \
  azurelearn:v3.0

# Run interactive (bash shell)
docker run -it \
  azurelearn:v3.0 \
  /bin/bash

# Stop container
docker stop myapp

# Remove container
docker rm myapp

# Remove all stopped containers
docker container prune

# Get container logs
docker logs myapp

# Follow logs
docker logs -f myapp

# Execute command in running container
docker exec myapp curl http://localhost:8080/health/live

# Container stats (CPU, memory)
docker stats myapp
```

### Image Management

```bash
# List images
docker images

# Tag image
docker tag azurelearn:v3.0 azurelearn:latest

# Remove image
docker rmi azurelearn:v1.0

# Remove dangling images
docker image prune

# Image history
docker history azurelearn:v3.0

# Inspect image
docker inspect azurelearn:v3.0
```

### Registry Operations

```bash
# Login to ACR
az acr login --name azurelearnacrhof7rpcc

# Tag for ACR
docker tag azurelearn:v3.0 \
  azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0

# Push to ACR
docker push azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0

# Pull from ACR
docker pull azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
```

---

## Bicep Commands {#bicep-commands}

### Building and Validating

```bash
# Build Bicep to ARM template
az bicep build --file infra/main.bicep

# Output:
# Bicep file "/path/to/main.bicep" was successfully decompiled.
# Generated "main.json"

# Validate Bicep template
az deployment group validate \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep

# Output:
# {
#   "kind": "template",
#   "id": "/subscriptions/.../providers/Microsoft.Resources/deployments/...",
#   "properties": {
#     "provisioningState": "Succeeded"
#   }
# }
```

### Deployment

```bash
# Deploy Bicep template (with parameters)
az deployment group create \
  --name "deploy-infrastructure" \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep \
  --parameters \
    location=westus \
    environment=dev \
    projectName=azurelearn

# Deploy with parameter file
az deployment group create \
  --name "deploy-infrastructure" \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep \
  --parameters infra/main.parameters.json

# Dry-run (show what would change)
az deployment group what-if \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep

# Output:
# Resource operations:
# 
# + Microsoft.DocumentDB/databaseAccounts/azurelearndb... Create
# + Microsoft.KeyVault/vaults/azurelearnkv... Create
# + Microsoft.ContainerRegistry/registries/azurelearnacr... Create

# Check deployment status
az deployment group show \
  --name "deploy-infrastructure" \
  --resource-group azure-learn-rg-dev

# Get deployment outputs
az deployment group show \
  --name "deploy-infrastructure" \
  --resource-group azure-learn-rg-dev \
  --query properties.outputs
```

---

## Complete Deployment Workflow {#deployment-workflow}

### Full Setup from Scratch

```bash
# ============================================================
# STEP 1: Initial Setup
# ============================================================

# Login to Azure
az login

# Create resource group
az group create \
  --name azure-learn-rg-dev \
  --location westus

# Set default subscription
az account set --subscription "your-subscription-id"


# ============================================================
# STEP 2: Deploy Infrastructure (Bicep)
# ============================================================

# Navigate to repo
cd AzureLearnApp

# Validate Bicep
az deployment group validate \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep

# Deploy infrastructure
az deployment group create \
  --name "deploy-core-infrastructure" \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep \
  --parameters \
    location=westus \
    environment=dev

# Wait for deployment to complete (5-10 minutes)


# ============================================================
# STEP 3: Deploy AKS Cluster (Bicep)
# ============================================================

# Deploy AKS cluster
az deployment group create \
  --name "deploy-aks-cluster" \
  --resource-group azure-learn-rg-dev \
  --template-file infra/aks.bicep \
  --parameters \
    location=westus \
    environment=dev \
    nodeCount=2

# Wait for AKS cluster to be ready (10-15 minutes)


# ============================================================
# STEP 4: Get AKS Credentials
# ============================================================

# Get kubeconfig
az aks get-credentials \
  --resource-group azure-learn-rg-dev \
  --name azurelearn-dev-aks \
  --overwrite-existing

# Verify connection
kubectl cluster-info


# ============================================================
# STEP 5: Build and Push Docker Image
# ============================================================

# Build locally (optional, for testing)
docker build -t azurelearn:v3.0 .

# Or build and push to ACR directly
az acr build \
  --registry azurelearnacrhof7rpcc \
  --image azurelearn:v3.0 \
  --image azurelearn:latest \
  .

# Wait for build to complete (2-5 minutes)


# ============================================================
# STEP 6: Deploy to Kubernetes
# ============================================================

# Apply all Kubernetes manifests
kubectl apply -f k8s/deployment.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod \
  -l app=azure-learn-app \
  -n azure-learn-app \
  --timeout=300s

# Verify deployment
kubectl get deployments -n azure-learn-app
kubectl get pods -n azure-learn-app
kubectl get svc -n azure-learn-app


# ============================================================
# STEP 7: Get External IP and Test
# ============================================================

# Get external IP (LoadBalancer)
EXTERNAL_IP=$(kubectl get svc azure-learn-app-service \
  -n azure-learn-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "External IP: $EXTERNAL_IP"

# Test health check
curl http://$EXTERNAL_IP/health/live

# Test API
curl http://$EXTERNAL_IP/api/products

# Open in browser
# http://$EXTERNAL_IP/swagger/index.html


# ============================================================
# STEP 8: Verify Everything Works
# ============================================================

# Check pod logs
kubectl logs -l app=azure-learn-app -n azure-learn-app

# Monitor HPA
kubectl get hpa -n azure-learn-app

# Check events
kubectl get events -n azure-learn-app

echo "✅ Deployment complete!"
```

---

## Troubleshooting Common Issues {#troubleshooting}

### Docker Issues

**Problem: "Docker daemon not running"**
```bash
# Windows
# Solution: Start Docker Desktop application
# Or restart:
restart-service com.docker.service

# Verify
docker ps
```

**Problem: "Permission denied while trying to connect to Docker daemon"**
```bash
# Linux
# Solution: Add user to docker group
sudo usermod -aG docker $USER

# Verify (logout and login first)
docker ps
```

**Problem: "image not found"**
```bash
# Check if image exists
docker images

# If not, build it
docker build -t azurelearn:v3.0 .
```

### Kubernetes Issues

**Problem: "Pods stuck in Pending"**
```bash
# Check why
kubectl describe pod <pod-name> -n azure-learn-app

# Common causes:
# 1. Insufficient resources: Increase node count
#    az aks scale --node-count 3 ...
# 2. Image pull error: Check image path
#    kubectl get events -n azure-learn-app
# 3. Node not ready: Check node status
#    kubectl get nodes
```

**Problem: "CrashLoopBackOff"**
```bash
# Pod is crashing repeatedly
# Check logs
kubectl logs <pod-name> -n azure-learn-app

# Common causes:
# 1. Application error: Fix code, rebuild image
# 2. Missing environment variables: Check ConfigMap
#    kubectl get configmap -n azure-learn-app
# 3. Database connection issue: Check secrets
#    kubectl describe secret cosmos-db-secret -n azure-learn-app
```

**Problem: "ImagePullBackOff"**
```bash
# Can't pull Docker image from registry
# Causes and solutions:

# 1. Image doesn't exist
az acr repository list --name azurelearnacrhof7rpcc

# 2. ACR credentials missing
kubectl get secrets -n azure-learn-app
# Create if missing:
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azurelearnacrhof7rpcc.azurecr.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n azure-learn-app

# 3. Image name incorrect
# Verify: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
# Check: kubectl describe deployment azure-learn-app
```

**Problem: "ReadinessProbe failed"**
```bash
# Pod not ready for traffic
# Check what's failing
kubectl describe pod <pod-name> -n azure-learn-app

# Look for: Readiness probe failed
# Solution: 
# 1. Verify health endpoint: /health/ready
# 2. Increase initialDelaySeconds if app slow to start
# 3. Check application logs:
kubectl logs <pod-name> -n azure-learn-app
```

### Azure Issues

**Problem: "Subscription not found"**
```bash
# List subscriptions
az account list

# Set default
az account set --subscription "subscription-id"

# Verify
az account show
```

**Problem: "Resource group not found"**
```bash
# Create it
az group create \
  --name azure-learn-rg-dev \
  --location westus

# Verify
az group show --name azure-learn-rg-dev
```

**Problem: "Permission denied (RBAC)"**
```bash
# Check current permissions
az role assignment list --assignee $(az account show -o tsv --query user.name)

# Assign role (requires admin)
az role assignment create \
  --assignee "your-email@company.com" \
  --role "Contributor" \
  --scope /subscriptions/subscription-id/resourceGroups/azure-learn-rg-dev
```

**Problem: "Key Vault: 403 Forbidden"**
```bash
# Managed Identity doesn't have permission
# Add access policy
az keyvault set-policy \
  --name azurelearnkvhof7rpcc \
  --object-id <managed-identity-principal-id> \
  --secret-permissions get list

# Get principal ID
az identity show \
  --name azure-learn-app-pod-identity \
  --resource-group azure-learn-rg-dev \
  --query principalId -o tsv
```

### Deployment Issues

**Problem: "Deployment stuck in RollingUpdate"**
```bash
# Check status
kubectl rollout status deployment/azure-learn-app \
  -n azure-learn-app

# See what's happening
kubectl describe deployment azure-learn-app \
  -n azure-learn-app

# Solution: Update readiness probe timeout
# kubectl set probe deployment/azure-learn-app \
#   readiness --initial-delay-seconds=20 ...
```

**Problem: "Database connection failed"**
```bash
# Check connection string
kubectl get secret cosmos-db-secret \
  -o jsonpath='{.data.connectionString}' \
  -n azure-learn-app | base64 -d

# Test from pod
kubectl exec -it <pod-name> -n azure-learn-app -- \
  curl "connection-string-from-above"

# Verify Cosmos DB is running
az cosmosdb show \
  --name azurelearndbhof7rpcc \
  --resource-group azure-learn-rg-dev
```

**Problem: "Service external IP is pending"**
```bash
# Check LoadBalancer status
kubectl get svc azure-learn-app-service \
  -n azure-learn-app

# If EXTERNAL-IP shows <pending>:
# 1. Wait longer (can take 5-10 minutes)
# 2. Check AKS capacity
#    kubectl describe nodes
# 3. Restart service
#    kubectl rollout restart deployment/azure-learn-app

# Wait for IP
kubectl get svc azure-learn-app-service \
  -n azure-learn-app \
  --watch
```

---

## Quick Reference Cheatsheet

```bash
# Most common commands

# Login
az login

# Deploy infrastructure
az deployment group create \
  --resource-group azure-learn-rg-dev \
  --template-file infra/main.bicep

# Get AKS credentials
az aks get-credentials --resource-group azure-learn-rg-dev --name azurelearn-dev-aks

# Build and push image
az acr build --registry azurelearnacrhof7rpcc --image azurelearn:v3.0 .

# Deploy to Kubernetes
kubectl apply -f k8s/deployment.yaml

# Watch deployment
kubectl get pods -n azure-learn-app --watch

# View logs
kubectl logs -f deployment/azure-learn-app -n azure-learn-app

# Get external IP
kubectl get svc -n azure-learn-app

# Restart deployment
kubectl rollout restart deployment/azure-learn-app -n azure-learn-app
```

---

## Key Takeaways

✅ **Prerequisites:** Install Azure CLI, kubectl, Docker, .NET SDK  
✅ **Setup:** Create resource group, configure credentials  
✅ **Deployment:** Infrastructure (Bicep) → AKS → Docker → Kubernetes  
✅ **Monitoring:** Use kubectl and Azure CLI for visibility  
✅ **Troubleshooting:** Check logs, describe pods, inspect manifests  
✅ **Automation:** Use CI/CD pipelines for repeatable deployments  

**Next Document:** Document 7 will cover monitoring, logging, and observability.
