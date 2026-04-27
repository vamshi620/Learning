# Prepare Your .NET App for Cloud Deployment
## File 02: Make Your App Cloud-Ready (Works for Both App Service and AKS)

---

## What You'll Do in This File

By the end of this guide, your .NET app will be ready for cloud deployment:
- ✅ Health check endpoint added
- ✅ Configuration externalized (no hardcoded settings)
- ✅ Structured logging configured
- ✅ Dockerfile created (optional for App Service, required for AKS)
- ✅ App tested locally with cloud-like settings

**Time Required:** 1-2 hours

---

## Why Prepare? — The Cloud-Ready Difference

```
❌ Local-Only App:                    ✅ Cloud-Ready App:
─────────────────                     ────────────────────
Hardcoded connection strings          Config from environment variables
Console.WriteLine for logging         Structured logging (JSON/Serilog)
No health check                       /health endpoint for monitoring
Listens on localhost:5000              Listens on configurable port
Secrets in appsettings.json           Secrets in Key Vault / env vars
No Dockerfile                         Multi-stage Dockerfile
```

---

## Step 1: Add Health Check Endpoints

### What Are Health Checks?

Azure uses health checks to know if your app is **alive and ready** to receive traffic. If the health check fails, Azure automatically restarts or replaces your app instance.

### Add to Your Project

```powershell
# If you don't have the health checks package yet:
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks
dotnet add package AspNetCore.HealthChecks.SqlServer    # if using SQL Server
dotnet add package AspNetCore.HealthChecks.Redis        # if using Redis
```

### Configure in Program.cs

```csharp
// Program.cs — Add health checks

var builder = WebApplication.CreateBuilder(args);

// ─── HEALTH CHECKS ────────────────────────────────────────────────
// These tell Azure "my app is working correctly"
builder.Services.AddHealthChecks()
    // Check that the database is reachable
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready" })    // "ready" tag = needed for full functionality
    // Add more checks as needed:
    // .AddRedis(builder.Configuration.GetConnectionString("Redis")!, name: "redis")
    // .AddUrlGroup(new Uri("https://external-api.com/health"), name: "external-api")
    ;

// ... your other services (controllers, swagger, etc.)

var app = builder.Build();

// ─── HEALTH CHECK ENDPOINTS ───────────────────────────────────────
// /health/live — "Is the process running?" (basic alive check)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    // No checks needed — if this endpoint responds, the app is alive
    Predicate = _ => false
});

// /health/ready — "Is the app fully functional?" (checks DB, Redis, etc.)
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

// ... your other middleware (app.UseHttpsRedirection, etc.)
app.Run();
```

### What Each Endpoint Does

```
GET /health/live   → Returns 200 OK if the process is running
                     Used by: Kubernetes liveness probe, App Service health check
                     Failure action: Restart the container/instance

GET /health/ready  → Returns 200 OK if DB + all dependencies are accessible
                     Used by: Kubernetes readiness probe, Load Balancer
                     Failure action: Stop sending traffic to this instance
```

### Test It

```powershell
# Run your app
dotnet run

# In another terminal, test health endpoints
curl http://localhost:5000/health/live    # Should return: Healthy
curl http://localhost:5000/health/ready   # Should return: Healthy (if DB is accessible)
```

---

## Step 2: Externalize Configuration

### The Problem

```csharp
// ❌ BAD — Never do this!
var connectionString = "Server=myserver.database.windows.net;Database=mydb;User=admin;Password=secret123";
```

### The Solution — Configuration Hierarchy

.NET reads configuration from multiple sources (in priority order):

```
Priority (highest wins):
1. Command-line arguments          ← Override anything from CLI
2. Environment variables           ← Set by Azure / Docker / K8s
3. User Secrets (local dev only)   ← dotnet user-secrets
4. appsettings.{Environment}.json  ← Per-environment settings
5. appsettings.json                ← Default settings
```

### Set Up Your Configuration Files

**appsettings.json** — Default values (committed to Git):
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ApplicationSettings": {
    "AppName": "MyProject API",
    "MaxPageSize": 50
  }
}
```

**appsettings.Development.json** — Dev overrides (committed to Git):
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=mydb;Trusted_Connection=true;TrustServerCertificate=true"
  }
}
```

