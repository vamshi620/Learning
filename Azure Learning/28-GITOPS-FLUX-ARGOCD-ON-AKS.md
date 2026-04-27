# GitOps with Flux & ArgoCD on AKS
## Document 28: Pull-Based Continuous Deployment for Kubernetes

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** GitOps principles, Flux CD, ArgoCD, AKS integration, multi-environment management

---

## TABLE OF CONTENTS

1. [What is GitOps?](#1-what-is-gitops)
2. [Push-Based vs Pull-Based Deployment](#2-push-vs-pull)
3. [Flux CD on AKS](#3-flux)
4. [ArgoCD on AKS](#4-argocd)
5. [Repository Structure](#5-repo-structure)
6. [Multi-Environment Management](#6-multi-env)
7. [Secrets Management in GitOps](#7-secrets)
8. [Image Automation](#8-image-automation)
9. [Monitoring & Troubleshooting](#9-monitoring)
10. [Best Practices](#10-best-practices)

---

## 1. What is GitOps? {#1-what-is-gitops}

GitOps is a set of practices that uses **Git as the single source of truth** for declarative infrastructure and application configuration. A GitOps operator runs inside the cluster and continuously reconciles the actual state with the desired state defined in Git.

```
Traditional CI/CD (Push):           GitOps (Pull):
─────────────────────               ───────────────

Developer                           Developer
    │                                    │
    ├── git push                         ├── git push (manifests)
    │                                    │
    ▼                                    ▼
CI Pipeline                          Git Repository
    │                                    │
    ├── build image                      │ (source of truth)
    ├── kubectl apply ──► Cluster        │
    │  (push from outside)               │
                                         ▼
                                    GitOps Operator
                                    (Flux / ArgoCD)
                                         │
                                    Continuously pulls
                                    from Git and applies
                                         │
                                         ▼
                                      K8s Cluster
                                    (self-healing)
```

### GitOps Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | Entire system described declaratively (YAML/Helm/Kustomize) |
| **Versioned** | Desired state stored in Git with full history |
| **Automated** | Approved changes auto-applied to the system |
| **Self-healing** | Drift detection + automatic reconciliation |

### Benefits

```
✅ Audit trail      — Every change is a git commit with author, message, timestamp
✅ Rollback          — git revert to restore previous state
✅ Security          — No kubectl access needed for deployment; CI doesn't need cluster credentials
✅ Consistency       — Drift detection ensures cluster matches Git
✅ Multi-cluster     — Same Git repo can deploy to dev/staging/prod clusters
```

---

## 2. Push-Based vs Pull-Based Deployment {#2-push-vs-pull}

| Aspect | Push (kubectl apply in CI) | Pull (GitOps) |
|--------|---------------------------|---------------|
| Who applies manifests? | CI pipeline | In-cluster operator |
| Cluster credentials needed in CI? | ✅ Yes (kubeconfig) | ❌ No |
| Drift detection? | ❌ None | ✅ Continuous |
| Self-healing? | ❌ No | ✅ Auto-reconcile |
| Security surface | CI has cluster admin | Git has write access only |
| Rollback mechanism | Re-run pipeline | git revert |
| Audit trail | Pipeline logs | Git history |

---

## 3. Flux CD on AKS {#3-flux}

Flux is a CNCF graduated project, natively integrated with AKS via the **GitOps extension**.

### Install Flux on AKS (Azure-managed)

```bash
# Register providers
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.KubernetesConfiguration

# Enable GitOps extension on AKS cluster
az k8s-configuration flux create \
  --resource-group myRG \
  --cluster-name myAKS \
  --cluster-type managedClusters \
  --name my-gitops-config \
  --namespace flux-system \
  --scope cluster \
  --url https://github.com/myorg/k8s-config \
  --branch main \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./apps/production prune=true dependsOn=infra
```

### Install Flux Standalone (CLI)

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux into cluster (creates repo structure + installs controllers)
flux bootstrap github \
  --owner=myorg \
  --repository=k8s-config \
  --branch=main \
  --path=clusters/production \
  --personal

# Check Flux components
flux check

# View Flux sources and kustomizations
flux get sources git
flux get kustomizations
```

### Flux Architecture

```
┌──────────── AKS Cluster ──────────────────────────────────────┐
│                                                                │
│  flux-system namespace:                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Source Controller│  │ Kustomize Ctrl  │  │ Helm Ctrl    │ │
│  │ (fetches from   │  │ (applies K8s    │  │ (installs    │ │
│  │  Git/Helm repos)│  │  manifests)     │  │  Helm charts)│ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘ │
│           │                    │                    │          │
│           └────────────────────┼────────────────────┘          │
│                                │                               │
│                                ▼                               │
│                          Apply manifests                       │
│                          to cluster                            │
└───────────────────────────────┬────────────────────────────────┘
                                │
                        Polls every 1-5 min
                                │
                                ▼
                        ┌──────────────────┐
                        │  Git Repository  │
                        │  (source of truth)│
                        └──────────────────┘
```

### Flux GitRepository + Kustomization

```yaml
# flux-system/sources/git-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-config
  namespace: flux-system
spec:
  interval: 5m               # Poll every 5 minutes
  url: https://github.com/myorg/k8s-config
  ref:
    branch: main
  secretRef:
    name: git-credentials     # For private repos
---
# flux-system/kustomizations/apps.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: app-config
  path: ./apps/production     # Path in the Git repo
  prune: true                 # Delete resources removed from Git
  healthChecks:               # Wait for resources to be healthy
    - apiVersion: apps/v1
      kind: Deployment
      name: api
      namespace: default
  timeout: 5m
```

### Flux HelmRelease

```yaml
# Deploy Helm chart via Flux
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: default
spec:
  interval: 30m
  chart:
    spec:
      chart: redis
      version: "18.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    architecture: standalone
    auth:
      enabled: true
      password: "${REDIS_PASSWORD}"
    master:
      persistence:
        size: 8Gi
```

---

## 4. ArgoCD on AKS {#4-argocd}

ArgoCD is a declarative GitOps tool with a powerful web UI for visualizing deployments.

### Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD UI (for development)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login localhost:8080 --username admin --password <password> --insecure
```

### ArgoCD Application

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-api
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/k8s-config.git
    targetRevision: main
    path: apps/production/api
    
    # For Helm charts:
    # helm:
    #   valueFiles:
    #     - values-production.yaml
  
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  
  syncPolicy:
    automated:
      prune: true              # Delete resources removed from Git
      selfHeal: true           # Auto-fix drift
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ArgoCD ApplicationSet (Multi-Cluster)

```yaml
# Deploy same app to multiple clusters/environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-api
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster-url
            path: apps/dev
          - cluster: staging
            url: https://staging-cluster-url
            path: apps/staging
          - cluster: production
            url: https://prod-cluster-url
            path: apps/production
  template:
    metadata:
      name: "my-api-{{cluster}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-config.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: "{{url}}"
        namespace: default
```

---

## 5. Repository Structure {#5-repo-structure}

### Mono-Repo (Recommended for Small Teams)

```
k8s-config/
├── clusters/                  # Per-cluster Flux bootstrap
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
│
├── infrastructure/            # Shared infra (ingress, cert-manager, monitoring)
│   ├── sources/               # Helm repositories
│   ├── ingress-nginx/
│   ├── cert-manager/
│   └── monitoring/
│
├── apps/                      # Application manifests
│   ├── base/                  # Base manifests (shared)
│   │   ├── api/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── web/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── kustomization.yaml
│   │
│   ├── dev/                   # Dev overlays
│   │   ├── api/
│   │   │   ├── kustomization.yaml   # patches for dev
│   │   │   └── patch-replicas.yaml
│   │   └── kustomization.yaml
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── production/            # Production overlays
│       ├── api/
│       │   ├── kustomization.yaml
│       │   ├── patch-replicas.yaml
│       │   └── patch-resources.yaml
│       └── kustomization.yaml
```

### Kustomize Overlay Example

```yaml
# apps/base/api/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

# apps/production/api/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/api
patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
images:
  - name: myacr.azurecr.io/api
    newTag: v1.5.2                # Pin production image tag

# apps/production/api/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
```

---

## 6. Multi-Environment Management {#6-multi-env}

### Promotion Workflow

```
Developer merges PR to main
     │
     ▼
Flux/ArgoCD auto-deploys to DEV
     │
     ├── Tests pass in dev
     │
     ▼
Developer opens PR: update staging overlay image tag
     │
     ├── Reviewer approves → merge
     │
     ▼
Flux/ArgoCD auto-deploys to STAGING
     │
     ├── QA validates in staging
     │
     ▼
Developer opens PR: update production overlay image tag
     │
     ├── Senior reviewer + security review → merge
     │
     ▼
Flux/ArgoCD auto-deploys to PRODUCTION
```

---

## 7. Secrets Management in GitOps {#7-secrets}

Git repos should **never** contain plaintext secrets. Options:

### SOPS (Secrets OPerationS) + Azure Key Vault

```bash
# Encrypt a secret with SOPS + Azure Key Vault
sops --encrypt \
  --azure-kv https://myvault.vault.azure.net/keys/sops-key/version \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml

# Flux decrypts automatically with SOPS provider
```

```yaml
# Flux Kustomization with SOPS decryption
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-azure     # Contains Azure Key Vault credentials
  path: ./apps/production
  sourceRef:
    kind: GitRepository
    name: app-config
```

### External Secrets Operator + Azure Key Vault

```yaml
# External Secrets Operator pulls secrets from Key Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: api-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-connection
      remoteRef:
        key: database-connection-string
    - secretKey: api-key
      remoteRef:
        key: external-api-key
```

---

## 8. Image Automation {#8-image-automation}

Flux can **automatically update image tags** in Git when new images are pushed to a registry.

```yaml
# Watch for new images in ACR
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: api
  namespace: flux-system
spec:
  image: myacr.azurecr.io/api
  interval: 5m
  provider: azure
---
# Auto-update policy (semver)
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: api
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: api
  policy:
    semver:
      range: "1.x.x"        # Auto-update within 1.x.x
---
# Auto-commit image tag updates to Git
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: auto-update
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: app-config
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: flux-bot
        email: flux@myorg.com
      messageTemplate: "chore: update image {{.ImageName}} to {{.NewTag}}"
    push:
      branch: main
  update:
    path: ./apps
    strategy: Setters
```

---

## 9. Monitoring & Troubleshooting {#9-monitoring}

### Flux

```bash
# Check overall status
flux check

# List all Flux resources
flux get all

# Debug a specific kustomization
flux get kustomization apps -o wide

# View Flux controller logs
flux logs --kind=Kustomization --name=apps

# Force reconciliation (don't wait for interval)
flux reconcile kustomization apps --with-source

# Suspend/resume (pause deployments)
flux suspend kustomization apps
flux resume kustomization apps
```

### ArgoCD

```bash
# Check app status
argocd app get my-api

# Sync manually
argocd app sync my-api

# View sync history
argocd app history my-api

# Rollback to previous version
argocd app rollback my-api <history-id>

# View diff (what would change)
argocd app diff my-api
```

---

## 10. Best Practices {#10-best-practices}

```
✅ Repository Strategy
├─ Separate app source code repos from K8s config repos
├─ Use Kustomize overlays for environment differences
├─ Pin image tags (never use :latest in GitOps)
├─ Require PR reviews for production overlay changes
└─ Use branch protection rules on the config repo

✅ Security
├─ Never store plaintext secrets in Git (use SOPS or External Secrets)
├─ Use deploy keys with read-only access for Flux/ArgoCD
├─ RBAC: Limit who can merge to production paths
├─ Use Workload Identity for Flux to access ACR/Key Vault
└─ Audit all git commits to the config repo

✅ Operations
├─ Start with Flux if using AKS (native Azure integration)
├─ Use ArgoCD if you need a web UI for visibility
├─ Set reconciliation intervals: 5min (dev), 10min (prod)
├─ Enable pruning to clean up removed resources
├─ Monitor reconciliation failures with alerts
└─ Test changes in dev → staging → production (promotion model)

✅ Flux vs ArgoCD Decision
├─ Flux: Lightweight, CLI-first, native AKS integration, CNCF graduated
├─ ArgoCD: Rich web UI, multi-cluster dashboard, app-of-apps pattern
├─ Both: Can be used together (Flux for infra, ArgoCD for apps)
└─ Small teams / AKS-only → Flux | Large teams / multi-cluster → ArgoCD
```
