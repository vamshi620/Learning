# Azure API Management (APIM) — Complete Guide
## Document 24: API Gateway, Policies, Security & Developer Portal

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** APIM concepts, policies, OAuth integration, rate limiting, versioning, developer portal

---

## TABLE OF CONTENTS

1. [What is API Management?](#1-what-is-api-management)
2. [Architecture & Tiers](#2-architecture--tiers)
3. [Creating APIM](#3-creating-apim)
4. [APIs, Operations & Products](#4-apis-operations--products)
5. [Policies — The Core of APIM](#5-policies)
6. [Authentication & Authorization](#6-authentication)
7. [Rate Limiting & Throttling](#7-rate-limiting)
8. [API Versioning & Revisions](#8-versioning)
9. [Developer Portal](#9-developer-portal)
10. [Monitoring & Analytics](#10-monitoring)
11. [Best Practices](#11-best-practices)

---

## 1. What is API Management?

Azure API Management is a **fully managed API gateway** that sits between API consumers and your backend services. It provides:

```
External Clients          APIM Gateway              Backend APIs
─────────────────         ────────────              ────────────
Mobile App        ──►  ┌──────────────────┐  ──►  App Service
Web SPA           ──►  │  API Management  │  ──►  Azure Functions
Partner API       ──►  │                  │  ──►  AKS Microservices
IoT Devices       ──►  │  • Auth (OAuth)  │  ──►  Logic Apps
                       │  • Rate Limiting │  ──►  On-prem APIs
                       │  • Caching       │
                       │  • Transform     │
                       │  • Analytics     │
                       └──────────────────┘
                              │
                       Developer Portal
                       (Self-service docs,
                        API keys, testing)
```

### Why Use APIM?

| Without APIM | With APIM |
|--------------|-----------|
| Each API implements its own auth | Centralized auth at gateway |
| No rate limiting | Configurable rate limiting per product/subscription |
| No unified docs | Auto-generated developer portal |
| No analytics | Built-in analytics and Application Insights |
| Direct backend exposure | Backend protection behind gateway |
| Manual API versioning | Built-in versioning and revisions |

---

## 2. Architecture & Tiers

### APIM Components

```
┌───────────────────────────────────────────────────────┐
│                 API Management Instance                │
│                                                       │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────┐│
│  │   Gateway    │  │ Azure    │  │  Developer       ││
│  │   (runtime)  │  │ Portal   │  │  Portal          ││
│  │             │  │ (config) │  │  (self-service)  ││
│  │  Routes,    │  │          │  │                  ││
│  │  policies,  │  │  API     │  │  API docs,       ││
│  │  throttles  │  │  design  │  │  try-it,         ││
│  │             │  │          │  │  subscribe       ││
│  └─────────────┘  └──────────┘  └──────────────────┘│
└───────────────────────────────────────────────────────┘
```

### Pricing Tiers

| Tier | Use Case | SLA | VNet | Gateway Regions | Price |
|------|----------|-----|------|-----------------|-------|
| **Consumption** | Serverless, pay-per-call | 99.95% | ❌ | 1 | ~$3.50/million calls |
| **Developer** | Dev/test, non-production | No SLA | ✅ | 1 | ~$50/month |
| **Basic** | Small production | 99.95% | ❌ | 1 | ~$150/month |
| **Standard** | Medium production | 99.95% | ❌ | 1+ | ~$700/month |
| **Premium** | Enterprise, multi-region | 99.99% | ✅ | Multiple | ~$2,800/month/unit |

---

## 3. Creating APIM

```bash
# Create APIM instance (Standard tier)
az apim create \
  --name myapim \
  --resource-group myRG \
  --location eastus \
  --publisher-name "Contoso" \
  --publisher-email "api@contoso.com" \
  --sku-name Standard \
  --sku-capacity 1

# Note: APIM provisioning takes 30-45 minutes for non-Consumption tiers

# Create Consumption tier (instant provisioning)
az apim create \
  --name myapim-consumption \
  --resource-group myRG \
  --location eastus \
  --publisher-name "Contoso" \
  --publisher-email "api@contoso.com" \
  --sku-name Consumption
```

---

## 4. APIs, Operations & Products

### Import an API

```bash
# Import from OpenAPI (Swagger) specification
az apim api import \
  --resource-group myRG \
  --service-name myapim \
  --api-id orders-api \
  --path orders \
  --specification-format OpenApi \
  --specification-url "https://mybackend.azurewebsites.net/swagger/v1/swagger.json" \
  --display-name "Orders API" \
  --protocols https \
  --service-url "https://mybackend.azurewebsites.net"

# Import from Azure Function App
az apim api import \
  --resource-group myRG \
  --service-name myapim \
  --api-id functions-api \
  --path functions \
  --specification-format OpenApi \
  --specification-url "https://myfuncapp.azurewebsites.net/api/swagger.json"
```

### Products

Products **group APIs** and apply access policies. Consumers subscribe to Products, not individual APIs.

```bash
# Create a product
az apim product create \
  --resource-group myRG \
  --service-name myapim \
  --product-id starter \
  --title "Starter" \
  --description "Rate-limited access for trial users" \
  --state published \
  --subscription-required true \
  --approval-required false \
  --subscriptions-limit 1

# Add API to product
az apim product api add \
  --resource-group myRG \
  --service-name myapim \
  --product-id starter \
  --api-id orders-api
```

```
Products Structure:
├── Starter (Free tier)
│   ├── Orders API
│   ├── Rate limit: 10 calls/minute
│   └── Auto-approval
│
├── Professional ($49/month)
│   ├── Orders API
│   ├── Payments API
│   ├── Rate limit: 1000 calls/minute
│   └── Manual approval
│
└── Enterprise (Custom pricing)
    ├── All APIs
    ├── No rate limit
    └── Manual approval + NDA
```

---

## 5. Policies — The Core of APIM {#5-policies}

Policies are XML statements that modify API behavior at the gateway. They execute at four scopes:

```
Request Flow:                          Response Flow:
                                       
Client ──► [inbound] ──► [backend] ──► [outbound] ──► Client
                │                          │
           [on-error]  ◄──────────────────┘
                                    (if error)
```

### Common Policies

#### Rate Limiting

```xml
<policies>
  <inbound>
    <!-- 100 calls per 60 seconds per subscription key -->
    <rate-limit calls="100" renewal-period="60" />
    
    <!-- OR quota: 10,000 calls per 7 days -->
    <quota calls="10000" renewal-period="604800" />
    
    <!-- Rate limit by custom key (e.g., IP address) -->
    <rate-limit-by-key calls="50" renewal-period="60"
                       counter-key="@(context.Request.IpAddress)" />
  </inbound>
</policies>
```

#### CORS

```xml
<policies>
  <inbound>
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://myapp.contoso.com</origin>
        <origin>https://portal.contoso.com</origin>
      </allowed-origins>
      <allowed-methods>
        <method>GET</method>
        <method>POST</method>
        <method>PUT</method>
        <method>DELETE</method>
      </allowed-methods>
      <allowed-headers>
        <header>Content-Type</header>
        <header>Authorization</header>
      </allowed-headers>
    </cors>
  </inbound>
</policies>
```

#### Caching

```xml
<policies>
  <inbound>
    <!-- Cache responses for 300 seconds -->
    <cache-lookup vary-by-developer="false"
                  vary-by-developer-groups="false"
                  caching-type="internal">
      <vary-by-query-parameter>page</vary-by-query-parameter>
      <vary-by-query-parameter>pageSize</vary-by-query-parameter>
    </cache-lookup>
  </inbound>
  <outbound>
    <cache-store duration="300" />
  </outbound>
</policies>
```

#### JWT Validation (OAuth 2.0)

```xml
<policies>
  <inbound>
    <validate-jwt header-name="Authorization" 
                  failed-validation-httpcode="401"
                  failed-validation-error-message="Unauthorized">
      <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud" match="all">
          <value>{api-client-id}</value>
        </claim>
        <claim name="roles" match="any">
          <value>API.Read</value>
          <value>API.ReadWrite</value>
        </claim>
      </required-claims>
    </validate-jwt>
  </inbound>
</policies>
```

#### Request/Response Transformation

```xml
<policies>
  <inbound>
    <!-- Add header to backend request -->
    <set-header name="X-Request-Source" exists-action="override">
      <value>APIM-Gateway</value>
    </set-header>
    
    <!-- Rewrite URL -->
    <rewrite-uri template="/api/v2/{path}" />
  </inbound>
  
  <outbound>
    <!-- Remove internal headers from response -->
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="X-AspNet-Version" exists-action="delete" />
    
    <!-- Transform JSON response -->
    <set-body>@{
      var response = context.Response.Body.As<JObject>();
      response.Add("gateway", "apim");
      return response.ToString();
    }</set-body>
  </outbound>
</policies>
```

#### Mock Responses

```xml
<policies>
  <inbound>
    <!-- Return mock response without hitting backend -->
    <mock-response status-code="200" content-type="application/json" />
  </inbound>
</policies>
```

#### IP Filtering

```xml
<policies>
  <inbound>
    <ip-filter action="allow">
      <address-range from="203.0.113.0" to="203.0.113.255" />
      <address>198.51.100.4</address>
    </ip-filter>
  </inbound>
</policies>
```

---

## 6. Authentication & Authorization {#6-authentication}

### OAuth 2.0 with Entra ID

```
Client App                    APIM                     Backend API
    │                          │                           │
    │ 1. Get token from        │                           │
    │    Entra ID ─────────►   │                           │
    │ ◄── Access Token         │                           │
    │                          │                           │
    │ 2. Call API with         │                           │
    │    Bearer token ────────►│                           │
    │                          │ 3. validate-jwt           │
    │                          │    (verify token)         │
    │                          │                           │
    │                          │ 4. Forward to backend ───►│
    │                          │    (with/without token)   │
    │                          │ ◄── Response ─────────────│
    │ ◄────── Response ────────│                           │
```

### Subscription Keys

```bash
# Create a subscription
az apim subscription create \
  --resource-group myRG \
  --service-name myapim \
  --subscription-id my-sub \
  --display-name "Partner App Subscription" \
  --scope "/products/professional" \
  --state active

# Call API with subscription key
curl https://myapim.azure-api.net/orders \
  -H "Ocp-Apim-Subscription-Key: <subscription-key>"
```

---

## 7. Rate Limiting & Throttling {#7-rate-limiting}

| Policy | Scope | Resets | Use Case |
|--------|-------|--------|----------|
| `rate-limit` | Per subscription | Rolling window | Burst protection |
| `rate-limit-by-key` | Custom key (IP, user, etc.) | Rolling window | Per-user/IP throttling |
| `quota` | Per subscription | Fixed window | Usage-based billing |
| `quota-by-key` | Custom key | Fixed window | Per-customer quota |

---

## 8. API Versioning & Revisions {#8-versioning}

### Versioning Schemes

```
URL Path:     https://api.contoso.com/v1/orders
              https://api.contoso.com/v2/orders

Query String: https://api.contoso.com/orders?api-version=2024-01-01

Header:       GET /orders  HTTP/1.1
              Api-Version: 2024-01-01
```

### Revisions (Non-Breaking Changes)

```
API: Orders API
├── Revision 1 (current) ← https://api.contoso.com/orders
├── Revision 2 (testing) ← https://api.contoso.com/orders;rev=2
└── Revision 3 (draft)   ← not accessible externally
```

---

## 9. Developer Portal {#9-developer-portal}

The Developer Portal is an **auto-generated, customizable website** where API consumers can:

- Browse available APIs and read documentation
- Try APIs interactively (built-in test console)
- Subscribe to products and get API keys
- View usage analytics for their subscriptions
- Read getting-started guides and tutorials

```bash
# Enable developer portal
# Go to Azure Portal → APIM → Developer Portal → Enable

# Customize via:
# 1. Visual editor (drag-and-drop in browser)
# 2. Self-hosted developer portal (GitHub repo for full customization)
```

---

## 10. Monitoring & Analytics {#10-monitoring}

```bash
# Enable Application Insights integration
az apim update \
  --name myapim \
  --resource-group myRG \
  --set properties.customProperties.'Microsoft.WindowsAzure.ApiManagement.Gateway.Protocols.Server.Http2'='true'

# Create diagnostic logger
az apim diagnostic create \
  --resource-group myRG \
  --service-name myapim \
  --api-id orders-api \
  --diagnostic-id applicationinsights \
  --logger-id my-app-insights-logger \
  --sampling-percentage 100
```

---

## 11. Best Practices {#11-best-practices}

```
✅ Architecture
├─ Use APIM as the single entry point for all APIs
├─ Group APIs into Products for access control
├─ Use named values for configuration (not hardcoded in policies)
├─ Use policy fragments for reusable policy snippets
└─ Enable Application Insights for full request tracing

✅ Security
├─ Always validate JWT tokens at the gateway
├─ Remove internal headers in outbound policies
├─ Use IP filtering + subscription keys for partner APIs
├─ Enable HTTPS only (disable HTTP)
└─ Use Managed Identity for backend authentication

✅ Performance
├─ Cache GET responses with cache-lookup/cache-store
├─ Use Consumption tier for low-traffic APIs (cost savings)
├─ Use Premium tier with multiple gateway regions for global APIs
└─ Set appropriate rate limits to protect backends

✅ Operations
├─ Use revisions for non-breaking changes
├─ Use versions for breaking changes
├─ Export OpenAPI specs to source control
├─ Automate APIM deployment with Bicep/Terraform
└─ Monitor P95 latency and 4xx/5xx rates in dashboards
```
