# Helm Charts for Your .NET Project
## File 11: Package Your Kubernetes Deployment with Helm

---

## What You'll Do in This File

By the end of this guide, you'll have:
- ✅ Understood what Helm is and why it replaces raw YAML manifests
- ✅ Created a Helm chart for your .NET app
- ✅ Deployed using Helm instead of `kubectl apply`
- ✅ Managed different environments (dev/staging/prod) with values files
- ✅ Integrated Helm into your CI/CD pipeline

**Time Required:** 1-2 hours  
**Prerequisites:** File 04 completed (AKS running with your app)

---

## What is Helm? (And Why Should You Care?)

### The Problem with Raw YAML

In File 04, you created separate YAML files for your deployment:

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── deployment.yaml
├── service.yaml
└── hpa.yaml
```

This works, but has problems:

```
❌ Raw YAML Problems:
├── Duplicated values (image name repeated in deployment + CI/CD)
├── No versioning (which version of manifests is in production?)
├── Hard to manage environments (copy-paste YAML for dev/staging/prod)
├── No rollback history (kubectl apply doesn't track releases)
├── No dependency management (what if your app needs Redis too?)
└── Error-prone (YAML indentation mistakes break everything)
```

### Helm Solves This

Helm is a **package manager for Kubernetes** — like NuGet for .NET or npm for Node.js.

```
✅ Helm Solutions:
├── Templates with variables ({{ .Values.image.tag }} instead of hardcoded)
├── Versioned releases (helm history shows all deployments)
├── One command to install/upgrade/rollback
├── Values files per environment (values-dev.yaml, values-prod.yaml)
├── Dependency management (automatically install Redis, SQL, etc.)
└── Atomic deployments (all-or-nothing — no partial deployments)
```

### Helm vs kubectl — Quick Comparison

```
Raw kubectl:                          Helm:
─────────────                         ──────
kubectl apply -f k8s/                 helm upgrade --install myapi ./chart
kubectl apply -f k8s/deployment.yaml  helm upgrade myapi ./chart --set image.tag=v1.1
kubectl delete -f k8s/                helm uninstall myapi
(no rollback tracking)                helm rollback myapi 1
(copy YAML for each env)              helm upgrade myapi ./chart -f values-prod.yaml
```

---

## Step 1: Install Helm

```powershell
# Install Helm CLI
winget install Helm.Helm

# Verify installation
helm version

# Make sure kubectl is connected to your AKS cluster
kubectl get nodes
```

---

## Step 2: Create a Helm Chart

```powershell
# Create a new chart (scaffolds the folder structure)
helm create myapi-chart

# This creates:
# myapi-chart/
# ├── Chart.yaml           ← Chart metadata (name, version)
# ├── values.yaml           ← Default configuration values
# ├── charts/               ← Dependencies (sub-charts)
# ├── templates/            ← Kubernetes manifest templates
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── serviceaccount.yaml
# │   ├── _helpers.tpl      ← Template helper functions
# │   ├── NOTES.txt          ← Post-install message
# │   └── tests/
# │       └── test-connection.yaml
# └── .helmignore            ← Files to exclude from chart package
```

Now let's customize each file for your .NET app:

### Chart.yaml — Chart Metadata

```yaml
# myapi-chart/Chart.yaml
apiVersion: v2
name: myapi
description: My .NET API application deployed to AKS
type: application
version: 1.0.0        # Chart version (increment when you change templates)
appVersion: "1.0.0"    # Your app version (matches your Docker image tag)

# Optional: Add dependencies
# dependencies:
#   - name: redis
#     version: "18.x.x"
#     repository: https://charts.bitnami.com/bitnami
#     condition: redis.enabled
```

### values.yaml — Default Configuration

This is the most important file. It contains ALL configurable values:

```yaml
# myapi-chart/values.yaml — DEFAULT values (dev environment)

# ─── IMAGE ──────────────────────────────────────────────────────
image:
  repository: myprojectdevacr.azurecr.io/myapi   # ACR image path
  tag: "latest"                                    # Overridden in CI/CD
  pullPolicy: IfNotPresent

# ─── REPLICAS & SCALING ────────────────────────────────────────
replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# ─── RESOURCES ──────────────────────────────────────────────────
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# ─── SERVICE ───────────────────────────────────────────────────
service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

# ─── APPLICATION SETTINGS ─────────────────────────────────────
config:
  aspnetcoreEnvironment: "Production"
  appName: "MyProject API"
  maxPageSize: "50"
  logLevel: "Information"

# ─── SECRETS (reference names, not actual values!) ─────────────
secrets:
  connectionString: ""    # Set via --set or values-{env}.yaml
  appInsightsKey: ""

# ─── HEALTH CHECKS ────────────────────────────────────────────
healthCheck:
  liveness:
    path: /health/live
    port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
  readiness:
    path: /health/ready
    port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10

# ─── INGRESS (optional — set enabled: true to use) ────────────
ingress:
  enabled: false
  className: nginx
  host: api.myproject.com
  tls: false

# ─── NAMESPACE ─────────────────────────────────────────────────
namespace: myproject
```

### templates/deployment.yaml — Deployment Template

Replace the auto-generated content with:

```yaml
# myapi-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapi.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapi.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapi.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          
          # Environment variables from values
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.config.aspnetcoreEnvironment | quote }}
            - name: ApplicationSettings__AppName
              value: {{ .Values.config.appName | quote }}
            - name: ApplicationSettings__MaxPageSize
              value: {{ .Values.config.maxPageSize | quote }}
            - name: Logging__LogLevel__Default
              value: {{ .Values.config.logLevel | quote }}
            {{- if .Values.secrets.connectionString }}
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapi.fullname" . }}-secrets
                  key: connectionString
            {{- end }}
            {{- if .Values.secrets.appInsightsKey }}
            - name: APPLICATIONINSIGHTS_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapi.fullname" . }}-secrets
                  key: appInsightsKey
            {{- end }}
          
          # Resource limits
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          
          # Health checks
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.liveness.path }}
              port: {{ .Values.healthCheck.liveness.port }}
            initialDelaySeconds: {{ .Values.healthCheck.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readiness.path }}
              port: {{ .Values.healthCheck.readiness.port }}
            initialDelaySeconds: {{ .Values.healthCheck.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.readiness.periodSeconds }}
