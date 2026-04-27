# Helm Charts — Complete Guide
## Document 21: Package Manager for Kubernetes

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Helm concepts, chart creation, deployment, and best practices

---

## TABLE OF CONTENTS

1. [What is Helm and Why Use It?](#what-is-helm)
2. [Helm Architecture](#architecture)
3. [Installing Helm](#installation)
4. [Helm Chart Structure](#chart-structure)
5. [Creating Your First Chart](#creating-chart)
6. [Templates and Values](#templates-values)
7. [Built-in Objects and Functions](#built-in)
8. [Chart Dependencies](#dependencies)
9. [Helm Repositories](#repositories)
10. [Helm Commands Reference](#commands)
11. [Helm in CI/CD Pipelines](#cicd)
12. [Best Practices](#best-practices)
13. [Real-World Example: AzureLearn App](#real-world)

---

## What is Helm and Why Use It? {#what-is-helm}

### Definition

**Helm** = The package manager for Kubernetes (like npm for Node.js, NuGet for .NET, or apt for Linux)

Instead of managing dozens of raw YAML files, Helm bundles them into a single deployable **chart**.

### Without Helm ❌

```
Deploying an application requires:
├─ namespace.yaml
├─ configmap.yaml
├─ secret.yaml
├─ serviceaccount.yaml
├─ deployment.yaml
├─ service.yaml
├─ hpa.yaml
├─ ingress.yaml
├─ networkpolicy.yaml
└─ ...more YAML files

Problems:
├─ 10+ kubectl apply commands
├─ Hard to version or rollback as a unit
├─ Environment differences (dev/prod) require duplicate files
├─ No dependency management
├─ No release history
└─ Manual, error-prone process
```

### With Helm ✅

```
helm install my-app ./my-chart \
  --values values-prod.yaml

One command:
├─ All resources created together
├─ Versioned as a single release
├─ Easy rollback: helm rollback my-app 1
├─ Environment overrides via values files
├─ Dependency management built in
└─ Full release history tracked
```

### Key Concepts

| Concept | Definition | Analogy |
|---------|-----------|---------|
| **Chart** | Package of K8s resource templates | A NuGet package |
| **Release** | Running instance of a chart | An installed app |
| **Repository** | Collection of charts | NuGet gallery |
| **Values** | Configuration for a chart | appsettings.json |
| **Template** | K8s YAML with Go templating | Razor views in .NET |

---

## Helm Architecture {#architecture}

### Helm 3 (Current)

```
┌──────────────────────────────────┐
│         Developer / CI/CD        │
│                                  │
│  $ helm install my-app ./chart   │
└──────────────┬───────────────────┘
               │
               ↓
┌──────────────────────────────────┐
│          Helm CLI (v3)           │
│                                  │
│  1. Read chart templates         │
│  2. Merge with values.yaml       │
│  3. Render final YAML            │
│  4. Send to Kubernetes API       │
│  5. Store release in K8s Secret  │
└──────────────┬───────────────────┘
               │
               ↓
┌──────────────────────────────────┐
│      Kubernetes API Server       │
│                                  │
│  Creates resources:              │
│  ├─ Deployment                   │
│  ├─ Service                      │
│  ├─ ConfigMap                    │
│  ├─ HPA                         │
│  └─ ...                         │
└──────────────────────────────────┘
```

> **Note:** Helm 2 used a server-side component called **Tiller** — this was removed in Helm 3 for security.

---

## Installing Helm {#installation}

```bash
# Windows (Chocolatey)
choco install kubernetes-helm

# Windows (winget)
winget install Helm.Helm

# macOS (Homebrew)
brew install helm

# Linux (script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
# version.BuildInfo{Version:"v3.16.x", ...}
```

---

## Helm Chart Structure {#chart-structure}

### Directory Layout

```
my-app-chart/
├── Chart.yaml            # Chart metadata (name, version, description)
├── values.yaml           # Default configuration values
├── charts/               # Dependency charts (sub-charts)
├── templates/            # Kubernetes manifest templates
│   ├── _helpers.tpl      # Reusable template snippets
│   ├── deployment.yaml   # Deployment template
│   ├── service.yaml      # Service template
│   ├── configmap.yaml    # ConfigMap template
│   ├── hpa.yaml          # HPA template
│   ├── ingress.yaml      # Ingress template (optional)
│   ├── serviceaccount.yaml
│   ├── NOTES.txt         # Post-install message shown to user
│   └── tests/
│       └── test-connection.yaml  # Helm test
├── .helmignore           # Files to exclude from chart package
└── README.md             # Chart documentation
```

### Chart.yaml — Chart Metadata

```yaml
apiVersion: v2                 # Helm 3 uses v2
name: azure-learn-app          # Chart name
description: Azure Learning App deployed on AKS
type: application              # application or library
version: 1.0.0                 # Chart version (SemVer)
appVersion: "3.0"              # Application version
keywords:
  - dotnet
  - aks
  - cosmosdb
maintainers:
  - name: Your Name
    email: you@company.com
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

**Version vs appVersion:**
- `version` — Chart packaging version (when templates change)
- `appVersion` — Your application version (v3.0 of your .NET app)

---

## Creating Your First Chart {#creating-chart}

### Scaffold a New Chart

```bash
# Create chart skeleton
helm create azure-learn-app

# Output:
# Creating azure-learn-app
# azure-learn-app/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# ├── templates/
# │   ├── _helpers.tpl
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── serviceaccount.yaml
# │   ├── NOTES.txt
# │   └── tests/
# │       └── test-connection.yaml
# └── .helmignore
```

---

## Templates and Values {#templates-values}

### values.yaml — Default Configuration

```yaml
# values.yaml — defaults for all environments
replicaCount: 2

image:
  repository: azurelearnacrhof7rpcc.azurecr.io/azurelearn
  tag: "v3.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: acr-pull-secret

serviceAccount:
  create: true
  name: "azure-learn-app-sa"
  annotations:
    azure.workload.identity/client-id: ""

podAnnotations:
  azure.workload.identity/use: "true"

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: api.azurelearn.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"

configMap:
  keyVaultUrl: "https://azurelearnkvhof7rpcc.vault.azure.net/"
  cosmosDbDatabaseName: "AzureLearnDb"
  cosmosDbContainerName: "Products"

probes:
  liveness:
    path: /health/live
    port: 8080
    initialDelaySeconds: 10
    periodSeconds: 30
  readiness:
    path: /health/ready
    port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10

nodeSelector:
  kubernetes.io/os: linux
```

### Environment Overrides

```yaml
# values-dev.yaml — overrides for development
replicaCount: 1

image:
  tag: "latest"

service:
  type: ClusterIP

autoscaling:
  enabled: false

env:
  ASPNETCORE_ENVIRONMENT: "Development"

configMap:
  keyVaultUrl: "https://azurelearnkv-dev.vault.azure.net/"

resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

```yaml
# values-prod.yaml — overrides for production
replicaCount: 3

image:
  tag: "v3.0"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.azurelearn.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: azurelearn-tls
      hosts:
        - api.azurelearn.com
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "azure-learn-app.fullname" . }}
  labels:
    {{- include "azure-learn-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "azure-learn-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
      labels:
        {{- include "azure-learn-app.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "azure-learn-app.serviceAccountName" . }}
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "azure-learn-app.fullname" . }}-config
          {{- range $key, $value := .Values.env }}
          env:
            - name: {{ $key }}
              value: {{ $value | quote }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.probes.liveness.port }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.probes.readiness.port }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "azure-learn-app.fullname" . }}
  labels:
    {{- include "azure-learn-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "azure-learn-app.selectorLabels" . | nindent 4 }}
```

### templates/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "azure-learn-app.fullname" . }}-config
  labels:
    {{- include "azure-learn-app.labels" . | nindent 4 }}
data:
  KeyVault__Url: {{ .Values.configMap.keyVaultUrl | quote }}
  CosmosDb__DatabaseName: {{ .Values.configMap.cosmosDbDatabaseName | quote }}
  CosmosDb__ContainerName: {{ .Values.configMap.cosmosDbContainerName | quote }}
```

### templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "azure-learn-app.fullname" . }}
  labels:
    {{- include "azure-learn-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "azure-learn-app.fullname" . }}
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

### templates/_helpers.tpl — Reusable Snippets

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "azure-learn-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a fully qualified app name.
*/}}
{{- define "azure-learn-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "azure-learn-app.labels" -}}
helm.sh/chart: {{ include "azure-learn-app.name" . }}-{{ .Chart.Version }}
{{ include "azure-learn-app.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "azure-learn-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "azure-learn-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "azure-learn-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "azure-learn-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### templates/NOTES.txt — Post-Install Message

```
🚀 {{ .Chart.Name }} deployed successfully!

Release: {{ .Release.Name }}
Version: {{ .Chart.AppVersion }}

{{- if eq .Values.service.type "LoadBalancer" }}
Get the external IP:
  kubectl get svc {{ include "azure-learn-app.fullname" . }} -n {{ .Release.Namespace }}
{{- else }}
Port-forward to access locally:
  kubectl port-forward svc/{{ include "azure-learn-app.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
{{- end }}

API endpoint: http://<EXTERNAL-IP>/api/products
Health check: http://<EXTERNAL-IP>/health/live
```

---

## Built-in Objects and Functions {#built-in}

### Built-in Objects

| Object | Description | Example |
|--------|------------|---------|
| `.Release.Name` | Name given at install | `my-app` |
| `.Release.Namespace` | Target namespace | `azure-learn-app` |
| `.Release.Revision` | Release revision number | `3` |
| `.Chart.Name` | Chart name from Chart.yaml | `azure-learn-app` |
| `.Chart.Version` | Chart version | `1.0.0` |
| `.Chart.AppVersion` | App version | `3.0` |
| `.Values` | Merged values | `values.yaml` content |
| `.Template.Name` | Current template filename | `templates/deployment.yaml` |

### Common Template Functions

```yaml
# String functions
{{ .Values.image.tag | upper }}          # "V3.0"
{{ .Values.image.tag | quote }}          # "v3.0" (with quotes)
{{ .Values.name | trunc 63 }}           # Truncate to 63 chars
{{ .Values.name | trimSuffix "-" }}      # Remove trailing dash

# Default values
{{ .Values.env | default "Production" }}

# Conditional
{{- if .Values.ingress.enabled }}
  # render ingress
{{- end }}

# Loops
{{- range .Values.ingress.hosts }}
  - host: {{ .host }}
{{- end }}

# Indentation
{{ toYaml .Values.resources | nindent 12 }}

# Include named template
{{ include "azure-learn-app.labels" . | nindent 4 }}
```

---

## Chart Dependencies {#dependencies}

### Adding a Dependency (e.g., Redis)

In `Chart.yaml`:
```yaml
dependencies:
  - name: redis
    version: "18.6.1"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# Download dependencies
helm dependency update ./azure-learn-app

# Dependencies downloaded to charts/ folder
```

In `values.yaml`:
```yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
```

---

## Helm Repositories {#repositories}

```bash
# Add popular repos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update repos
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress    # Search Artifact Hub

# List added repos
helm repo list
```

---

## Helm Commands Reference {#commands}

### Install / Upgrade / Rollback

```bash
# Install chart
helm install my-app ./azure-learn-app \
  --namespace azure-learn-app \
  --create-namespace

# Install with overrides
helm install my-app ./azure-learn-app \
  --namespace azure-learn-app \
  --values values-prod.yaml \
  --set image.tag=v4.0

# Upgrade existing release
helm upgrade my-app ./azure-learn-app \
  --namespace azure-learn-app \
  --values values-prod.yaml

# Install or upgrade (idempotent — best for CI/CD)
helm upgrade --install my-app ./azure-learn-app \
  --namespace azure-learn-app \
  --create-namespace \
  --values values-prod.yaml \
  --wait --timeout 5m

# Rollback to previous revision
helm rollback my-app 1 --namespace azure-learn-app

# Uninstall
helm uninstall my-app --namespace azure-learn-app
```

### Inspect and Debug

```bash
# List releases
helm list --namespace azure-learn-app
helm list --all-namespaces

# Release history
helm history my-app --namespace azure-learn-app

# Show rendered YAML (dry-run, no install)
helm template my-app ./azure-learn-app --values values-prod.yaml

# Dry-run install (validates against cluster)
helm install my-app ./azure-learn-app --dry-run --debug

# Show chart info
helm show chart ./azure-learn-app
helm show values ./azure-learn-app

# Get release values
helm get values my-app --namespace azure-learn-app
helm get manifest my-app --namespace azure-learn-app
```

### Package and Publish

```bash
# Lint chart for errors
helm lint ./azure-learn-app

# Package chart to .tgz
helm package ./azure-learn-app
# Output: azure-learn-app-1.0.0.tgz

# Push to ACR as OCI artifact
helm push azure-learn-app-1.0.0.tgz oci://azurelearnacrhof7rpcc.azurecr.io/helm
```

---

## Helm in CI/CD Pipelines {#cicd}

### Azure DevOps Pipeline

```yaml
- task: HelmDeploy@0
  displayName: 'Helm upgrade --install'
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscription: 'AzureLearnConnection'
    azureResourceGroup: 'azure-learn-rg-dev'
    kubernetesCluster: 'azurelearn-dev-aks'
    namespace: 'azure-learn-app'
    command: 'upgrade'
    chartType: 'FilePath'
    chartPath: 'helm/azure-learn-app'
    releaseName: 'azure-learn-app'
    overrideValues: 'image.tag=$(Build.BuildId)'
    valueFile: 'helm/azure-learn-app/values-prod.yaml'
    arguments: '--install --wait --timeout 5m'
```

### GitHub Actions

```yaml
- name: Deploy with Helm
  run: |
    az aks get-credentials \
      --resource-group azure-learn-rg-dev \
      --name azurelearn-dev-aks
    helm upgrade --install azure-learn-app ./helm/azure-learn-app \
      --namespace azure-learn-app \
      --create-namespace \
      --values ./helm/azure-learn-app/values-prod.yaml \
      --set image.tag=${{ github.sha }} \
      --wait --timeout 5m
```

---

## Best Practices {#best-practices}

```
✅ Chart Design
├─ Use helm create as starting point
├─ Keep values.yaml well-documented with comments
├─ Use _helpers.tpl for repeated label blocks
├─ Always lint before packaging: helm lint
├─ Use SemVer for chart versions
└─ Include NOTES.txt for post-install guidance

✅ Values Management
├─ Provide sane defaults in values.yaml
├─ Use per-environment override files (values-dev.yaml, values-prod.yaml)
├─ Never put secrets in values — use External Secrets Operator or CSI
├─ Validate required values with required function
└─ Document every value with inline comments

✅ CI/CD
├─ Use helm upgrade --install for idempotent deploys
├─ Always pass --wait so pipeline fails if pods don't start
├─ Use --atomic to auto-rollback on failure
├─ Pin chart dependencies to exact versions
└─ Store charts in ACR as OCI artifacts

✅ Security
├─ Never store secrets in values.yaml
├─ Use Helm Secrets plugin or External Secrets Operator
├─ Sign charts with helm package --sign
├─ Scan charts with checkov or kubesec
└─ Use .helmignore to exclude sensitive files
```

---

## Real-World Example: AzureLearn App {#real-world}

### Deploy to Dev

```bash
helm upgrade --install azure-learn-app ./helm/azure-learn-app \
  --namespace azure-learn-app \
  --create-namespace \
  --values ./helm/azure-learn-app/values-dev.yaml \
  --set image.tag=latest \
  --wait
```

### Deploy to Production

```bash
helm upgrade --install azure-learn-app ./helm/azure-learn-app \
  --namespace azure-learn-app \
  --values ./helm/azure-learn-app/values-prod.yaml \
  --set image.tag=v3.0 \
  --wait --timeout 10m --atomic
```

### Rollback Production

```bash
# View history
helm history azure-learn-app -n azure-learn-app

# Rollback to previous
helm rollback azure-learn-app 0 -n azure-learn-app
```

---

## Key Takeaways

✅ **Helm** packages multiple K8s manifests into a single versioned release  
✅ **values.yaml** drives all environment-specific configuration  
✅ **Templates** use Go templating for dynamic YAML generation  
✅ **helm upgrade --install** is the idempotent CI/CD best practice  
✅ **Rollbacks** are instant with `helm rollback`  
✅ **OCI registries** (like ACR) can store Helm charts alongside Docker images  

**Next:** Use Document 22 for a step-by-step guide to Dockerize and deploy any .NET app to AKS.