**For secrets in local development, use User Secrets:**
```powershell
# Initialize user secrets for your project
dotnet user-secrets init

# Store secrets locally (NOT in Git, stored in your user profile)
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=mydb;User=sa;Password=YourP@ss;TrustServerCertificate=true"

# View stored secrets
dotnet user-secrets list
```

### Read Configuration in Your Code

```csharp
// Program.cs — Configuration is already loaded by default
var builder = WebApplication.CreateBuilder(args);

// Access configuration anywhere:
var appName = builder.Configuration["ApplicationSettings:AppName"];
var connString = builder.Configuration.GetConnectionString("DefaultConnection");

// Or bind to a strongly-typed class:
builder.Services.Configure<ApplicationSettings>(
    builder.Configuration.GetSection("ApplicationSettings"));

// Use in a controller or service with dependency injection:
public class MyController : ControllerBase
{
    private readonly IOptions<ApplicationSettings> _settings;
    
    public MyController(IOptions<ApplicationSettings> settings)
    {
        _settings = settings;
    }
    
    [HttpGet("info")]
    public IActionResult GetInfo() => Ok(new { _settings.Value.AppName });
}

// Settings class:
public class ApplicationSettings
{
    public string AppName { get; set; } = "";
    public int MaxPageSize { get; set; } = 50;
}
```

### How Azure Overrides Configuration

```
In Azure App Service:
  → Configuration → Application Settings
  → Set: ConnectionStrings__DefaultConnection = "Server=azure-sql.database.windows.net..."
  → Azure injects this as an environment variable
  → .NET picks it up automatically (overrides appsettings.json)

In AKS (Kubernetes):
  → ConfigMap or Secret
  → Mounted as environment variables in the pod
  → .NET picks it up automatically
```

**Key rule:** Your code never changes between environments. Only configuration changes.

---

## Step 3: Set Up Structured Logging

### Why Structured Logging?

```
❌ Unstructured: "User john@email.com placed order 1234 for $99.99"
   → Hard to search, hard to parse, hard to alert on

✅ Structured: {"event":"OrderPlaced","user":"john@email.com","orderId":1234,"total":99.99}
   → Searchable, filterable, alertable in Azure Monitor
```

### Add Serilog

```powershell
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.ApplicationInsights    # For Azure monitoring
```

### Configure in Program.cs

```csharp
using Serilog;

// Set up Serilog BEFORE anything else
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", Serilog.Events.LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .CreateLogger();

try
{
    Log.Information("Starting application");
    
    var builder = WebApplication.CreateBuilder(args);
    builder.Host.UseSerilog();   // ← Replace default logging with Serilog
    
    // ... your services
    
    var app = builder.Build();
    
    // Log every HTTP request automatically
    app.UseSerilogRequestLogging();
    
    // ... your middleware
    
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### Use Logging in Your Code

```csharp
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;
    
    public OrdersController(ILogger<OrdersController> logger)
    {
        _logger = logger;
    }
    
    [HttpPost]
    public IActionResult CreateOrder(OrderRequest request)
    {
        // ✅ Structured logging — properties are searchable in Azure Monitor
        _logger.LogInformation("Order created for {UserId} with total {OrderTotal}",
            request.UserId, request.Total);
        
        // ❌ Don't do this — string interpolation loses structure
        // _logger.LogInformation($"Order created for {request.UserId} with total {request.Total}");
        
        return Ok();
    }
}
```

---

## Step 4: Configure Port and Host

Azure App Service and AKS containers need your app to listen on a specific port.

```csharp
// Program.cs — Listen on the right port
var builder = WebApplication.CreateBuilder(args);

// This makes your app listen on the port Azure/Docker expects
// In App Service: Azure sets PORT automatically
// In Docker/AKS: You control it via ASPNETCORE_URLS
builder.WebHost.UseUrls("http://+:8080");
// OR use environment variable (more flexible):
// Set ASPNETCORE_URLS=http://+:8080

