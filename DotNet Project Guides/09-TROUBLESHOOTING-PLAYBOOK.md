# Troubleshooting Playbook
## File 09: Fix Common Issues Fast

---

## How to Use This File

When something goes wrong, find your symptom in the table below and follow the fix. Each issue includes:
- **What you see** (the error/symptom)
- **Why it happens** (root cause)
- **How to fix it** (step-by-step)

---

## App Service Issues

### 🔴 App returns 500 Internal Server Error

**Why:** Unhandled exception in your code, missing config, or database connection failure.

```powershell
# Step 1: Check the logs
az webapp log tail --name $APP_NAME --resource-group $RG

# Step 2: Check Application Insights for the exception
# Azure Portal → App Insights → Failures → Filter by time

# Step 3: Common causes:
# - Missing connection string → Check App Settings in Portal
# - Database not accessible → Check SQL firewall rules
# - Missing NuGet package → Rebuild and redeploy
# - Key Vault access denied → Check Managed Identity role assignment
```

### 🔴 App is slow (high latency)

```powershell
# Check if it's the app or the database
# Azure Portal → App Insights → Performance → Server response time

# If slow requests correlate with slow SQL queries:
# → Optimize queries, add indexes, upgrade SQL tier

# If CPU is high:
# → Scale up (bigger plan) or scale out (more instances)
az appservice plan update --name $APP_PLAN --resource-group $RG --sku S1

# If memory is high:
# → Check for memory leaks, increase plan size
```

### 🔴 Deployment failed

```powershell
# Check deployment logs
az webapp deployment list-publishing-profiles --name $APP_NAME --resource-group $RG
az webapp log show --name $APP_NAME --resource-group $RG

# Common causes:
# - Wrong .NET version → Check runtime: az webapp config show --name $APP_NAME -g $RG
# - Build error → Build locally first: dotnet publish -c Release
# - ZIP too large → Check .dockerignore / file size
```

### 🔴 Custom domain not working

```powershell
# Check domain binding
az webapp config hostname list --webapp-name $APP_NAME --resource-group $RG

# Common causes:
# - DNS CNAME not propagated yet → Wait 5-30 minutes, check with: nslookup api.yourdomain.com
# - SSL binding missing → Create and bind a managed certificate
# - Plan tier too low → Custom domains require B1 or higher
```

---

## AKS / Kubernetes Issues

### 🔴 Pod is in `ImagePullBackOff`

**Why:** Kubernetes can't download your Docker image from the registry.

```powershell
# Step 1: Check the error details
kubectl describe pod <pod-name> -n myproject | Select-String -Pattern "Error|Failed"

# Step 2: Common fixes:
# A. ACR not attached to AKS
az aks update --name $AKS_NAME --resource-group $RG --attach-acr $ACR_NAME

# B. Image name/tag is wrong
# Check: kubectl get deployment myapi -n myproject -o jsonpath='{.spec.template.spec.containers[0].image}'
# Verify: az acr repository show-tags --name $ACR_NAME --repository myapi

# C. Image doesn't exist in ACR
az acr repository list --name $ACR_NAME --output table
```

### 🔴 Pod is in `CrashLoopBackOff`

**Why:** Your container starts but crashes immediately. K8s restarts it, it crashes again, repeat.

```powershell
# Step 1: Check the crash logs
kubectl logs <pod-name> -n myproject --previous
# --previous shows logs from the CRASHED container

# Step 2: Common causes:
# A. App throws exception at startup → Check connection strings, missing config
# B. Port mismatch → Dockerfile EXPOSE and containerPort in deployment.yaml must match
# C. Missing environment variable → Check ConfigMap/Secret is applied
kubectl get configmap myapi-config -n myproject -o yaml
kubectl get secret myapi-secrets -n myproject -o yaml

# D. Out of memory → Increase memory limits in deployment.yaml
# resources.limits.memory: "512Mi" → "1Gi"
```

### 🔴 Pod is in `Pending`

**Why:** Kubernetes can't schedule the pod on any node.

```powershell
# Step 1: Check why
kubectl describe pod <pod-name> -n myproject | Select-String -Pattern "Events" -Context 0,20

# Step 2: Common causes:
# A. Not enough resources → Nodes are full
kubectl top nodes                         # Check node CPU/memory usage
kubectl get pods --all-namespaces          # Check total pod count
# Fix: Add more nodes
az aks nodepool scale --name nodepool1 --cluster-name $AKS_NAME -g $RG --node-count 3

# B. Resource requests too high → Lower requests in deployment.yaml
# requests.cpu: "500m" → "250m"
# requests.memory: "512Mi" → "256Mi"
```

### 🔴 Service has no External IP (`<pending>`)

```powershell
# Check the service
kubectl get service myapi-service -n myproject

# If EXTERNAL-IP shows <pending> for more than 5 minutes:
# Check events
kubectl describe service myapi-service -n myproject

# Common causes:
# A. Load balancer provisioning is slow → Wait 5-10 minutes
# B. Azure quota exceeded → Check your subscription quota
# C. Service type is ClusterIP (not LoadBalancer) → Edit service.yaml
```

### 🔴 Can't connect to SQL Database from AKS

```powershell
# Test connectivity from a pod
kubectl exec -it <pod-name> -n myproject -- sh
# Inside the pod:
apt-get update && apt-get install -y dnsutils
nslookup myproject-dev-sql.database.windows.net

# Common causes:
# A. SQL Firewall doesn't allow AKS IPs
#    → Add AKS outbound IPs to SQL firewall
$AKS_IPS = az aks show --name $AKS_NAME -g $RG --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[].id" -o tsv
# Add each IP to SQL firewall rules

# B. Connection string is wrong
#    → Check the secret in K8s
kubectl get secret myapi-secrets -n myproject -o jsonpath='{.data.ConnectionStrings__DefaultConnection}' | base64 -d
```

---

## General Azure Issues

### 🔴 `az login` fails or token expired

```powershell
# Clear cached credentials and re-login
az account clear
az login

# If using a service principal:
az login --service-principal -u <app-id> -p <password> --tenant <tenant-id>
```

### 🔴 "AuthorizationFailed" / "does not have authorization"

```powershell
# You don't have the right role
# Check your current role
az role assignment list --assignee (az ad signed-in-user show --query id -o tsv) --output table

# Ask your admin to assign the right role:
# az role assignment create --assignee <your-email> --role Contributor --scope <resource-id>
```

### 🔴 "QuotaExceeded" error

```powershell
# Check your subscription quota
az vm list-usage --location eastus --output table

# If quota is exceeded:
# 1. Delete unused resources
# 2. Request quota increase:
#    Azure Portal → Subscriptions → Usage + Quotas → Request Increase
```

---

## Emergency Rollback Procedures

### App Service — Rollback

```powershell
# If you have deployment slots:
az webapp deployment slot swap --name $APP_NAME -g $RG --slot staging --target-slot production
# This swaps back to the previous version instantly

# If no slots — redeploy previous version from your CI/CD pipeline
# Go to Azure DevOps / GitHub Actions → find the last successful build → re-run
```

### AKS — Rollback

```powershell
# Rollback to previous deployment version
kubectl rollout undo deployment/myapi -n myproject

# Check rollback status
kubectl rollout status deployment/myapi -n myproject

# View rollout history
kubectl rollout history deployment/myapi -n myproject
```

---

> **Next Step:** Quick reference commands → [10-COMMANDS-CHEATSHEET.md](10-COMMANDS-CHEATSHEET.md)
