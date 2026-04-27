# Commands Cheatsheet
## File 10: All Commands on One Page

---

## Azure CLI

```powershell
# ─── LOGIN & SUBSCRIPTION ──────────────────────────────────────
az login                                    # Login via browser
az account show                             # Show current subscription
az account list --output table              # List all subscriptions
az account set --subscription "name"        # Switch subscription

# ─── RESOURCE GROUPS ───────────────────────────────────────────
az group create --name myRG --location eastus --tags Env=Dev
az group list --output table
az group delete --name myRG --yes --no-wait

# ─── APP SERVICE ───────────────────────────────────────────────
# Create
az appservice plan create --name myplan --resource-group myRG --sku B1 --is-linux
az webapp create --name myapp --resource-group myRG --plan myplan --runtime "DOTNETCORE:8.0"

# Configure
az webapp config set --name myapp -g myRG --always-on true --min-tls-version 1.2
az webapp update --name myapp -g myRG --https-only true
az webapp config appsettings set --name myapp -g myRG --settings "KEY=value"

# Deploy
az webapp deploy --name myapp -g myRG --src-path ./deploy.zip --type zip

# Logs
az webapp log tail --name myapp -g myRG
az webapp log download --name myapp -g myRG --log-file ./logs.zip

# Slots
az webapp deployment slot create --name myapp -g myRG --slot staging
az webapp deployment slot swap --name myapp -g myRG --slot staging

# Scaling
az appservice plan update --name myplan -g myRG --sku S1
az appservice plan update --name myplan -g myRG --number-of-workers 3

# ─── SQL DATABASE ──────────────────────────────────────────────
az sql server create --name myserver -g myRG -l eastus --admin-user sqladmin --admin-password "P@ss!"
az sql db create --name mydb -g myRG --server myserver --service-objective S0
az sql server firewall-rule create --name AllowMe -g myRG --server myserver --start-ip-address 1.2.3.4 --end-ip-address 1.2.3.4

# ─── KEY VAULT ─────────────────────────────────────────────────
az keyvault create --name mykv -g myRG -l eastus --enable-rbac-authorization true
az keyvault secret set --vault-name mykv --name "MySecret" --value "secret-value"
az keyvault secret show --vault-name mykv --name "MySecret" --query value -o tsv
az keyvault secret list --vault-name mykv --output table

# ─── CONTAINER REGISTRY (ACR) ─────────────────────────────────
az acr create --name myacr -g myRG --sku Basic
az acr login --name myacr
az acr build --registry myacr --image myapi:v1.0 .
az acr repository list --name myacr --output table
az acr repository show-tags --name myacr --repository myapi --output table

# ─── AKS ───────────────────────────────────────────────────────
# Create
az aks create --name myaks -g myRG --node-count 2 --node-vm-size Standard_B2s --attach-acr myacr --generate-ssh-keys --enable-managed-identity

# Connect
az aks get-credentials --name myaks -g myRG
az aks get-credentials --name myaks -g myRG --admin    # Admin access

# Scale
az aks scale --name myaks -g myRG --node-count 3
az aks nodepool scale --name nodepool1 --cluster-name myaks -g myRG --node-count 0

# Upgrade
az aks get-upgrades --name myaks -g myRG --output table
az aks upgrade --name myaks -g myRG --kubernetes-version 1.29.0

# ─── MANAGED IDENTITY ─────────────────────────────────────────
az webapp identity assign --name myapp -g myRG
az role assignment create --assignee <identity-id> --role "Key Vault Secrets User" --scope <keyvault-id>

# ─── APPLICATION INSIGHTS ─────────────────────────────────────
az monitor app-insights component create --app myai -g myRG -l eastus --application-type web
az monitor app-insights component show --app myai -g myRG --query connectionString -o tsv

# ─── COST ──────────────────────────────────────────────────────
az advisor recommendation list --category Cost --output table
```

---

## Docker

```powershell
# ─── BUILD ─────────────────────────────────────────────────────
docker build -t myapi:v1.0 .                              # Build image
docker build -t myacr.azurecr.io/myapi:v1.0 .             # Build with ACR tag
docker build --no-cache -t myapi:v1.0 .                    # Build without cache

# ─── RUN ───────────────────────────────────────────────────────
docker run -d -p 5000:8080 --name myapi myapi:v1.0         # Run container
docker run -d -p 5000:8080 -e "ASPNETCORE_ENVIRONMENT=Production" myapi:v1.0
docker run --rm -it myapi:v1.0 sh                          # Run and shell in

# ─── MANAGE ────────────────────────────────────────────────────
docker ps                                                   # List running containers
docker ps -a                                                # List all containers
docker logs myapi                                           # View logs
docker logs myapi -f                                        # Stream logs
docker stop myapi                                           # Stop container
docker rm myapi                                             # Remove container
docker images                                               # List images
docker rmi myapi:v1.0                                       # Remove image

# ─── PUSH TO ACR ──────────────────────────────────────────────
az acr login --name myacr
docker push myacr.azurecr.io/myapi:v1.0

# ─── CLEANUP ──────────────────────────────────────────────────
docker system prune -a                                      # Remove all unused data
```