var app = builder.Build();
app.Run();
```

**appsettings.json** (add Kestrel config):
```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://+:8080"
      }
    }
  }
}
```

---

## Step 5: Create a Dockerfile (Required for AKS, Optional for App Service)

### What is a Dockerfile?

A Dockerfile is a recipe that tells Docker how to package your .NET app into a **container image** — a self-contained package with your app, the .NET runtime, and everything it needs to run.

### Create the Dockerfile

Create a file named `Dockerfile` (no extension) in your project root:

```dockerfile
# ══════════════════════════════════════════════════════════════════
# Stage 1: BUILD — Use the full SDK to compile your app
# ══════════════════════════════════════════════════════════════════
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project file first (for Docker layer caching)
# This means NuGet restore only re-runs when .csproj changes
COPY ["MyProject.Api/MyProject.Api.csproj", "MyProject.Api/"]
RUN dotnet restore "MyProject.Api/MyProject.Api.csproj"

# Now copy everything else and build
COPY . .
WORKDIR /src/MyProject.Api
RUN dotnet publish -c Release -o /app/publish --no-restore

# ══════════════════════════════════════════════════════════════════
# Stage 2: RUNTIME — Use the slim runtime image (no SDK)
# ══════════════════════════════════════════════════════════════════
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Security: Run as non-root user
USER app

# Copy only the published output from the build stage
COPY --from=build /app/publish .

# Tell Docker which port the app listens on
EXPOSE 8080

# Set the entry point
ENTRYPOINT ["dotnet", "MyProject.Api.dll"]
```

### Why Two Stages?

```
SDK image: ~750MB (has compiler, NuGet, build tools)
Runtime image: ~220MB (only has .NET runtime)

Multi-stage builds use the SDK to compile, then copy
ONLY the compiled output to the slim runtime image.
Result: Smaller, faster, more secure container.
```

### Create .dockerignore

Create `.dockerignore` in the same folder as your Dockerfile:

```
# Don't copy these into the Docker build context
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/node_modules/
**/.git/
**/*.user
**/*.suo
**/Thumbs.db
.dockerignore
docker-compose*.yml
*.md
LICENSE
.gitignore
.env
```

### Build and Test Locally

```powershell
# Build the Docker image
docker build -t myproject-api:dev .

# Run the container
docker run -d -p 5000:8080 --name myapi myproject-api:dev

# Test it
curl http://localhost:5000/health/live

# View logs
docker logs myapi

# Stop and remove
docker stop myapi
docker rm myapi
```

---

## Step 6: Solution Structure Recommendation

```
MyProject/
├── MyProject.sln
├── Dockerfile                    ← Container build recipe
├── .dockerignore                 ← Files to exclude from Docker
├── .gitignore
│
├── src/
│   ├── MyProject.Api/            ← Web API project
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json
│   │   └── MyProject.Api.csproj
│   │
│   ├── MyProject.Core/           ← Business logic (no dependencies)
│   │   ├── Models/
│   │   ├── Interfaces/
│   │   └── Services/
│   │
│   └── MyProject.Infrastructure/ ← Data access, external services
│       ├── Data/
│       └── Repositories/
│
├── tests/
│   └── MyProject.Tests/
│       └── MyProject.Tests.csproj
│
└── k8s/                          ← Kubernetes manifests (for AKS path)
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

---

## ✅ Cloud-Ready Checklist

Before deploying, verify your app has:

- [ ] Health check endpoints (`/health/live` and `/health/ready`)
- [ ] No hardcoded connection strings or secrets
- [ ] Configuration read from environment variables / appsettings
- [ ] Structured logging with Serilog
- [ ] App listens on port 8080 (configurable)
- [ ] Dockerfile created and tested locally (for AKS)
- [ ] `.dockerignore` created
- [ ] All secrets in User Secrets (local) — never in appsettings.json

---

> **Next Step:**
> - **App Service path:** Open [03-DEPLOY-TO-APP-SERVICE.md](03-DEPLOY-TO-APP-SERVICE.md)
> - **AKS path:** Open [04-DEPLOY-TO-AKS-STEP-BY-STEP.md](04-DEPLOY-TO-AKS-STEP-BY-STEP.md)
