# Deploy .NET App to AKS (Kubernetes)
## File 04: Container Deployment Step-by-Step

---

## What You'll Do in This File

By the end of this guide, your .NET app will be:
- ✅ Containerized in Docker and pushed to Azure Container Registry
- ✅ Running on AKS (Azure Kubernetes Service)
- ✅ Accessible via a load balancer with a public IP
- ✅ Auto-scaling based on CPU/memory

**Time Required:** 2-3 hours  
**Prerequisites:** File 02 completed (app prepared + Dockerfile created)

---

## What is AKS?

AKS (Azure Kubernetes Service) is a **managed Kubernetes cluster**. Kubernetes orchestrates your Docker containers — starting, stopping, scaling, and networking them automatically.

```
Without Kubernetes:                    With Kubernetes (AKS):
──────────────────                     ─────────────────────
You manually run Docker                K8s automatically runs containers
containers on VMs                      across multiple nodes

You restart crashed containers         K8s auto-restarts crashed containers

You manually scale up/down             K8s auto-scales based on CPU/traffic

You configure networking               K8s provides built-in service discovery
between containers manually            and load balancing

You manage each VM                     Azure manages the VMs (nodes) for you
```

### Key Kubernetes Concepts (Simplified)

```
AKS Cluster
├── Node Pool (VMs that run your containers)
│   ├── Node 1 (VM)
│   │   ├── Pod: myapi (your app container)
│   │   └── Pod: myapi (replica for scaling)
│   └── Node 2 (VM)
│       └── Pod: myapi (another replica)
│
├── Service (Load Balancer — routes traffic to pods)
│   └── Public IP → distributes to all myapi pods
│
├── Deployment (tells K8s how to run your app)
│   ├── Image: myacr.azurecr.io/myapi:v1.0
│   ├── Replicas: 3
│   └── Resources: 0.5 CPU, 256MB RAM per pod
│
└── ConfigMap / Secret (configuration and secrets)
```

| Concept | Analogy | Purpose |
|---------|---------|---------|
| **Cluster** | A data center | The entire Kubernetes environment |
| **Node** | A server/VM | Machine that runs containers |
| **Pod** | A process | One running instance of your app |
| **Deployment** | A job description | "Run 3 copies of my app using this image" |
| **Service** | A phone number | Stable endpoint that routes to pods |
| **Namespace** | A folder | Logical grouping (dev, staging, prod) |

---

## Step 1: Create Azure Infrastructure

```powershell
# ─── Variables ──────────────────────────────────────────────────
$PROJECT = "myproject"
$ENV = "dev"
$RG = "$PROJECT-$ENV-rg"
$LOCATION = "eastus"

$ACR_NAME = "${PROJECT}${ENV}acr"           # No hyphens! e.g., myprojectdevacr
$AKS_NAME = "$PROJECT-$ENV-aks"
$KEYVAULT = "$PROJECT-$ENV-kv"

# ─── Step 1a: Create Azure Container Registry (ACR) ───────────
# ACR stores your Docker images (like a private Docker Hub)

az acr create `
  --name $ACR_NAME `
  --resource-group $RG `
  --location $LOCATION `
  --sku Basic

# Basic = $5/month, 10 GB storage — good for dev/test
# Standard = $20/month, 100 GB — good for production

# ─── Step 1b: Create AKS Cluster ──────────────────────────────

az aks create `
  --name $AKS_NAME `
  --resource-group $RG `
  --location $LOCATION `
  --node-count 2 `
  --node-vm-size Standard_B2s `
  --generate-ssh-keys `
  --attach-acr $ACR_NAME `
  --enable-managed-identity `
  --network-plugin azure `
  --enable-addons monitoring

# This takes 5-10 minutes. Let it run.
```

### Understanding the AKS Options

```
--node-count 2        → 2 VMs in the cluster (min for reliability)
--node-vm-size B2s    → Each VM: 2 vCPUs, 4 GB RAM (~$30/month each)
--attach-acr          → AKS can pull images from ACR (no password needed)
--enable-managed-identity → Secure auth without passwords
--network-plugin azure    → Azure CNI networking (pods get VNet IPs)
--enable-addons monitoring → Container Insights for monitoring
```

