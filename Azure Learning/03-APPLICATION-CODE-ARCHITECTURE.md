# Application Code Architecture and Development Patterns
## Document 3: Building Cloud-Native Applications with .NET

**Last Updated:** April 26, 2026  
**Document Version:** 1.0  
**Focus:** Application design, patterns, and best practices

---

## TABLE OF CONTENTS

1. [Application Architecture](#app-architecture)
2. [Dependency Injection Pattern](#dependency-injection)
3. [Code Structure Overview](#code-structure)
4. [Configuration Management](#config-management)
5. [Authentication and Authorization](#authentication)
6. [Database Integration](#database-integration)
7. [API Design](#api-design)
8. [Error Handling](#error-handling)
9. [Health Checks](#health-checks)
10. [Logging and Monitoring](#logging)

---

## Application Architecture {#app-architecture}

### Project Overview

**Application Name:** AzureLearnApp  
**Technology:** ASP.NET Core 8.0  
**Language:** C#  
**Type:** REST API  
**Pattern:** Layered Architecture

### Architecture Layers

```
┌────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                  │
│                  (API Controllers)                     │
│  - ProductsController                                 │
│  - Handles HTTP requests                              │
│  - Returns JSON responses                             │
│  - Status codes (200, 404, 500, etc)                  │
└────────────────────────────────────────────────────────┘
                           ↑↓
┌────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                       │
│              (Business Logic)                          │
│  - ICosmosDbService                                   │
│  - IKeyVaultService                                   │
│  - Validation logic                                    │
│  - Data transformation                                │
└────────────────────────────────────────────────────────┘
                           ↑↓
┌────────────────────────────────────────────────────────┐
│                    DATA LAYER                          │
│            (Database Access)                          │
│  - Cosmos DB Client                                   │
│  - Query execution                                    │
│  - Entity mapping                                     │
│  - Connection management                              │
└────────────────────────────────────────────────────────┘
                           ↑↓
┌────────────────────────────────────────────────────────┐
│              EXTERNAL SERVICES                         │
│                                                        │
│  - Azure Cosmos DB (NoSQL database)                   │
│  - Azure Key Vault (Secrets)                          │
│  - Azure AD (Authentication)                          │
└────────────────────────────────────────────────────────┘
```

### Data Flow: Creating a Product

```
1. User Request
   POST http://20.245.92.146/api/products
   Body: {"name": "Product1", "price": 99.99}
        ↓

2. Controller Layer (ProductsController)
   [HttpPost]
   public async Task<IActionResult> CreateProduct(ProductDto dto)
   {
       var product = _mapper.Map<Product>(dto);
       var result = await _cosmosDbService.CreateItemAsync(product);
       return CreatedAtAction(nameof(GetProduct), result);
   }
        ↓

3. Service Layer (ICosmosDbService)
   public async Task<T> CreateItemAsync<T>(T item)
   {
       var response = await _container.CreateItemAsync(item);
       return response.Resource;
   }
        ↓

4. Data Layer (Cosmos DB)
   INSERT INTO Products VALUES (...)
        ↓

5. Database
   Document stored in Cosmos DB
        ↓

6. Response Back
   201 Created
   Location: http://20.245.92.146/api/products/[id]
   Body: {"id": "...", "name": "Product1", ...}
```

---

## Dependency Injection Pattern {#dependency-injection}

### What is Dependency Injection (DI)?

**Without DI (Tightly Coupled):**
```csharp
public class ProductsController
{
    public ProductsController()
    {
        // ❌ Creates own instance - hard to test
        _cosmosDbService = new CosmosDbService();
        _keyVaultService = new KeyVaultService();
    }
}
```

**Problem:** 
- Can't easily swap implementation for testing
- Testing requires real database/Key Vault
- Hard to change services later

**With DI (Loosely Coupled):**
```csharp
public class ProductsController
{
    private readonly ICosmosDbService _cosmosDbService;
    private readonly IKeyVaultService _keyVaultService;
    
    // ✅ Constructor injection - receives dependencies
    public ProductsController(
        ICosmosDbService cosmosDbService,
        IKeyVaultService keyVaultService)
    {
        _cosmosDbService = cosmosDbService;
        _keyVaultService = keyVaultService;
    }
}
```

**Benefit:**
- Testing: Can inject mock services
- Flexibility: Swap implementations easily
- Maintenance: Changes in one place

### DI Container Registration

In `Program.cs`:

```csharp
// LEARNING CONCEPT: Service Registration
// The DI container knows how to create these

// Transient: New instance every time (stateless services)
builder.Services.AddTransient<IEmailService, EmailService>();

// Scoped: One instance per HTTP request (DbContext pattern)
builder.Services.AddScoped<ICosmosDbService, CosmosDbService>();

// Singleton: One instance for entire application lifetime
// (expensive to create, thread-safe)
builder.Services.AddSingleton<CosmosClient>(sp => 
{
    // Custom creation logic
    return new CosmosClient(connectionString, options);
});
```

### Registration vs Usage

```
Registration:
builder.Services.AddScoped<ICosmosDbService, CosmosDbService>();
  └─ Tells DI: "When someone asks for ICosmosDbService,
                give them a CosmosDbService instance"

Usage:
public class ProductsController
{
    public ProductsController(ICosmosDbService service)
    {
        // DI automatically provides CosmosDbService instance
        _cosmosDbService = service;
    }
}
```

---

## Code Structure Overview {#code-structure}

### Directory Layout

```
src/
├── Program.cs                    # Application entry point
├── appsettings.json             # Configuration
├── appsettings.Development.json # Dev-specific config
├── AzureLearnApp.csproj        # Project file (dependencies)
│
├── Controllers/
│   └── ProductsController.cs    # HTTP endpoints
│      └─ GET /api/products
│      └─ POST /api/products
│      └─ GET /api/products/{id}
│      └─ PUT /api/products/{id}
│      └─ DELETE /api/products/{id}
│
├── Models/
│   ├── Product.cs              # Data model
│   ├── ProductDto.cs           # Data Transfer Object (DTO)
│   └── ApiResponse.cs          # Standard response wrapper
│
├── Services/
│   ├── ICosmosDbService.cs    # Interface (contract)
│   ├── CosmosDbService.cs     # Implementation
│   ├── IKeyVaultService.cs    # Interface
│   └── KeyVaultService.cs     # Implementation
│
├── Middleware/                 # (Optional)
│   ├── ErrorHandlingMiddleware.cs
│   └── LoggingMiddleware.cs
│
└── GlobalUsings.cs            # Common using statements
```

### Key Files Explained

#### 1. Product.cs (Data Model)

```csharp
public class Product
{
    [JsonProperty("id")]
    public string Id { get; set; }
    
    [JsonProperty("name")]
    public string Name { get; set; }
    
    [JsonProperty("description")]
    public string? Description { get; set; }
    
    [JsonProperty("price")]
    public decimal Price { get; set; }
    
    [JsonProperty("stock")]
    public int Stock { get; set; }
    
    [JsonProperty("category")]
    public string Category { get; set; }
    
    [JsonProperty("createdAt")]
    public DateTime CreatedAt { get; set; }
    
    [JsonProperty("updatedAt")]
    public DateTime UpdatedAt { get; set; }
}
```

**Why `[JsonProperty]`?**
- Maps C# property names to JSON
- "CreatedAt" (C#) → "createdAt" (JSON, camelCase)
- Cosmos DB stores JSON documents

#### 2. ICosmosDbService.cs (Interface)

```csharp
public interface ICosmosDbService
{
    /// <summary>
    /// Retrieve items using SQL query
    /// </summary>
    /// <param name="query">SQL query string</param>
    /// <returns>Collection of items matching query</returns>
    Task<IEnumerable<T>> GetItemsAsync<T>(string query);
    
    /// <summary>
    /// Get single item by ID
    /// </summary>
    Task<T?> GetItemAsync<T>(string id);
    
    /// <summary>
    /// Create new item
    /// </summary>
    Task<T> CreateItemAsync<T>(T item);
    
    /// <summary>
    /// Update existing item
    /// </summary>
    Task<T> UpdateItemAsync<T>(string id, T item);
    
    /// <summary>
    /// Delete item by ID
    /// </summary>
    Task DeleteItemAsync<T>(string id);
}
```

**Why use interfaces?**
- Contract definition (what methods exist)
- Dependency injection (inject interface, not class)
- Testing (mock the interface)
- Flexibility (swap implementations)

#### 3. CosmosDbService.cs (Implementation)

```csharp
public class CosmosDbService : ICosmosDbService
{
    private readonly CosmosClient _cosmosClient;
    private readonly Container _container;
    private readonly ILogger<CosmosDbService> _logger;
    
    // Constructor receives dependencies
    public CosmosDbService(
        CosmosClient cosmosClient,
        IConfiguration configuration,
        ILogger<CosmosDbService> logger)
    {
        _cosmosClient = cosmosClient;
        _logger = logger;
        
        // Navigate to database and container
        var database = cosmosClient.GetDatabase(
            configuration["CosmosDb:DatabaseName"]);
        _container = database.GetContainer(
            configuration["CosmosDb:ContainerName"]);
    }
    
    // Get items with query
    public async Task<IEnumerable<T>> GetItemsAsync<T>(string query)
    {
        _logger.LogInformation($"Executing query: {query}");
        
        try
        {
            var items = new List<T>();
            var iterator = _container.GetItemQueryIterator<T>(query);
            
            while (iterator.HasMoreResults)
            {
                var response = await iterator.ReadNextAsync();
                items.AddRange(response);
            }
            
            _logger.LogInformation($"Query returned {items.Count} items");
            return items;
        }
        catch (CosmosException ex)
        {
            _logger.LogError($"Cosmos DB error: {ex.Message}");
            throw;
        }
    }
    
    // Create item
    public async Task<T> CreateItemAsync<T>(T item)
    {
        _logger.LogInformation($"Creating item of type {typeof(T).Name}");
        
        try
        {
            var response = await _container.CreateItemAsync(item);
            _logger.LogInformation("Item created successfully");
            return response.Resource;
        }
        catch (CosmosException ex)
        {
            _logger.LogError($"Failed to create item: {ex.Message}");
            throw;
        }
    }
}
```

**Key Points:**
- Concrete implementation of interface
- Uses dependency injection for CosmosClient
- Error handling with try-catch
- Logging for troubleshooting
- Generic `<T>` for reusability

#### 4. ProductsController.cs (HTTP Endpoints)

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ICosmosDbService _cosmosDbService;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        ICosmosDbService cosmosDbService,
        ILogger<ProductsController> logger)
    {
        _cosmosDbService = cosmosDbService;
        _logger = logger;
    }
    
    // GET /api/products
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        try
        {
            _logger.LogInformation("Fetching all products");
            var products = await _cosmosDbService.GetItemsAsync<Product>(
                "SELECT * FROM c");
            return Ok(products);
        }
        catch (Exception ex)
        {
            _logger.LogError($"Error: {ex.Message}");
            return StatusCode(StatusCodes.Status500InternalServerError);
        }
    }
    
    // GET /api/products/{id}
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<Product>> GetProduct(string id)
    {
        var product = await _cosmosDbService.GetItemAsync<Product>(id);
        
        if (product == null)
            return NotFound($"Product {id} not found");
            
        return Ok(product);
    }
    
    // POST /api/products
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status201Created)]
    public async Task<ActionResult<Product>> CreateProduct(
        CreateProductRequest request)
    {
        // Validation
        if (string.IsNullOrEmpty(request.Name))
            return BadRequest("Name is required");
        
        var product = new Product
        {
            Id = Guid.NewGuid().ToString(),
            Name = request.Name,
            Price = request.Price,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };
        
        var created = await _cosmosDbService.CreateItemAsync(product);
        
        return CreatedAtAction(nameof(GetProduct), 
            new { id = created.Id }, created);
    }
    
    // PUT /api/products/{id}
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<Product>> UpdateProduct(
        string id,
        UpdateProductRequest request)
    {
        var existing = await _cosmosDbService.GetItemAsync<Product>(id);
        if (existing == null)
            return NotFound();
        
        existing.Name = request.Name ?? existing.Name;
        existing.Price = request.Price ?? existing.Price;
        existing.UpdatedAt = DateTime.UtcNow;
        
        var updated = await _cosmosDbService.UpdateItemAsync(id, existing);
        return Ok(updated);
    }
    
    // DELETE /api/products/{id}
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteProduct(string id)
    {
        var existing = await _cosmosDbService.GetItemAsync<Product>(id);
        if (existing == null)
            return NotFound();
        
        await _cosmosDbService.DeleteItemAsync<Product>(id);
        return NoContent();
    }
}
```

**HTTP Status Codes Used:**
- `200 OK`: Request successful, data returned
- `201 Created`: Resource created
- `204 No Content`: Success, no response body
- `400 Bad Request`: Invalid input
- `404 Not Found`: Resource doesn't exist
- `500 Internal Server Error`: Server error

---

## Configuration Management {#config-management}

### appsettings.json (Non-sensitive config)

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "AllowedHosts": "*",
  "CosmosDb": {
    "DatabaseName": "AzureLearnDb",
    "ContainerName": "Products"
  },
  "KeyVault": {
    "Url": "https://azurelearnkvhof7rpcc.vault.azure.net/"
  },
  "ASPNETCORE_ENVIRONMENT": "Development",
  "ASPNETCORE_URLS": "http://+:8080"
}
```

### appsettings.Development.json (Dev-specific)

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft.AspNetCore": "Information"
    }
  },
  "CosmosDb": {
    "ConnectionString": "AccountEndpoint=https://localhost:8081;..."
  }
}
```

### How Configuration is Read in Program.cs

```csharp
var configuration = builder.Configuration;

// From appsettings.json
string dbName = configuration["CosmosDb:DatabaseName"];
// Returns: "AzureLearnDb"

// With fallback
string url = configuration["KeyVault:Url"] 
    ?? "https://default.vault.azure.net/";

// Configuration from environment variables
string env = configuration["ASPNETCORE_ENVIRONMENT"];
// In Kubernetes: Set by ConfigMap
```

### Configuration in Kubernetes (ConfigMap)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: azure-learn-app
data:
  KeyVault__Url: "https://azurelearnkvhof7rpcc.vault.azure.net/"
  CosmosDb__DatabaseName: "AzureLearnDb"
  CosmosDb__ContainerName: "Products"
  ASPNETCORE_ENVIRONMENT: "Production"
```

**Note:** Double underscore `__` → colon `:` in C# configuration

---

## Authentication and Authorization {#authentication}

### Our Authentication Flow

```
┌────────────────────────────────────────────────────────┐
│    Managed Identity Authentication Flow                │
└────────────────────────────────────────────────────────┘

Pod in AKS
    ↓
Code: var credential = new DefaultAzureCredential();
    ↓
DefaultAzureCredential checks:
    1. Environment variables (CI/CD)
    2. Managed Identity (AKS - WE USE THIS)
    3. Azure CLI credentials (Local dev)
    4. Shared token cache (VS Code)
    ↓
Found: Managed Identity (azure-learn-app-pod-identity)
    ↓
Workload Identity Provider:
    - Client ID: bf9ea53d-6351-42bf-aa45-3328a0bd296f
    - Gets JWT token from Azure AD
    ↓
Azure AD validates:
    - Is this identity legitimate?
    - What permissions does it have?
    ↓
Token returned with scopes:
    - https://vault.azure.net/.default
    - https://cosmos.azure.com/.default
    ↓
Code uses token to authenticate:
    var secretClient = new SecretClient(vaultUri, credential);
    var secret = await secretClient.GetSecretAsync("secret-name");
    ↓
Azure services verify token:
    - Is token valid?
    - Has identity permission to access?
    ↓
Access granted → Return secret
```

### Code Implementation

```csharp
// In Program.cs

// Create credential (handles token management)
var credential = new DefaultAzureCredential(
    new DefaultAzureCredentialOptions 
    { 
        ExcludeVisualStudioCodeCredential = false,
        ExcludeSharedTokenCacheCredential = false
    }
);

// Use credential with Key Vault
var keyVaultUrl = configuration["KeyVault:Url"];
var secretClient = new SecretClient(
    new Uri(keyVaultUrl), 
    credential);

// Use credential with Cosmos DB
var cosmosClient = new CosmosClient(
    cosmosConnectionString,
    new CosmosClientOptions { }
);
```

**DefaultAzureCredential Priority Order:**
1. **Environment Variables** (CI/CD systems)
2. **Managed Identity** (AKS) ← We use this
3. **Azure CLI** (Your computer)
4. **VS Code** (IDE credentials)
5. **Shared Token Cache** (Windows sign-in)

---

## Database Integration {#database-integration}

### Cosmos DB Client Setup

```csharp
builder.Services.AddSingleton<CosmosClient>(serviceProvider =>
{
    var cosmosClient = new CosmosClient(
        cosmosConnectionString,
        new CosmosClientOptions
        {
            // Connection mode: How to connect
            ConnectionMode = ConnectionMode.Gateway,
            // Gateway: Good for learning, slower
            // Direct: Better performance, requires firewall rules
            
            // Retry policy
            MaxRetryAttemptsOnRateLimitedRequests = 9,
            MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30),
            
            // Connection idle timeout
            IdleTcpConnectionTimeout = TimeSpan.FromMinutes(1),
            
            // Serializer options
            SerializerOptions = new CosmosSerializationOptions
            {
                PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
            }
        }
    );
    return cosmosClient;
});
```

### Async/Await Pattern

```csharp
// All database operations are async (non-blocking)

public async Task<IEnumerable<T>> GetItemsAsync<T>(string query)
{
    // await: Wait for database response
    // Don't block thread while waiting
    var iterator = _container.GetItemQueryIterator<T>(query);
    
    while (iterator.HasMoreResults)
    {
        // ReadNextAsync: Non-blocking call to database
        var response = await iterator.ReadNextAsync();
        // Continue only when response arrives
    }
}

// Why async?
// - Free up thread to handle other requests
// - Better scalability (more concurrent users)
// - Non-blocking I/O (fast response)
```

### Error Handling

```csharp
try
{
    // Database operation
    await _container.CreateItemAsync(item);
}
catch (CosmosException ex)
{
    // Cosmos DB specific errors
    if (ex.StatusCode == System.Net.HttpStatusCode.Conflict)
    {
        // Item already exists
        _logger.LogWarning("Item already exists");
    }
    else if (ex.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
    {
        // Rate limited (too many requests)
        // Cosmos DB returns 429
        _logger.LogWarning("Rate limited, retry later");
    }
    else if (ex.StatusCode == System.Net.HttpStatusCode.RequestTimeout)
    {
        // Network timeout
        _logger.LogError("Request timeout");
    }
    else
    {
        // Other errors
        _logger.LogError($"Error: {ex.Message}");
    }
    
    throw;
}
```

---

## API Design {#api-design}

### RESTful Principles

```
REST = Representational State Transfer

Principle 1: Resources, not Actions
❌ Bad:  GET /api/GetProducts
✅ Good: GET /api/products

Principle 2: Use HTTP Methods
✅ GET    /api/products       → Retrieve all
✅ GET    /api/products/{id}  → Retrieve one
✅ POST   /api/products       → Create new
✅ PUT    /api/products/{id}  → Update
✅ DELETE /api/products/{id}  → Delete

Principle 3: Status Codes Convey Meaning
✅ 200: OK (success)
✅ 201: Created (new resource created)
✅ 204: No Content (success, no body)
✅ 400: Bad Request (client error)
✅ 404: Not Found (resource missing)
✅ 500: Internal Server Error (server error)

Principle 4: Consistent Data Format
✅ Always JSON request/response
✅ Use camelCase for properties
✅ Include timestamps
✅ Standard error response format
```

### Request/Response Examples

```
Create Product:
─────────────
POST /api/products
Content-Type: application/json

{
  "name": "Laptop",
  "price": 999.99,
  "stock": 10,
  "category": "Electronics"
}

Response: 201 Created
Location: /api/products/550e8400-e29b-41d4-a716-446655440000

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Laptop",
  "price": 999.99,
  "stock": 10,
  "category": "Electronics",
  "createdAt": "2026-04-26T10:30:00Z",
  "updatedAt": "2026-04-26T10:30:00Z"
}

─────────────────────────────────

Get Product:
─────────────
GET /api/products/550e8400-e29b-41d4-a716-446655440000

Response: 200 OK

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Laptop",
  "price": 999.99,
  "stock": 10,
  "category": "Electronics",
  "createdAt": "2026-04-26T10:30:00Z",
  "updatedAt": "2026-04-26T10:30:00Z"
}

─────────────────────────────────

Error Response:
─────────────
GET /api/products/invalid-id

Response: 404 Not Found

{
  "error": "Product not found",
  "statusCode": 404,
  "timestamp": "2026-04-26T10:35:00Z"
}
```

---

## Error Handling {#error-handling}

### Exception Hierarchy

```
Exception
├─ SystemException
│  ├─ ArgumentException (Invalid arguments)
│  ├─ NullReferenceException (Null value)
│  └─ TimeoutException (Operation timeout)
│
├─ CosmosException (Cosmos DB errors)
│  ├─ 404 Not Found
│  ├─ 409 Conflict
│  ├─ 429 Too Many Requests
│  └─ 500 Internal Server Error
│
└─ HttpRequestException (Network errors)
   ├─ Connection refused
   ├─ DNS resolution failed
   └─ SSL certificate error
```

### Global Exception Middleware

```csharp
// In Program.cs
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandlerPathFeature = 
            context.Features.Get<IExceptionHandlerPathFeature>();
        var exception = exceptionHandlerPathFeature?.Error;
        
        var response = new
        {
            message = exception?.Message,
            statusCode = context.Response.StatusCode,
            timestamp = DateTime.UtcNow
        };
        
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(response);
    });
});
```

---

## Health Checks {#health-checks}

### Why Health Checks?

```
Kubernetes needs to know:
1. Is app running? (Liveness probe)
2. Is app ready for traffic? (Readiness probe)

Liveness Probe:
├─ Runs every 30 seconds
├─ If fails 3 times → Container restarts
├─ Checks: Is app process running?
└─ Endpoint: /health/live

Readiness Probe:
├─ Runs every 10 seconds
├─ If fails → Pod excluded from load balancer
├─ Checks: Is app ready to handle requests?
│  ├─ Database connected?
│  ├─ Dependencies available?
│  └─ Warmup completed?
└─ Endpoint: /health/ready
```

### Implementation

```csharp
// In Program.cs

// Register health checks
builder.Services.AddHealthChecks()
    .AddCheck<CosmosDbHealthCheck>("cosmosdb")
    .AddCheck<KeyVaultHealthCheck>("keyvault");

// Map endpoints
app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = healthCheck => healthCheck.Tags.Contains("startup")
});
```

### Custom Health Check

```csharp
public class CosmosDbHealthCheck : IHealthCheck
{
    private readonly ICosmosDbService _cosmosDbService;
    
    public CosmosDbHealthCheck(ICosmosDbService cosmosDbService)
    {
        _cosmosDbService = cosmosDbService;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Try to read a document
            var products = await _cosmosDbService.GetItemsAsync<Product>(
                "SELECT TOP 1 * FROM c");
            
            return HealthCheckResult.Healthy(
                "Cosmos DB is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Cosmos DB is not accessible",
                ex);
        }
    }
}
```

### Kubernetes Probe Configuration

```yaml
containers:
- name: app
  image: azurelearnacrhof7rpcc.azurecr.io/azurelearn:v3.0
  
  # Liveness probe
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8080
    initialDelaySeconds: 10      # Wait 10s after start
    periodSeconds: 30             # Check every 30s
    failureThreshold: 3           # Restart after 3 fails
    timeoutSeconds: 5             # Request timeout
  
  # Readiness probe
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
    failureThreshold: 2
    timeoutSeconds: 5
```

---

## Logging and Monitoring {#logging}

### Logging Levels

```
Trace     (0)  - Detailed trace info (rarely used)
Debug     (1)  - Debug info (development only)
Information (2) - General informational messages
Warning   (3)  - Warning messages (potential issues)
Error     (4)  - Error messages (failures)
Critical  (5)  - Critical errors (system-level failures)
None      (6)  - No logging
```

### Logging in Code

```csharp
public class ProductsController : ControllerBase
{
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(ILogger<ProductsController> logger)
    {
        _logger = logger;
    }
    
    [HttpGet]
    public async Task<IActionResult> GetProducts()
    {
        _logger.LogInformation("GetProducts called");
        
        try
        {
            var products = await _cosmosDbService.GetItemsAsync();
            _logger.LogInformation($"Retrieved {products.Count} products");
            return Ok(products);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error fetching products");
            return StatusCode(500);
        }
    }
}
```

### Structured Logging

```csharp
// Bad (string concatenation)
_logger.LogInformation($"Product {productId} created by user {userId}");

// Good (structured logging)
_logger.LogInformation(
    "Product {ProductId} created by user {UserId}",
    productId, userId);

// Searchable in Azure Monitor:
// Find all logs: ProductId = "123"
// Find all logs: UserId = "user@example.com"
```

### Correlation IDs (Trace Request)

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<LoggingMiddleware> _logger;
    
    public LoggingMiddleware(RequestDelegate next, 
        ILogger<LoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = Guid.NewGuid().ToString();
        context.Items["CorrelationId"] = correlationId;
        
        using (_logger.BeginScope(
            new Dictionary<string, object>
            {
                ["CorrelationId"] = correlationId
            }))
        {
            _logger.LogInformation(
                "Request started: {Method} {Path}",
                context.Request.Method,
                context.Request.Path);
            
            await _next(context);
            
            _logger.LogInformation(
                "Request completed: {StatusCode}",
                context.Response.StatusCode);
        }
    }
}
```

**Benefits:**
- Track single request through all services
- Find related logs in distributed system
- Debug complex multi-service issues

---

## Key Takeaways

✅ **Layered Architecture:** Controllers → Services → Data  
✅ **Dependency Injection:** Loose coupling, easy testing  
✅ **Async/Await:** Non-blocking, scalable operations  
✅ **Configuration Management:** Separate config from code  
✅ **Error Handling:** Try-catch, logging, meaningful responses  
✅ **Health Checks:** Kubernetes probes, automatic restart  
✅ **Structured Logging:** Searchable, monitorable logs  
✅ **RESTful API Design:** Standard methods, status codes, resources  

**Next Document:** Document 4 will cover Infrastructure as Code (Bicep) and deployment templates.