---

## Kubernetes (kubectl)

```powershell
# ─── CLUSTER ───────────────────────────────────────────────────
kubectl cluster-info                           # Cluster info
kubectl get nodes                              # List nodes
kubectl top nodes                              # Node CPU/memory

# ─── PODS ──────────────────────────────────────────────────────
kubectl get pods -n myproject                  # List pods
kubectl get pods -n myproject -o wide          # More details (node, IP)
kubectl describe pod <name> -n myproject       # Full pod details
kubectl logs <name> -n myproject               # View logs
kubectl logs <name> -n myproject -f            # Stream logs
kubectl logs <name> -n myproject --previous    # Previous crash logs
kubectl exec -it <name> -n myproject -- sh     # Shell into pod
kubectl delete pod <name> -n myproject         # Delete (restarts)
kubectl top pods -n myproject                  # Pod CPU/memory

# ─── DEPLOYMENTS ───────────────────────────────────────────────
kubectl get deployments -n myproject
kubectl set image deployment/myapi -n myproject myapi=myacr.azurecr.io/myapi:v1.1
kubectl rollout status deployment/myapi -n myproject
kubectl rollout undo deployment/myapi -n myproject
kubectl rollout history deployment/myapi -n myproject
kubectl scale deployment/myapi -n myproject --replicas=5

# ─── SERVICES ─────────────────────────────────────────────────
kubectl get services -n myproject
kubectl describe service myapi-service -n myproject

# ─── CONFIG ────────────────────────────────────────────────────
kubectl get configmap -n myproject
kubectl get secrets -n myproject
kubectl get configmap myapi-config -n myproject -o yaml

# ─── APPLY / DELETE ───────────────────────────────────────────
kubectl apply -f k8s/                          # Apply all manifests
kubectl apply -f k8s/deployment.yaml           # Apply single file
kubectl delete -f k8s/                         # Delete all
kubectl delete namespace myproject             # Delete everything in namespace

# ─── EVERYTHING ───────────────────────────────────────────────
kubectl get all -n myproject                   # List all resources
kubectl get all --all-namespaces               # All resources in all namespaces
```

---

## .NET CLI

```powershell
# ─── PROJECT ───────────────────────────────────────────────────
dotnet new webapi -n MyProject.Api             # Create new Web API
dotnet new classlib -n MyProject.Core          # Create class library
dotnet new xunit -n MyProject.Tests            # Create test project
dotnet sln add src/MyProject.Api               # Add project to solution

# ─── BUILD & RUN ──────────────────────────────────────────────
dotnet restore                                  # Restore NuGet packages
dotnet build                                    # Build project
dotnet run                                      # Run project
dotnet watch run                                # Run with hot reload
dotnet publish -c Release -o ./publish          # Create release package
dotnet test                                     # Run tests

# ─── PACKAGES ─────────────────────────────────────────────────
dotnet add package Serilog.AspNetCore
dotnet add package Azure.Identity
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet list package                             # List installed packages
dotnet list package --vulnerable                # Check for vulnerabilities

# ─── USER SECRETS (local dev) ────────────────────────────────
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost..."
dotnet user-secrets list
```

---

## Git (Quick Reference)

```powershell
# ─── DAILY WORKFLOW ────────────────────────────────────────────
git status                                      # Check changes
git add -A                                      # Stage all changes
git commit -m "feat: add order endpoint"        # Commit
git push origin main                            # Push to remote
git pull origin main                            # Get latest changes

# ─── BRANCHES ─────────────────────────────────────────────────
git checkout -b feature/new-api                 # Create and switch branch
git checkout main                               # Switch to main
git merge feature/new-api                       # Merge branch into current
git branch -d feature/new-api                   # Delete branch

# ─── UNDO ─────────────────────────────────────────────────────
git stash                                       # Save changes temporarily
git stash pop                                   # Restore stashed changes
git reset HEAD~1                                # Undo last commit (keep changes)
git checkout -- <file>                          # Discard file changes
```
