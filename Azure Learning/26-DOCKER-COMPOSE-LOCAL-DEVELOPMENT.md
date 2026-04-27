# Docker Compose for Local Development
## Document 26: Multi-Container Local Development Workflow

**Last Updated:** April 27, 2026  
**Document Version:** 1.0  
**Focus:** Docker Compose V2, multi-service .NET apps, local development with databases, debugging, hot reload

---

## TABLE OF CONTENTS

1. [Why Docker Compose?](#1-why)
2. [Docker Compose Fundamentals](#2-fundamentals)
3. [compose.yaml Syntax Reference](#3-syntax)
4. [.NET Multi-Service Example](#4-dotnet-example)
5. [Working with Databases Locally](#5-databases)
6. [Debugging Containers in VS/VS Code](#6-debugging)
7. [Hot Reload & Development Mode](#7-hot-reload)
8. [Networking Between Services](#8-networking)
9. [Environment & Secrets Management](#9-environment)
10. [Common CLI Commands](#10-commands)
11. [Best Practices](#11-best-practices)

---

## 1. Why Docker Compose? {#1-why}

Docker Compose lets you define and run **multi-container applications** with a single YAML file. Instead of running multiple `docker run` commands, you define all services, networks, and volumes in `compose.yaml`.

```
Without Compose:                    With Compose:
──────────────────                  ─────────────────
docker run --name sql ...           docker compose up
docker run --name redis ...         (starts everything)
docker run --name api ...
docker run --name web ...           docker compose down
docker network create ...           (stops everything)
docker volume create ...
```

**Key Benefits:**
- **One command** to spin up entire development stack
- **Reproducible** environments across team members
- **Isolated** from the host machine's installed software
- **Matches production** container configurations
- **Version controlled** alongside application code

---

## 2. Docker Compose Fundamentals {#2-fundamentals}

### File Naming

Docker Compose V2 uses `compose.yaml` (preferred) or `docker-compose.yml` (legacy, still works).

```
Project Root/
├── compose.yaml              ← Main compose file
├── compose.override.yaml     ← Auto-merged (dev overrides)
├── compose.prod.yaml         ← Production overrides
├── src/
│   ├── Api/
│   │   └── Dockerfile
│   └── Web/
│       └── Dockerfile
└── .env                      ← Environment variables
```

### Compose V2 vs V1

| Feature | V1 (Legacy) | V2 (Current) |
|---------|-------------|--------------|
| Command | `docker-compose` | `docker compose` (subcommand) |
| Config file | `docker-compose.yml` | `compose.yaml` |
| Profiles | ❌ | ✅ |
| GPU support | ❌ | ✅ |
| Build context options | Limited | Extended |
| Watch mode | ❌ | ✅ (`docker compose watch`) |

---

## 3. compose.yaml Syntax Reference {#3-syntax}

```yaml
# compose.yaml — Full Annotated Reference

# Optional: Project name (defaults to directory name)
name: my-app

# ─── SERVICES ──────────────────────────────────────────────
services:

  api:
    # Build from Dockerfile
    build:
      context: ./src/Api           # Build context directory
      dockerfile: Dockerfile       # Dockerfile path (relative to context)
      target: development          # Multi-stage build target
      args:                        # Build arguments
        DOTNET_VERSION: "8.0"
    
    # OR use a pre-built image
    # image: myacr.azurecr.io/api:latest
    
    # Container configuration
    container_name: myapp-api      # Custom container name (optional)
    ports:
      - "5000:8080"                # host:container
      - "5001:8081"                # Additional ports
    
    environment:                   # Environment variables
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=mydb;User=sa;Password=P@ssw0rd!;TrustServerCertificate=True
    
    env_file:                      # Load from .env file
      - .env
    
    volumes:
      - ./src/Api:/app/src         # Bind mount (host ← → container)
      - api-data:/app/data         # Named volume
    
    depends_on:                    # Start order
      db:
        condition: service_healthy # Wait for health check
      redis:
        condition: service_started
    
    networks:
      - backend
    
    restart: unless-stopped        # Restart policy
    
    healthcheck:                   # Container health check
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    
    profiles:                      # Only start with specific profile
      - dev
      - full

  web:
    build:
      context: ./src/Web
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://api:8080    # Use service name as hostname
    depends_on:
      - api
    networks:
      - backend

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=P@ssw0rd!
    ports:
      - "1433:1433"
    volumes:
      - sql-data:/var/opt/mssql    # Persist data between restarts
    healthcheck:
      test: /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "P@ssw0rd!" -Q "SELECT 1" -C
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - backend

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - backend

# ─── VOLUMES ───────────────────────────────────────────────
volumes:
  sql-data:                        # Named volume (persistent)
  redis-data:
  api-data:

# ─── NETWORKS ──────────────────────────────────────────────
networks:
  backend:
    driver: bridge                 # Default network driver
```

---

## 4. .NET Multi-Service Example {#4-dotnet-example}

### Project Structure

```
ecommerce/
├── compose.yaml
├── compose.override.yaml
├── .env
├── src/
│   ├── Api/
│   │   ├── Dockerfile
│   │   ├── Api.csproj
│   │   ├── Program.cs
│   │   └── Controllers/
│   ├── Worker/
│   │   ├── Dockerfile
│   │   ├── Worker.csproj
│   │   └── Worker.cs
│   └── Shared/
│       └── Shared.csproj
└── tests/
    └── Api.Tests/
```

### compose.yaml (Base)

```yaml
name: ecommerce

services:
  api:
    build:
      context: .
      dockerfile: src/Api/Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__SqlServer=Server=db;Database=ECommerce;User=sa;Password=${DB_PASSWORD};TrustServerCertificate=True
      - ConnectionStrings__Redis=redis:6379
      - AzureKeyVault__Endpoint=https://myvault.vault.azure.net
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  worker:
    build:
      context: .
      dockerfile: src/Worker/Dockerfile
    environment:
      - ConnectionStrings__SqlServer=Server=db;Database=ECommerce;User=sa;Password=${DB_PASSWORD};TrustServerCertificate=True
      - ServiceBus__ConnectionString=${SERVICEBUS_CONN}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=${DB_PASSWORD}
    ports:
      - "1433:1433"
    volumes:
      - sql-data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "${DB_PASSWORD}" -Q "SELECT 1" -C
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  sql-data:
```

### .env File

```bash
# .env — DO NOT commit to source control
DB_PASSWORD=YourStr0ng!P@ssw0rd
SERVICEBUS_CONN=Endpoint=sb://mybus.servicebus.windows.net/;SharedAccessKeyName=...
```

### Dockerfile for .NET API (Development)

```dockerfile
# src/Api/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS development
WORKDIR /app

# Copy project files for restore
COPY src/Shared/Shared.csproj src/Shared/
COPY src/Api/Api.csproj src/Api/
RUN dotnet restore src/Api/Api.csproj

# Copy everything and build
COPY src/ src/
WORKDIR /app/src/Api
RUN dotnet build -c Debug

# Development: use dotnet watch for hot reload
ENTRYPOINT ["dotnet", "watch", "run", "--no-launch-profile", "--urls", "http://+:8080"]

# ─── Production Stage ───
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app
COPY src/Shared/Shared.csproj src/Shared/
COPY src/Api/Api.csproj src/Api/
RUN dotnet restore src/Api/Api.csproj
COPY src/ src/
RUN dotnet publish src/Api/Api.csproj -c Release -o /out --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS production
WORKDIR /app
COPY --from=build /out .
EXPOSE 8080
USER app
ENTRYPOINT ["dotnet", "Api.dll"]
```

---

## 5. Working with Databases Locally {#5-databases}

### SQL Server

```yaml
db:
  image: mcr.microsoft.com/mssql/server:2022-latest
  environment:
    - ACCEPT_EULA=Y
    - MSSQL_SA_PASSWORD=P@ssw0rd!2024
  ports:
    - "1433:1433"
  volumes:
    - sql-data:/var/opt/mssql
    - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # seed data
```

### PostgreSQL

```yaml
postgres:
  image: postgres:16-alpine
  environment:
    POSTGRES_USER: appuser
    POSTGRES_PASSWORD: P@ssw0rd!
    POSTGRES_DB: mydb
  ports:
    - "5432:5432"
  volumes:
    - pg-data:/var/lib/postgresql/data
    - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
```

### MongoDB

```yaml
mongo:
  image: mongo:7
  environment:
    MONGO_INITDB_ROOT_USERNAME: root
    MONGO_INITDB_ROOT_PASSWORD: P@ssw0rd!
  ports:
    - "27017:27017"
  volumes:
    - mongo-data:/data/db
```

### Cosmos DB Emulator

```yaml
cosmos:
  image: mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest
  environment:
    - AZURE_COSMOS_EMULATOR_PARTITION_COUNT=5
    - AZURE_COSMOS_EMULATOR_ENABLE_DATA_PERSISTENCE=true
  ports:
    - "8081:8081"     # HTTPS endpoint
    - "10250-10255:10250-10255"
  volumes:
    - cosmos-data:/tmp/cosmos/appdata
```

### Azurite (Azure Storage Emulator)

```yaml
azurite:
  image: mcr.microsoft.com/azure-storage/azurite:latest
  ports:
    - "10000:10000"   # Blob
    - "10001:10001"   # Queue
    - "10002:10002"   # Table
  volumes:
    - azurite-data:/data
  command: "azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0"
```

---

## 6. Debugging Containers in VS/VS Code {#6-debugging}

### Visual Studio

```json
// launchSettings.json — Docker Compose profile
{
  "profiles": {
    "Docker Compose": {
      "commandName": "DockerCompose",
      "serviceActions": {
        "api": "StartDebugging",
        "worker": "StartDebugging"
      }
    }
  }
}
```

### VS Code (devcontainer or attach)

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker .NET Attach",
      "type": "docker",
      "request": "attach",
      "platform": "netCore",
      "sourceFileMap": {
        "/app/src": "${workspaceFolder}/src"
      }
    }
  ]
}
```

---

## 7. Hot Reload & Development Mode {#7-hot-reload}

### Docker Compose Watch (V2.22+)

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/Api/Dockerfile
      target: development
    develop:
      watch:
        - action: sync            # Sync files into running container
          path: ./src/Api
          target: /app/src/Api
          ignore:
            - bin/
            - obj/
        - action: rebuild          # Rebuild container when these change
          path: ./src/Api/Api.csproj
```

```bash
# Start with watch mode
docker compose watch

# Changes to .cs files → synced instantly → dotnet watch detects → hot reload
# Changes to .csproj → triggers full container rebuild
```

### Volume Mount Approach (Alternative)

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/Api/Dockerfile
      target: development
    volumes:
      - ./src:/app/src             # Mount source code
    command: ["dotnet", "watch", "run", "--project", "src/Api/Api.csproj"]
```

---

## 8. Networking Between Services {#8-networking}

```
┌──────────── Docker Network (bridge: ecommerce_default) ──────────┐
│                                                                    │
│  api (ecommerce-api)         web (ecommerce-web)                  │
│  ┌──────────────────┐        ┌──────────────────┐                 │
│  │ host:5000         │        │ host:3000         │                 │
│  │ container:8080    │◄───────│ API_URL=          │                 │
│  │                  │        │  http://api:8080  │                 │
│  └──────┬───────────┘        └──────────────────┘                 │
│         │                                                          │
│         │ conn: Server=db                                          │
│         ▼                                                          │
│  db (ecommerce-db)           redis (ecommerce-redis)              │
│  ┌──────────────────┐        ┌──────────────────┐                 │
│  │ host:1433         │        │ host:6379         │                 │
│  │ container:1433    │        │ container:6379    │                 │
│  └──────────────────┘        └──────────────────┘                 │
└────────────────────────────────────────────────────────────────────┘

KEY: Service names ARE the hostnames within the Docker network.
     "api" can reach "db" at hostname "db" on port 1433.
     "web" can reach "api" at hostname "api" on port 8080.
```

**Rules:**
- Services communicate using **service names** as hostnames
- Use the **container port** (not the host-mapped port) for inter-service communication
- Host port mappings are only for accessing services from your machine

---

## 9. Environment & Secrets Management {#9-environment}

```yaml
# Priority order (highest to lowest):
# 1. CLI: docker compose run -e VAR=val
# 2. environment: in compose.yaml
# 3. env_file: in compose.yaml
# 4. .env file (auto-loaded)
# 5. Dockerfile ENV

services:
  api:
    env_file:
      - .env                     # Default environment
      - .env.local               # Local overrides (gitignored)
    environment:
      - ASPNETCORE_ENVIRONMENT=Development   # Inline override
```

### .gitignore

```gitignore
# Never commit secrets
.env
.env.local
.env.*.local
```

---

## 10. Common CLI Commands {#10-commands}

```bash
# ─── LIFECYCLE ──────────────────────────────────────────────
docker compose up                  # Start all services (foreground)
docker compose up -d               # Start in background (detached)
docker compose up --build          # Force rebuild images
docker compose up api db           # Start specific services only
docker compose down                # Stop and remove containers
docker compose down -v             # Also remove volumes (fresh start)
docker compose restart api         # Restart specific service

# ─── MONITORING ─────────────────────────────────────────────
docker compose ps                  # List running services
docker compose logs                # View all logs
docker compose logs api -f         # Follow logs for specific service
docker compose logs --tail 50      # Last 50 lines

# ─── EXECUTION ──────────────────────────────────────────────
docker compose exec api bash       # Shell into running container
docker compose exec db sqlcmd -S localhost -U sa -P "P@ss" -C
docker compose run --rm api dotnet test   # Run one-off command

# ─── BUILDING ──────────────────────────────────────────────
docker compose build               # Build all images
docker compose build api           # Build specific service
docker compose build --no-cache    # Full rebuild (no layer cache)

# ─── PROFILES ──────────────────────────────────────────────
docker compose --profile dev up    # Start services with "dev" profile
docker compose --profile full up   # Start services with "full" profile

# ─── OVERRIDES ─────────────────────────────────────────────
docker compose -f compose.yaml -f compose.prod.yaml up
```

---

## 11. Best Practices {#11-best-practices}

```
✅ Development Workflow
├─ Use compose.yaml for base config, compose.override.yaml for dev settings
├─ Use .env files for secrets (never commit to source control)
├─ Use depends_on with health checks for reliable startup order
├─ Use named volumes for database persistence across restarts
├─ Use docker compose watch for instant hot reload
└─ Match container images to your production versions

✅ Performance
├─ Use bind mounts for source code (fast sync)
├─ Use named volumes for node_modules, .nuget cache (avoid host mounts)
├─ Add .dockerignore to exclude bin/, obj/, node_modules/
├─ Use BuildKit (DOCKER_BUILDKIT=1) for faster builds
└─ Use multi-stage Dockerfiles with "development" target

✅ Team Collaboration
├─ Commit compose.yaml, Dockerfiles, and .env.example to source control
├─ Use compose profiles to make optional services opt-in
├─ Document the setup in README: "Run docker compose up to start"
├─ Pin image versions (redis:7-alpine not redis:latest)
└─ Provide seed data scripts for databases

✅ Security
├─ Never put production credentials in compose files
├─ Use different passwords for local dev vs production
├─ Don't expose database ports in production compose files
└─ Use read-only mounts where possible
```
