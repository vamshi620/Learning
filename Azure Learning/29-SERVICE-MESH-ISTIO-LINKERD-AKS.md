# Service Mesh on AKS — Istio & Linkerd Guide
## Document 29: Traffic Management, mTLS, and Observability

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Service mesh concepts, Istio on AKS, Linkerd comparison, traffic management, mutual TLS, observability

---

## TABLE OF CONTENTS

1. [What is a Service Mesh?](#1-overview)
2. [When Do You Need a Service Mesh?](#2-when)
3. [Istio vs Linkerd vs AKS Service Mesh](#3-comparison)
4. [Istio on AKS](#4-istio)
5. [Traffic Management](#5-traffic)
6. [Security (mTLS)](#6-mtls)
7. [Observability](#7-observability)
8. [Linkerd on AKS](#8-linkerd)
9. [AKS Istio Add-on (Managed)](#9-managed)
10. [Best Practices](#10-best-practices)

---

## 1. What is a Service Mesh? {#1-overview}

A service mesh is a **dedicated infrastructure layer** for managing service-to-service communication in a microservices architecture. It adds capabilities like traffic management, security, and observability **without changing application code**.

```
Without Service Mesh:                   With Service Mesh:

Service A ──── HTTP ──── Service B      Service A ──► [Proxy] ═══ [Proxy] ──► Service B
  │                        │              │          (sidecar)    (sidecar)        │
  │ Manual retry logic     │              │  ✅ Auto-retry                         │
  │ Manual circuit breaker │              │  ✅ Circuit breaking                   │
  │ No mTLS               │              │  ✅ mTLS encryption                    │
  │ No traffic control    │              │  ✅ Canary routing                     │
  │ No distributed tracing│              │  ✅ Distributed tracing                │
  │ No rate limiting      │              │  ✅ Rate limiting                      │

Each service implements its own         Mesh handles all cross-cutting
cross-cutting concerns                  concerns transparently
```

### Service Mesh Architecture

```
┌────────────────── Control Plane ──────────────────┐
│  (Istiod / linkerd-destination / etc.)             │
│                                                     │
│  ├── Configuration (routing rules, policies)       │
│  ├── Certificate Authority (mTLS certs)            │
│  ├── Service Discovery                             │
│  └── Telemetry Collection                          │
└─────────────────────┬───────────────────────────────┘
                      │ pushes config
                      ▼
┌──────────────── Data Plane ───────────────────────┐
│                                                     │
│  Pod A                    Pod B                     │
│  ┌──────┬──────────┐     ┌──────┬──────────┐      │
│  │ App  │ Envoy    │═════│ Envoy│ App      │      │
│  │      │ Sidecar  │mTLS │Sidecar│         │      │
│  └──────┴──────────┘     └──────┴──────────┘      │
│                                                     │
│  All traffic flows through sidecar proxies          │
└─────────────────────────────────────────────────────┘
```

### Core Capabilities

| Capability | Description |
|-----------|-------------|
| **Traffic Management** | Routing, load balancing, canary, A/B testing, fault injection |
| **Security** | mTLS (mutual TLS), authorization policies, certificate rotation |
| **Observability** | Metrics, distributed tracing, access logging (no code changes) |
| **Resilience** | Retries, timeouts, circuit breaking, rate limiting |

---

## 2. When Do You Need a Service Mesh? {#2-when}

```
✅ You likely NEED a service mesh if:
├─ You have 10+ microservices communicating with each other
├─ You need zero-trust security (mTLS between all services)
├─ You need advanced traffic management (canary, A/B, fault injection)
├─ You need service-level observability without code changes
├─ You have compliance requirements for encryption in transit
└─ You're doing multi-cluster or multi-cloud deployments

❌ You probably DON'T need a service mesh if:
├─ You have fewer than 5 services
├─ Services communicate primarily through message queues
├─ Your team is new to Kubernetes
├─ Performance overhead is a critical concern (adds ~2-5ms latency)
└─ You can achieve your goals with simpler tools (Ingress + NetworkPolicy)

🔄 Alternatives to consider first:
├─ Dapr (built into Container Apps, available for AKS)
├─ Azure Front Door + Application Gateway for north-south traffic
├─ Kubernetes NetworkPolicy for basic east-west security
└─ OpenTelemetry SDK for application-level observability
```

---

## 3. Istio vs Linkerd vs AKS Managed {#3-comparison}

| Feature | Istio | Linkerd | AKS Istio Add-on |
|---------|-------|---------|-------------------|
| **Complexity** | High | Low | Medium (managed) |
| **Proxy** | Envoy (heavy) | linkerd2-proxy (Rust, lightweight) | Envoy (managed) |
| **Latency overhead** | ~5-10ms p99 | ~1-2ms p99 | ~5-10ms p99 |
| **Memory per sidecar** | ~50-100MB | ~10-20MB | ~50-100MB |
| **mTLS** | ✅ | ✅ | ✅ |
| **Traffic management** | ✅ (very rich) | ✅ (basic) | ✅ (rich) |
| **Multi-cluster** | ✅ | ✅ | ✅ |
| **Web dashboard** | Kiali | Linkerd Viz | Azure Portal |
| **WASM extensions** | ✅ | ❌ | ✅ |
| **Community** | Huge (CNCF graduated) | Active (CNCF graduated) | Azure-managed |
| **Best for** | Full-featured mesh | Simplicity, low overhead | AKS-native, supported |

---

## 4. Istio on AKS {#4-istio}

### Install Istio (Manual)

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with demo profile (includes all features)
istioctl install --set profile=demo -y

# For production, use the "default" profile
# istioctl install --set profile=default -y

# Enable sidecar injection for a namespace
kubectl label namespace default istio-injection=enabled

# Verify installation
istioctl verify-install
kubectl get pods -n istio-system
```

### Istio Profiles

| Profile | Components | Use Case |
|---------|-----------|----------|
| `minimal` | Istiod only | Custom setup |
| `default` | Istiod + Ingress Gateway | Production |
| `demo` | All components including egress gateway | Learning/testing |
| `ambient` | Ambient mode (no sidecars, uses ztunnel) | Lightweight, new |

---

## 5. Traffic Management {#5-traffic}

### VirtualService (Routing Rules)

```yaml
# Route traffic between versions (canary deployment)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
    - api                      # Kubernetes service name
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"    # Header-based routing
      route:
        - destination:
            host: api
            subset: v2
    - route:                   # Default route
        - destination:
            host: api
            subset: v1
          weight: 90           # 90% to v1
        - destination:
            host: api
            subset: v2
          weight: 10           # 10% to v2 (canary)
```

### DestinationRule (Subsets & Load Balancing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:            # Circuit breaker
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### Fault Injection (Chaos Testing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
    - api
  http:
    - fault:
        delay:
          percentage:
            value: 10          # 10% of requests
          fixedDelay: 5s       # 5 second delay
        abort:
          percentage:
            value: 5           # 5% of requests
          httpStatus: 500      # Return 500 error
      route:
        - destination:
            host: api
```

### Timeout & Retry

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
    - api
  http:
    - timeout: 10s             # Request timeout
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"
      route:
        - destination:
            host: api
```

---

## 6. Security (mTLS) {#6-mtls}

### Enable Strict mTLS (Mesh-Wide)

```yaml
# All traffic between services MUST use mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system       # Mesh-wide
spec:
  mtls:
    mode: STRICT                # STRICT, PERMISSIVE, or DISABLE
```

### Authorization Policy

```yaml
# Only allow specific services to call the API
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
  namespace: default
spec:
  selector:
    matchLabels:
      app: api
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/default/sa/web-service"
              - "cluster.local/ns/default/sa/worker-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

### Deny All by Default

```yaml
# Default deny all traffic (zero-trust)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}  # Empty spec = deny all
```

---

## 7. Observability {#7-observability}

### Metrics (Prometheus + Grafana)

```bash
# Install Kiali (service mesh dashboard)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/kiali.yaml

# Install Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/prometheus.yaml

# Install Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/grafana.yaml

# Access Kiali dashboard
istioctl dashboard kiali

# Access Grafana
istioctl dashboard grafana
```

### Key Metrics (Auto-Collected by Istio)

```
istio_requests_total            — Total request count
istio_request_duration_seconds  — Request latency histogram
istio_request_bytes             — Request body sizes
istio_response_bytes            — Response body sizes
istio_tcp_connections_opened    — TCP connections

Labels: source_app, destination_app, response_code, request_protocol
```

### Distributed Tracing (Jaeger)

```bash
# Install Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/jaeger.yaml

# Access Jaeger dashboard
istioctl dashboard jaeger
```

```
Distributed Trace Example:

[Web Service] ──200ms──► [API Service] ──50ms──► [Database Service]
     │                        │                        │
     │ Total: 280ms           │ Total: 80ms            │ Total: 30ms
     │ Self: 200ms            │ Self: 50ms             │ Self: 30ms
     │                        │                        │
     └────── Trace ID: abc-123 ────────────────────────┘

Istio automatically propagates trace headers (B3, W3C TraceContext)
Your app just needs to forward the trace headers to downstream calls
```

---

## 8. Linkerd on AKS {#8-linkerd}

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# Pre-flight checks
linkerd check --pre

# Install Linkerd CRDs + control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify installation
linkerd check

# Add mesh to a namespace (inject sidecars)
kubectl annotate namespace default linkerd.io/inject=enabled

# Restart existing pods to inject sidecar
kubectl rollout restart deployment -n default

# Install Linkerd Viz (dashboard + metrics)
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

### Linkerd Traffic Split (Canary)

```yaml
# Uses SMI TrafficSplit API
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: api-canary
  namespace: default
spec:
  service: api                  # Root service
  backends:
    - service: api-v1           # Stable version
      weight: 900               # 90%
    - service: api-v2           # Canary version
      weight: 100               # 10%
```

---

## 9. AKS Istio Add-on (Managed) {#9-managed}

Azure offers a **managed Istio add-on** for AKS that handles installation, upgrades, and configuration.

```bash
# Enable Istio add-on on existing AKS cluster
az aks mesh enable \
  --resource-group myRG \
  --name myAKS

# Create new AKS cluster with Istio
az aks create \
  --resource-group myRG \
  --name myAKS \
  --enable-asm           # Azure Service Mesh (Istio)

# Enable sidecar injection for a namespace
kubectl label namespace default istio.io/rev=asm-1-22

# Enable external ingress gateway
az aks mesh enable-ingress-gateway \
  --resource-group myRG \
  --name myAKS \
  --ingress-gateway-type external

# Check mesh status
az aks show \
  --resource-group myRG \
  --name myAKS \
  --query "serviceMeshProfile"

# Upgrade Istio version
az aks mesh upgrade start \
  --resource-group myRG \
  --name myAKS \
  --revision asm-1-23
```

---

## 10. Best Practices {#10-best-practices}

```
✅ Getting Started
├─ Start with mTLS only (biggest security win, least complexity)
├─ Add observability next (Kiali/Viz dashboards)
├─ Add traffic management last (canary, fault injection)
├─ Use PERMISSIVE mTLS mode during migration, then switch to STRICT
└─ Start with one namespace, expand gradually

✅ Performance
├─ Right-size sidecar proxy resources (CPU/memory limits)
├─ Use Linkerd if latency overhead is critical (~1-2ms vs ~5-10ms)
├─ Exclude high-throughput internal services if mesh overhead is unacceptable
├─ Monitor proxy memory usage — set appropriate limits
└─ Consider Istio Ambient mode (no sidecars) for reduced overhead

✅ Security
├─ Enable STRICT mTLS mesh-wide for zero-trust
├─ Use AuthorizationPolicy to enforce least-privilege
├─ Rotate mTLS certificates automatically (default: 24 hours in Istio)
├─ Deny all traffic by default, allow explicitly
└─ Audit authorization policies regularly

✅ Operations
├─ Use AKS Istio add-on for Azure-managed upgrades and support
├─ Pin Istio version to avoid unexpected upgrades
├─ Monitor control plane health (Istiod CPU/memory)
├─ Use canary upgrades for Istio itself (revision-based)
├─ Keep mesh configuration in Git (GitOps)
└─ Test traffic policies in dev before production

✅ Decision Framework
├─ < 5 services → Skip service mesh; use NetworkPolicy + Dapr
├─ 5-20 services → Linkerd (simple) or AKS Istio add-on (managed)
├─ 20+ services → Istio (full features) or AKS Istio add-on
└─ Multi-cluster → Istio (built-in multi-cluster support)
```