```powershell
# ─── Step 1c: Get AKS Credentials (connect kubectl) ───────────

az aks get-credentials `
  --name $AKS_NAME `
  --resource-group $RG

# Verify connection
kubectl get nodes
# Should show 2 nodes in "Ready" state

# ─── Step 1d: Create Key Vault (if not already created) ───────

az keyvault create `
  --name $KEYVAULT `
  --resource-group $RG `
  --location $LOCATION `
  --enable-rbac-authorization true
```

---

## Step 2: Build and Push Docker Image

```powershell
# ─── Option A: Build locally and push ─────────────────────────

# Login to ACR
az acr login --name $ACR_NAME

# Build the Docker image
docker build -t "${ACR_NAME}.azurecr.io/myapi:v1.0" .

# Push to ACR
docker push "${ACR_NAME}.azurecr.io/myapi:v1.0"

# ─── Option B: Build in the cloud (no local Docker needed!) ───

az acr build `
  --registry $ACR_NAME `
  --image myapi:v1.0 `
  .

# Verify the image is in ACR
az acr repository list --name $ACR_NAME --output table
az acr repository show-tags --name $ACR_NAME --repository myapi --output table
```

> **Tip:** Option B (`az acr build`) is great for team members who don't have Docker Desktop installed.

---

## Step 3: Create Kubernetes Manifests

Create a `k8s/` folder in your project root with these files:

### k8s/namespace.yaml

```yaml
# Namespace = a "folder" to organize your app's resources
apiVersion: v1
kind: Namespace
metadata:
  name: myproject
  labels:
    app: myproject
    environment: dev
```

### k8s/configmap.yaml

```yaml
# ConfigMap = non-secret configuration (injected as environment variables)
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
  namespace: myproject
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ApplicationSettings__AppName: "MyProject API"
  ApplicationSettings__MaxPageSize: "50"
  Logging__LogLevel__Default: "Information"
```

### k8s/secret.yaml

```yaml
# Secret = sensitive data (connection strings, passwords)
# Values must be base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
  namespace: myproject
type: Opaque
data:
  # To encode: echo -n "your-value" | base64
  ConnectionStrings__DefaultConnection: <base64-encoded-connection-string>
```

```powershell
# Generate base64 encoded value:
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("Server=tcp:myproject-dev-sql.database.windows.net,1433;Database=myproject-dev-db;User ID=sqladmin;Password=YourP@ss;Encrypt=True;"))
```

### k8s/deployment.yaml

```yaml
# Deployment = tells K8s HOW to run your app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  namespace: myproject
  labels:
    app: myapi
spec:
  replicas: 2                    # Run 2 instances of your app
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
        - name: myapi
          image: myprojectdevacr.azurecr.io/myapi:v1.0    # ← Change ACR name!
          ports:
            - containerPort: 8080
          
          # Environment variables from ConfigMap and Secret
          envFrom:
            - configMapRef:
                name: myapi-config
            - secretRef:
                name: myapi-secrets
          
          # Resource limits (important for auto-scaling!)
          resources:
            requests:              # Minimum resources guaranteed
              cpu: "250m"          # 0.25 CPU cores
              memory: "256Mi"      # 256 MB RAM
            limits:                # Maximum resources allowed
              cpu: "500m"          # 0.5 CPU cores
              memory: "512Mi"      # 512 MB RAM
          
          # Health checks — K8s uses these to manage your app
          livenessProbe:           # "Is the process alive?"
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15  # Wait 15 sec before first check
            periodSeconds: 10        # Check every 10 sec
            failureThreshold: 3      # Restart after 3 failures
          
          readinessProbe:          # "Is the app ready for traffic?"
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
```

### k8s/service.yaml

```yaml
# Service = makes your app accessible (load balancer)
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
  namespace: myproject
spec:
  type: LoadBalancer             # Creates a public IP
  selector:
    app: myapi                   # Routes traffic to pods with label app=myapi
  ports:
    - port: 80                   # External port (users access this)
      targetPort: 8080           # Internal port (your app listens on this)
      protocol: TCP
```

### k8s/hpa.yaml

```yaml
# HorizontalPodAutoscaler = auto-scaling based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: myproject
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi
  minReplicas: 2                 # Always run at least 2 pods
  maxReplicas: 10                # Scale up to 10 pods max
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # Scale when CPU > 70%
```

---

## Step 4: Deploy to AKS

```powershell
# ─── Apply all manifests ───────────────────────────────────────

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# Or apply everything at once:
kubectl apply -f k8s/
```

---

## Step 5: Verify Your Deployment

```powershell
# ─── Check pod status ─────────────────────────────────────────

