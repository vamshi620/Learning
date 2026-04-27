# Database Migrations with EF Core in Azure
## File 12: Managing SQL Schema Changes in App Service and AKS

---

## What You'll Do in This File

By the end of this guide, you'll have:
- ✅ A strategy for running EF Core migrations in the cloud
- ✅ Automated migrations in your CI/CD pipeline
- ✅ Handled migrations in AKS (Kubernetes Jobs)
- ✅ Learned how to rollback a bad migration
- ✅ Secured your DB connection with Managed Identity

**Time Required:** 1 hour  
**Prerequisites:** File 03 or 04 completed (SQL Database running)

---

## The Challenge: Migrations in the Cloud

In local development, you just run `dotnet ef database update`. But in Azure, you can't just "run a command" on a server manually.

### ❌ What NOT to do in Production
**Do not** use `context.Database.Migrate()` inside `Program.cs` for production.
- **Why?** If you scale to 5 instances, all 5 might try to migrate at the same time, causing corruption or locks. It also slows down app startup.

### ✅ What TO do
Run migrations as a **deployment step** before the app starts.

---

## Step 1: Generate a Migration Script (Idempotent)

Instead of running EF commands directly against Azure, generate a SQL script. This is safer and can be reviewed.

```powershell
# Install EF tool if you haven't
dotnet tool install --global dotnet-ef

# Generate an IDEMPOTENT script
# This script checks if a migration was already applied before running it
dotnet ef migrations script --idempotent --output ./publish/migrate.sql --project src/MyProject.Infrastructure --startup-project src/MyProject.Api
```

---

## Step 2: Running Migrations for App Service

### Option A: Azure DevOps / GitHub Actions (Recommended)

Add a step to your pipeline to run the SQL script against your Azure SQL Database.

```yaml
# GitHub Actions example
- name: Run SQL Migrations
  uses: azure/sql-action@v2
  with:
    connection-string: ${{ secrets.AZURE_SQL_CONNECTION_STRING }}
    path: './publish/migrate.sql'
```

### Option B: Azure CLI (Manual/Scripted)

```powershell
# Run a SQL file against Azure SQL
az sql db copy ... # (advanced)
# Better to use the sql-action in CI/CD or a local runner with access
```

---

## Step 3: Running Migrations for AKS (Kubernetes Job)

In AKS, the best way to migrate is using a **Kubernetes Job**. This runs a one-off container that applies the migration and then exits.

### 1. Create a Migration Dockerfile

You can use your existing app image, but call the EF tool, or just use a small image that runs the SQL script.

**Better approach:** Use a "Migration Container" that has the EF tools.

```dockerfile
# Dockerfile.migration
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet tool install --global dotnet-ef
ENV PATH="$PATH:/root/.dotnet/tools"
ENTRYPOINT ["dotnet", "ef", "database", "update", "--project", "MyProject.Infrastructure", "--startup-project", "MyProject.Api"]
```

### 2. Create the Kubernetes Job Manifest

```yaml
# k8s/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myapi-migration-{{ .Release.Revision }} # Unique name for each release
  namespace: myproject
spec:
  template:
    spec:
      containers:
      - name: migration
        image: myacr.azurecr.io/myapi-migration:v1.0
        envFrom:
        - secretRef:
            name: myapi-secrets
      restartPolicy: Never
  backoffLimit: 1 # Only try once
```

### 3. Integrate with Helm (Hooks)

If you use Helm, you can use **Hooks** to run this job automatically during `helm upgrade`.

```yaml
# myapi-chart/templates/migration-job.yaml
metadata:
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

---

## Step 4: Security — Passwordless SQL

Instead of putting passwords in connection strings, use **Managed Identity**.

### 1. Enable Managed Identity on App Service/AKS
(See File 07 for instructions)

### 2. Add the Identity as a User in SQL

```sql
-- Connect to your SQL DB as Admin
CREATE USER [your-app-service-name] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [your-app-service-name];
ALTER ROLE db_datawriter ADD MEMBER [your-app-service-name];
ALTER ROLE db_ddladmin ADD MEMBER [your-app-service-name]; -- Needed for migrations!
```

### 3. Update Connection String

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;Authentication=Active Directory Default;TrustServerCertificate=True;"
}
```

---

## Step 5: Rollback Strategy

What if a migration fails or breaks the app?

1. **Always backup before migration:** Azure SQL does automatic backups, but a "Point-in-time restore" takes time.
2. **EF Core Rollback:**
   ```powershell
   # Revert to a specific migration locally
   dotnet ef database update PreviousMigrationName
   
   # Generate a script to undo changes
   dotnet ef migrations script CurrentMigrationName PreviousMigrationName --output rollback.sql
   ```
3. **Best Practice:** Keep migrations "Additive" only. Avoid `DROP COLUMN` if possible, as it makes rolling back the app code impossible without restoring the DB.

---

## ✅ Migration Checklist

- [ ] Idempotent SQL script generated
- [ ] Pipeline runs migration BEFORE app deployment
- [ ] `db_ddladmin` permissions granted to the deployment identity
- [ ] Tested rollback script locally
- [ ] Verified migration logs in CI/CD output

---

> **Next Step:** Practical messaging with Service Bus → [13-SERVICE-BUS-MESSAGING-DOTNET.md](13-SERVICE-BUS-MESSAGING-DOTNET.md)