```

### templates/service.yaml

```yaml
# myapi-chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapi.fullname" . }}-service
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
  selector:
    {{- include "myapi.selectorLabels" . | nindent 4 }}
```

### templates/hpa.yaml

```yaml
# myapi-chart/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapi.fullname" . }}-hpa
  namespace: {{ .Values.namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapi.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

### templates/secret.yaml

```yaml
# myapi-chart/templates/secret.yaml
{{- if or .Values.secrets.connectionString .Values.secrets.appInsightsKey }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapi.fullname" . }}-secrets
  namespace: {{ .Values.namespace }}
type: Opaque
data:
  {{- if .Values.secrets.connectionString }}
  connectionString: {{ .Values.secrets.connectionString | b64enc | quote }}
  {{- end }}
  {{- if .Values.secrets.appInsightsKey }}
  appInsightsKey: {{ .Values.secrets.appInsightsKey | b64enc | quote }}
  {{- end }}
{{- end }}
```

### templates/namespace.yaml

```yaml
# myapi-chart/templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
```

---

## Step 3: Create Environment-Specific Values

### values-dev.yaml

```yaml
# myapi-chart/values-dev.yaml — DEV environment overrides
image:
  repository: myprojectdevacr.azurecr.io/myapi
  tag: "latest"

replicaCount: 1

autoscaling:
  enabled: false          # No auto-scaling in dev

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"

config:
  aspnetcoreEnvironment: "Development"
  logLevel: "Debug"

namespace: myproject-dev
```

### values-staging.yaml

```yaml
# myapi-chart/values-staging.yaml — STAGING environment overrides
image:
  repository: myprojectdevacr.azurecr.io/myapi
  tag: "v1.0.0"           # Pinned version

replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5

config:
  aspnetcoreEnvironment: "Staging"
  logLevel: "Information"

namespace: myproject-staging
```

### values-prod.yaml

```yaml
# myapi-chart/values-prod.yaml — PRODUCTION environment overrides
image:
  repository: myprojectprodacr.azurecr.io/myapi
  tag: "v1.0.0"           # Always pinned version — never "latest"

replicaCount: 3

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

config:
  aspnetcoreEnvironment: "Production"
  logLevel: "Warning"

namespace: myproject-prod
```

---

## Step 4: Deploy with Helm

```powershell
# ─── VALIDATE YOUR CHART ──────────────────────────────────────

# Check for syntax errors
helm lint myapi-chart/

# Preview what Kubernetes YAML will be generated (without deploying)
helm template myapi myapi-chart/ -f myapi-chart/values-dev.yaml

# Dry run (sends to K8s API but doesn't create anything)
helm install myapi myapi-chart/ -f myapi-chart/values-dev.yaml --dry-run

# ─── INSTALL (first deployment) ───────────────────────────────

helm install myapi myapi-chart/ -f myapi-chart/values-dev.yaml

# Or install with secret values passed securely (not in files):
helm install myapi myapi-chart/ `
  -f myapi-chart/values-dev.yaml `
  --set secrets.connectionString="Server=tcp:mydb.database.windows.net..." `
  --set secrets.appInsightsKey="InstrumentationKey=..."

# ─── UPGRADE (update existing deployment) ─────────────────────

# Deploy a new version
helm upgrade myapi myapi-chart/ `
  -f myapi-chart/values-dev.yaml `
  --set image.tag="v1.1"

# ─── INSTALL OR UPGRADE (recommended — works for both) ────────

helm upgrade --install myapi myapi-chart/ `
  -f myapi-chart/values-dev.yaml `
  --set image.tag="v1.1"

# Deploy to different environments:
helm upgrade --install myapi myapi-chart/ -f myapi-chart/values-staging.yaml
helm upgrade --install myapi myapi-chart/ -f myapi-chart/values-prod.yaml
```

---

## Step 5: Manage Releases

```powershell
# ─── VIEW ──────────────────────────────────────────────────────
helm list                              # List all releases
helm list --all-namespaces             # All releases across namespaces
helm status myapi                      # Status of a release
helm history myapi                     # All versions (rollback targets)

# ─── ROLLBACK ─────────────────────────────────────────────────
helm rollback myapi 1                  # Rollback to revision 1
helm rollback myapi 0                  # Rollback to previous version

# ─── UNINSTALL ────────────────────────────────────────────────
helm uninstall myapi                   # Remove everything

# ─── VIEW GENERATED YAML ─────────────────────────────────────
helm get manifest myapi                # See what's deployed
helm get values myapi                  # See current values
```

---

## Step 6: Add Helm to Your CI/CD Pipeline

### Azure DevOps

```yaml
# Add to your azure-pipelines.yml deploy stage
- task: HelmDeploy@0
  displayName: 'Helm upgrade'
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscription: $(azureSubscription)
    azureResourceGroup: $(resourceGroup)
    kubernetesCluster: $(aksCluster)
    namespace: 'myproject'
    command: 'upgrade'
    chartType: 'FilePath'
    chartPath: 'myapi-chart/'
    releaseName: 'myapi'
    overrideValues: 'image.tag=$(Build.BuildId)'
    valueFile: 'myapi-chart/values-dev.yaml'
    install: true
    waitForExecution: true
```

### GitHub Actions

```yaml
# Add to your .github/workflows/deploy-aks.yml
- name: Deploy with Helm
  run: |
    helm upgrade --install myapi ./myapi-chart/ \
      -f ./myapi-chart/values-dev.yaml \
      --set image.tag=${{ github.sha }} \
      --namespace myproject \
      --wait --timeout 300s
```

---

## Project Structure (Updated)

```
MyProject/
├── MyProject.sln
├── Dockerfile                     ← Docker build recipe
├── .dockerignore
├── src/
│   ├── MyProject.Api/
│   └── MyProject.Core/
├── tests/
│   └── MyProject.Tests/
├── k8s/                           ← Raw manifests (keep as backup)
│   ├── deployment.yaml
│   └── service.yaml
└── myapi-chart/                   ← Helm chart (use this for deployments!)
    ├── Chart.yaml
    ├── values.yaml                ← Default values
    ├── values-dev.yaml            ← Dev overrides
    ├── values-staging.yaml        ← Staging overrides
    ├── values-prod.yaml           ← Production overrides
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── hpa.yaml
        ├── secret.yaml
        ├── namespace.yaml
        └── _helpers.tpl
```

---

## ✅ Helm Checklist

- [ ] Helm CLI installed and working
- [ ] Chart created with `helm create`
- [ ] `values.yaml` has all configurable values
- [ ] Per-environment values files created (dev, staging, prod)
- [ ] `helm lint` passes with no errors
- [ ] `helm template` generates correct YAML
- [ ] First deployment works with `helm install`
- [ ] Upgrade works with `helm upgrade --install`
- [ ] Rollback tested with `helm rollback`
- [ ] CI/CD pipeline uses Helm for deployment

---

## Quick Reference — Helm Commands

```powershell
helm create myapi-chart                        # Create new chart
helm lint myapi-chart/                         # Validate chart
helm template myapi myapi-chart/ -f values.yaml  # Preview YAML
helm install myapi myapi-chart/ -f values.yaml # First deploy
helm upgrade --install myapi myapi-chart/ -f values.yaml --set image.tag=v1.1  # Deploy/update
helm list                                      # List releases
helm history myapi                             # Release history
helm rollback myapi 1                          # Rollback
helm uninstall myapi                           # Remove release
helm get values myapi                          # Show current values
helm get manifest myapi                        # Show deployed YAML
```

---

> **Reference:** For deep-dive Helm topics (chart dependencies, Go templating, OCI registries), see [Azure Learning/21-HELM-CHARTS-COMPLETE-GUIDE.md](../Azure%20Learning/21-HELM-CHARTS-COMPLETE-GUIDE.md)