kubectl get pods -n myproject
# Expected output:
# NAME                     READY   STATUS    RESTARTS   AGE
# myapi-7d8f9b6c4d-abc12   1/1     Running   0          1m
# myapi-7d8f9b6c4d-def34   1/1     Running   0          1m

# If STATUS is not "Running", check logs:
kubectl logs -n myproject <pod-name>

# ─── Get the public IP ─────────────────────────────────────────

kubectl get service myapi-service -n myproject
# Wait for EXTERNAL-IP to change from <pending> to an actual IP
# This can take 1-3 minutes

# ─── Test the app ──────────────────────────────────────────────

$EXTERNAL_IP = kubectl get service myapi-service -n myproject -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
curl "http://$EXTERNAL_IP/health/live"
# Expected: Healthy

# ─── Check HPA ─────────────────────────────────────────────────

kubectl get hpa -n myproject
# Shows current CPU% and replica count
```

---

## Step 6: Common kubectl Commands (Daily Use)

```powershell
# ─── VIEWING ───────────────────────────────────────────────────
kubectl get pods -n myproject                   # List pods
kubectl get services -n myproject               # List services
kubectl get deployments -n myproject            # List deployments
kubectl get all -n myproject                    # List everything

# ─── DEBUGGING ─────────────────────────────────────────────────
kubectl logs <pod-name> -n myproject            # View logs
kubectl logs <pod-name> -n myproject -f         # Stream logs (like tail -f)
kubectl describe pod <pod-name> -n myproject    # Detailed pod info (events, status)
kubectl exec -it <pod-name> -n myproject -- sh  # Shell into a running container

# ─── UPDATING ─────────────────────────────────────────────────
# Deploy a new version (change the image tag)
kubectl set image deployment/myapi -n myproject myapi=myprojectdevacr.azurecr.io/myapi:v1.1

# Or edit the deployment.yaml and re-apply
kubectl apply -f k8s/deployment.yaml

# Check rollout status
kubectl rollout status deployment/myapi -n myproject

# ─── ROLLBACK ─────────────────────────────────────────────────
kubectl rollout undo deployment/myapi -n myproject    # Rollback to previous version
kubectl rollout history deployment/myapi -n myproject  # View rollout history

# ─── SCALING ──────────────────────────────────────────────────
kubectl scale deployment/myapi -n myproject --replicas=5   # Manual scale

# ─── DELETING ─────────────────────────────────────────────────
kubectl delete -f k8s/                           # Delete everything from manifests
kubectl delete namespace myproject               # Delete entire namespace
```

---

## Step 7: Update Your App (Deploy New Version)

When you have new code to deploy:

```powershell
# 1. Build new image with new tag
az acr build `
  --registry $ACR_NAME `
  --image myapi:v1.1 `
  .

# 2. Update the deployment to use the new image
kubectl set image deployment/myapi -n myproject `
  myapi="${ACR_NAME}.azurecr.io/myapi:v1.1"

# 3. Watch the rollout (K8s replaces pods one by one = zero downtime)
kubectl rollout status deployment/myapi -n myproject

# 4. Verify
kubectl get pods -n myproject
curl "http://$EXTERNAL_IP/health/live"
```

---

## 📊 Cost Estimate (Dev Environment)

| Resource | SKU | Monthly Cost |
|----------|-----|-------------|
| AKS Cluster (2x B2s nodes) | Standard_B2s | ~$60 |
| Container Registry | Basic | ~$5 |
| Load Balancer | Standard | ~$18 |
| SQL Database | S0 | ~$15 |
| Key Vault | Standard | ~$0.03 |
| **Total (Dev)** | | **~$98/month** |

---

## ✅ AKS Deployment Checklist

- [ ] ACR created and image pushed
- [ ] AKS cluster created and kubectl connected
- [ ] Namespace created
- [ ] ConfigMap and Secret applied
- [ ] Deployment applied with health probes
- [ ] Service created with LoadBalancer
- [ ] External IP accessible and health check passing
- [ ] HPA configured for auto-scaling

---

> **Next Step:** Set up CI/CD to automate deployments → [05-CI-CD-PIPELINE-SETUP.md](05-CI-CD-PIPELINE-SETUP.md)
