# Entra External ID (B2C) — Customer Identity Guide
## Document 27: Customer-Facing Authentication & Social Logins

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Entra External ID, social identity providers, custom flows, token customization, B2C migration

---

## TABLE OF CONTENTS

1. [What is Entra External ID?](#1-overview)
2. [B2C vs Entra ID vs External ID](#2-comparison)
3. [Architecture & Concepts](#3-architecture)
4. [Setting Up External ID Tenant](#4-setup)
5. [User Flows (Built-in)](#5-user-flows)
6. [Custom Authentication Extensions](#6-custom-extensions)
7. [Social Identity Providers](#7-social-idps)
8. [Token Customization](#8-tokens)
9. [Branding & UI Customization](#9-branding)
10. [Security Best Practices](#10-security)
11. [Integration with .NET](#11-dotnet)
12. [Migration from B2C](#12-migration)

---

## 1. What is Entra External ID? {#1-overview}

Microsoft Entra External ID is a **customer identity and access management (CIAM)** platform. It handles authentication for your **external users** — customers, partners, and consumers — as opposed to Entra ID which handles employees (internal workforce).

```
Entra ID (Workforce)                    Entra External ID (Customer)
──────────────────                      ─────────────────────────
✅ Employees                            ✅ Customers / Consumers
✅ Corporate SSO                        ✅ Self-service sign-up
✅ Internal apps                        ✅ Customer-facing apps
✅ Office 365 / Teams                   ✅ Social logins (Google, Apple, Facebook)
✅ Conditional Access                   ✅ Custom branding
✅ Device management                    ✅ Progressive profiling
                                        ✅ Age gating / Terms of Use
```

---

## 2. B2C vs Entra ID vs External ID {#2-comparison}

| Feature | Entra ID | Azure AD B2C (Legacy) | Entra External ID |
|---------|----------|----------------------|-------------------|
| **Audience** | Employees | Customers | Customers |
| **Status** | Active | Maintenance mode | **New (recommended)** |
| **Tenant type** | Workforce | Separate B2C tenant | External tenant |
| **Social logins** | B2B guest only | ✅ | ✅ |
| **Self-service sign-up** | Limited | ✅ | ✅ |
| **Custom policies** | ❌ | ✅ (XML-based IEF) | ✅ (Auth extensions) |
| **Branding** | Company branding | Full custom HTML/CSS | Full custom |
| **MSAL SDK support** | ✅ | ✅ | ✅ |
| **Pricing** | Per user/month | Per authentication | Per MAU (50K free) |

> **Recommendation:** For new projects, use **Entra External ID**. Azure AD B2C is legacy and in maintenance mode. Existing B2C apps continue to work.

---

## 3. Architecture & Concepts {#3-architecture}

```
Customer App                  Entra External ID              Your APIs
(SPA / Mobile)                   (Tenant)
     │                              │                           │
     │ 1. Sign-up/Sign-in ────────►│                           │
     │    (social or email)         │                           │
     │                              │ 2. Authenticate user      │
     │                              │    (Google/Apple/email)    │
     │                              │                           │
     │ ◄── 3. ID Token + Access ───│                           │
     │        Token returned        │                           │
     │                              │                           │
     │ 4. Call API with ───────────────────────────────────────►│
     │    Access Token              │                           │
     │                              │                    5. Validate token
     │ ◄───────────────────── API Response ────────────────────│
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **External tenant** | Separate Entra tenant for customer identities |
| **User flow** | Pre-built authentication flow (sign-up, sign-in, profile edit) |
| **Custom auth extension** | Code-based customization (Azure Functions) |
| **Identity provider** | Authentication source (email, Google, Apple, Facebook) |
| **App registration** | Your application registered in the tenant |
| **Token claims** | Data included in the JWT token (name, email, roles) |
| **Company branding** | Custom look and feel for sign-in pages |

---

## 4. Setting Up External ID Tenant {#4-setup}

### Create External Tenant

```bash
# Via Azure Portal: Entra ID → Manage tenants → Create → Customer (External)
# Or via CLI:

# Switch to the new tenant
az login --tenant <external-tenant-id>

# Register your application
az ad app create \
  --display-name "My Customer App" \
  --sign-in-audience AzureADandPersonalMicrosoftAccount \
  --web-redirect-uris "https://myapp.com/auth/callback" "http://localhost:3000/auth/callback" \
  --enable-id-token-issuance true \
  --enable-access-token-issuance true

# Create a client secret
az ad app credential reset \
  --id <app-id> \
  --display-name "Production Secret" \
  --years 2
```

### Register API

```bash
# Register your backend API
az ad app create \
  --display-name "My Customer API" \
  --identifier-uris "api://<api-app-id>"

# Expose an API scope
az ad app update \
  --id <api-app-id> \
  --set "api.oauth2PermissionScopes=[{
    'adminConsentDescription': 'Access the API',
    'adminConsentDisplayName': 'API Access',
    'id': '$(uuidgen)',
    'isEnabled': true,
    'type': 'User',
    'userConsentDescription': 'Access the API',
    'userConsentDisplayName': 'API Access',
    'value': 'api.read'
  }]"
```

---

## 5. User Flows (Built-in) {#5-user-flows}

User flows are pre-built authentication experiences.

| Flow Type | Description |
|-----------|-------------|
| **Sign up and sign in** | Combined registration and login |
| **Sign in only** | Login only (no self-service registration) |
| **Profile editing** | Let users update their profile data |
| **Password reset** | Self-service password reset via email |

### Configure User Flow (Portal)

```
External ID tenant → User flows → New user flow

1. Select flow type: "Sign up and sign in"
2. Configure identity providers:
   ✅ Email + Password
   ✅ Google
   ✅ Apple
3. Select user attributes to collect:
   ✅ Display Name
   ✅ Email Address
   ✅ Country/Region
   ☐ Job Title (optional)
4. Configure MFA:
   ☐ Disabled / ✅ Email OTP / ✅ SMS
5. Token configuration:
   - Token lifetime: 1 hour
   - Refresh token lifetime: 24 hours
```

---

## 6. Custom Authentication Extensions {#6-custom-extensions}

Custom authentication extensions let you run code at specific points in the authentication flow using Azure Functions.

```
Authentication Flow with Custom Extensions:

User starts sign-in
     │
     ├── 🔄 OnTokenIssuanceStart
     │   └── Enrich token with custom claims
     │       (call your Azure Function)
     │       e.g., Add "loyaltyTier": "gold"
     │
     ├── 🔄 OnAttributeCollectionStart
     │   └── Pre-fill or modify attribute form
     │
     ├── 🔄 OnAttributeCollectionSubmit
     │   └── Validate submitted attributes
     │       e.g., Check if email domain is allowed
     │
     └── Token issued with custom claims
```

### Azure Function for Token Enrichment

```csharp
// Azure Function to add custom claims
[Function("OnTokenIssuanceStart")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req)
{
    var requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var data = JsonSerializer.Deserialize<TokenIssuanceRequest>(requestBody);
    
    string userId = data.Data.AuthenticationContext.User.Id;
    
    // Look up user's loyalty tier from your database
    string loyaltyTier = await GetLoyaltyTier(userId);
    
    // Return custom claims to include in the token
    var response = new
    {
        data = new
        {
            actions = new[]
            {
                new
                {
                    odataType = "microsoft.graph.tokenIssuanceStart.provideClaimsForToken",
                    claims = new
                    {
                        loyaltyTier = loyaltyTier,
                        accountType = "premium",
                        lastLogin = DateTime.UtcNow.ToString("O")
                    }
                }
            }
        }
    };
    
    return new OkObjectResult(response);
}
```

---

## 7. Social Identity Providers {#7-social-idps}

### Supported Providers

| Provider | Setup Required | Protocol |
|----------|---------------|----------|
| Microsoft Account | Pre-configured | OIDC |
| Google | Client ID + Secret | OIDC |
| Apple | App ID + Key + Team ID | OIDC |
| Facebook | App ID + Secret | OAuth 2.0 |
| Generic OIDC | Discovery URL + Client ID | OIDC |
| Generic SAML | Metadata URL + Entity ID | SAML 2.0 |

### Google Setup

```
1. Go to Google Cloud Console → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID:
   - Application type: Web application
   - Authorized redirect URI:
     https://<tenant>.ciamlogin.com/<tenant>.onmicrosoft.com/federation/oauth2
3. Copy Client ID and Client Secret
4. In Entra External ID:
   → Identity providers → Add Google
   → Paste Client ID and Secret
```

### Apple Setup

```
1. Go to Apple Developer → Certificates, Identifiers & Profiles
2. Register an App ID:
   - Enable "Sign in with Apple"
3. Create a Services ID:
   - Configure web authentication domain + return URL
4. Create a private key for Sign in with Apple
5. In Entra External ID:
   → Identity providers → Add Apple
   → Provide: Apple ID, App ID, Key ID, Certificate
```

---

## 8. Token Customization {#8-tokens}

### Add Custom Claims

```
App Registration → Token Configuration → Add optional claim

Standard claims available:
├── email         (user's email)
├── family_name   (last name)
├── given_name    (first name)
├── idtyp         (token type)
├── locale        (user's locale)
├── tenant_ctry   (country)
└── xms_pdl       (preferred data location)

Custom claims (via authentication extensions):
├── loyaltyTier   (from your database)
├── accountType   (premium/free)
├── permissions   (custom RBAC)
└── Any custom data you need
```

### JWT Token Example

```json
{
  "iss": "https://<tenant>.ciamlogin.com/<tenant-id>/v2.0",
  "sub": "AAAAABBBBBccccc-user-id",
  "aud": "<your-app-client-id>",
  "exp": 1714200000,
  "iat": 1714196400,
  "auth_time": 1714196400,
  "idp": "google.com",
  "name": "Jane Doe",
  "email": "jane@gmail.com",
  "given_name": "Jane",
  "family_name": "Doe",
  "loyaltyTier": "gold",
  "accountType": "premium"
}
```

---

## 9. Branding & UI Customization {#9-branding}

### Company Branding

```
External ID tenant → Company branding → Configure:

✅ Sign-in page:
├── Background image (1920×1080)
├── Banner logo (280×60)
├── Favicon
├── Background color (#hex)
├── Custom CSS
├── Header/footer text
└── Custom strings (localization)

✅ Language customization:
├── Override any UI string
├── Support multiple languages
└── RTL language support
```

### Custom HTML/CSS Templates

```html
<!-- Custom sign-in page template -->
<!DOCTYPE html>
<html>
<head>
  <style>
    .ext-sign-in-box {
      font-family: 'Inter', sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      border-radius: 16px;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
      padding: 2rem;
    }
    .ext-sign-in-box .ext-button-primary {
      background: #5c2d91;
      border: none;
      border-radius: 8px;
      padding: 12px 24px;
      font-weight: 600;
    }
    .ext-social-button {
      border-radius: 8px;
      margin-bottom: 8px;
    }
  </style>
</head>
<body>
  <div id="api">
    <!-- Entra External ID injects the sign-in UI here -->
  </div>
</body>
</html>
```

---

## 10. Security Best Practices {#10-security}

```
✅ Authentication
├─ Enable MFA for high-value operations
├─ Use PKCE (Proof Key for Code Exchange) for SPAs and mobile apps
├─ Set short token lifetimes (1 hour ID, 24 hour refresh)
├─ Implement refresh token rotation
└─ Use Conditional Access for risk-based policies

✅ Account Protection
├─ Enable Identity Protection (risk detection)
├─ Block sign-ups from disposable email domains
├─ Implement CAPTCHA for sign-up flows
├─ Set account lockout policies (5 failed attempts → lockout)
└─ Monitor for suspicious sign-in activity

✅ Application Security
├─ Validate ID tokens on the frontend (audience, issuer, expiry)
├─ Validate access tokens on the backend API
├─ Use confidential clients for server-side apps
├─ Store client secrets in Key Vault, not config files
└─ Implement proper CORS policies

✅ Privacy & Compliance
├─ Collect only necessary user attributes (data minimization)
├─ Provide Terms of Use and Privacy Policy links
├─ Support user data export and deletion (GDPR Right to Erasure)
├─ Enable audit logs for sign-in and admin activities
└─ Configure data residency (store data in required region)
```

---

## 11. Integration with .NET {#11-dotnet}

### ASP.NET Core Authentication

```csharp
// Program.cs
using Microsoft.Identity.Web;

var builder = WebApplication.CreateBuilder(args);

// Add Entra External ID authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(options =>
    {
        builder.Configuration.Bind("AzureAdB2C", options); // Works for External ID too
        options.TokenValidationParameters.NameClaimType = "name";
    },
    options => builder.Configuration.Bind("AzureAdB2C", options));

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

```json
// appsettings.json
{
  "AzureAdB2C": {
    "Instance": "https://<tenant>.ciamlogin.com/",
    "Domain": "<tenant>.onmicrosoft.com",
    "TenantId": "<tenant-id>",
    "ClientId": "<api-app-client-id>",
    "Scopes": "api.read"
  }
}
```

### React SPA with MSAL

```typescript
// authConfig.ts
import { Configuration } from "@azure/msal-browser";

export const msalConfig: Configuration = {
  auth: {
    clientId: "<spa-app-client-id>",
    authority: "https://<tenant>.ciamlogin.com/",
    redirectUri: "http://localhost:3000",
    knownAuthorities: ["<tenant>.ciamlogin.com"],
  },
};

export const loginRequest = {
  scopes: ["openid", "profile", "email", "api://<api-app-id>/api.read"],
};
```

---

## 12. Migration from B2C {#12-migration}

```
Azure AD B2C → Entra External ID Migration Path:

Phase 1: Assessment
├── Inventory all B2C user flows and custom policies
├── List all identity providers configured
├── Count total users and monthly active users
└── Document custom claims and API connectors

Phase 2: Parallel Setup
├── Create new External ID tenant
├── Configure identity providers
├── Create user flows (match B2C functionality)
├── Set up custom authentication extensions (replace IEF policies)
└── Register applications

Phase 3: User Migration
├── Export users from B2C (Graph API bulk export)
├── Import users to External ID (Graph API bulk create)
├── Note: Passwords CANNOT be migrated (users must reset)
├── Alternative: JIT migration (verify in B2C, create in External ID)
└── Validate user count and attributes

Phase 4: Cutover
├── Update application configurations (new tenant endpoints)
├── Update redirect URIs
├── Test all flows end-to-end
├── Switch DNS / load balancer to new endpoints
└── Monitor sign-in success rates for 30 days

Timeline: 4-8 weeks for typical migration
```
