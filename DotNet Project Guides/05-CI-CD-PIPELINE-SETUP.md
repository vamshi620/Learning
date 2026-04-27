# CI/CD Pipeline Setup
## File 05: Automate Builds and Deployments

---

## What You'll Do in This File

By the end of this guide, you'll have:
- ✅ Automated build on every code push
- ✅ Automated deployment to App Service OR AKS
- ✅ Pull request validation (build + test before merge)
- ✅ Environment-specific deployments (dev → staging → production)

**Time Required:** 1-2 hours  
**Options:** Azure DevOps Pipelines OR GitHub Actions (both covered)

---

## What is CI/CD?

```
CI (Continuous Integration):          CD (Continuous Deployment):
──────────────────────────            ──────────────────────────
Developer pushes code                 After build passes...
   │                                     │
   ▼                                     ▼
Pipeline automatically:               Pipeline automatically:
├── Restores NuGet packages           ├── Deploys to Dev
├── Builds the project                ├── Runs smoke tests
├── Runs unit tests                   ├── Deploys to Staging (after approval)
└── Creates artifact/image            └── Deploys to Production (after approval)

Result: "Does the code compile         Result: "Get the code to users
and pass tests?"                       without manual work"
```

---

## Option A: Azure DevOps Pipeline

### A1: Pipeline for App Service

Create file `azure-pipelines.yml` in your repository root:

```yaml
# azure-pipelines.yml — Build and Deploy .NET App to App Service

trigger:
  branches:
    include:
      - main              # Run on every push to main
      - develop            # And develop branch

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'
  azureSubscription: 'MyAzureServiceConnection'    # Create this in Project Settings
  appServiceName: 'myproject-dev-app'
  resourceGroup: 'myproject-dev-rg'

# ─── STAGE 1: BUILD ───────────────────────────────────────────
stages:
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildJob
        displayName: 'Build .NET App'
        steps:
          # Install .NET SDK
          - task: UseDotNet@2
            displayName: 'Install .NET SDK'
            inputs:
              packageType: 'sdk'
              version: $(dotnetVersion)

          # Restore NuGet packages
          - task: DotNetCoreCLI@2
            displayName: 'Restore packages'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'

          # Build the project
          - task: DotNetCoreCLI@2
            displayName: 'Build project'
            inputs:
              command: 'build'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          # Run tests
          - task: DotNetCoreCLI@2
            displayName: 'Run tests'
            inputs:
              command: 'test'
              arguments: '--configuration $(buildConfiguration) --no-build --logger trx'
              projects: '**/*Tests*.csproj'

          # Publish (create deployment package)
          - task: DotNetCoreCLI@2
            displayName: 'Publish'
            inputs:
              command: 'publish'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
              publishWebProjects: true
              zipAfterPublish: true

          # Upload artifact for deployment stage
          - task: PublishBuildArtifacts@1
            displayName: 'Upload artifact'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'app'

  # ─── STAGE 2: DEPLOY TO DEV ─────────────────────────────────
  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToDev
        displayName: 'Deploy to App Service (Dev)'
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to App Service'
                  inputs:
                    azureSubscription: $(azureSubscription)
                    appType: 'webAppLinux'
                    appName: $(appServiceName)
                    package: '$(Pipeline.Workspace)/app/**/*.zip'
```

### A2: Pipeline for AKS

```yaml
# azure-pipelines-aks.yml — Build Docker Image and Deploy to AKS

trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  acrName: 'myprojectdevacr'
  imageName: 'myapi'
  aksCluster: 'myproject-dev-aks'
  resourceGroup: 'myproject-dev-rg'
  k8sNamespace: 'myproject'
  azureSubscription: 'MyAzureServiceConnection'

stages:
  # ─── STAGE 1: BUILD AND PUSH DOCKER IMAGE ───────────────────
  - stage: Build
    displayName: 'Build & Push Image'
    jobs:
      - job: BuildImage
        displayName: 'Build Docker Image'
        steps:
          # Run tests first
          - task: DotNetCoreCLI@2
            displayName: 'Run tests'
            inputs:
              command: 'test'
              projects: '**/*Tests*.csproj'

          # Build and push Docker image to ACR
          - task: Docker@2
            displayName: 'Build and Push to ACR'
            inputs:
              containerRegistry: $(acrName)   # Service connection to ACR
              repository: $(imageName)
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

  # ─── STAGE 2: DEPLOY TO AKS ─────────────────────────────────
  - stage: Deploy
    displayName: 'Deploy to AKS'
    dependsOn: Build
    jobs:
      - deployment: DeployToAKS
        displayName: 'Deploy to AKS'
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                # Update the image tag in deployment manifest
                - task: KubernetesManifest@1
                  displayName: 'Deploy to AKS'
                  inputs:
                    action: 'deploy'
                    connectionType: 'azureResourceManager'
                    azureSubscriptionConnection: $(azureSubscription)
                    azureResourceGroup: $(resourceGroup)
                    kubernetesCluster: $(aksCluster)
                    namespace: $(k8sNamespace)
                    manifests: |
                      k8s/deployment.yaml
                      k8s/service.yaml
                    containers: |
                      $(acrName).azurecr.io/$(imageName):$(Build.BuildId)
```

### A3: Set Up Azure DevOps Service Connection

```
1. Go to Azure DevOps → Project Settings → Service Connections
2. Click "New service connection" → "Azure Resource Manager"
3. Choose "Service principal (automatic)"
4. Select your subscription
5. Name it: "MyAzureServiceConnection"
6. Grant access to all pipelines ✅
```

---

## Option B: GitHub Actions

### B1: GitHub Actions for App Service

Create file `.github/workflows/deploy-appservice.yml`:

```yaml
# .github/workflows/deploy-appservice.yml

name: Build and Deploy to App Service

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]     # Build on PRs but don't deploy

env:
  DOTNET_VERSION: '8.0.x'
  APP_NAME: 'myproject-dev-app'
  RESOURCE_GROUP: 'myproject-dev-rg'

jobs:
  # ─── JOB 1: BUILD AND TEST ──────────────────────────────────
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --verbosity normal

      - name: Publish
        run: dotnet publish -c Release -o ./publish

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app
          path: ./publish

  # ─── JOB 2: DEPLOY (only on main branch push) ──────────────
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: app
          path: ./publish

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.APP_NAME }}
          package: ./publish
```

### B2: GitHub Actions for AKS

Create file `.github/workflows/deploy-aks.yml`:

```yaml
# .github/workflows/deploy-aks.yml

name: Build and Deploy to AKS

on:
  push:
    branches: [main]

env:
  ACR_NAME: 'myprojectdevacr'
  IMAGE_NAME: 'myapi'
  AKS_CLUSTER: 'myproject-dev-aks'
  RESOURCE_GROUP: 'myproject-dev-rg'
  K8S_NAMESPACE: 'myproject'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Run tests
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Test
        run: dotnet test --verbosity normal

      # Login to Azure
      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # Build and push Docker image to ACR
      - name: Login to ACR
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and push image
        run: |
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}

      # Deploy to AKS
      - name: Set AKS context
        uses: azure/aks-set-context@v4
        with:
          resource-group: ${{ env.RESOURCE_GROUP }}
          cluster-name: ${{ env.AKS_CLUSTER }}

      - name: Deploy to AKS
        run: |
          kubectl set image deployment/myapi -n ${{ env.K8S_NAMESPACE }} myapi=${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl rollout status deployment/myapi -n ${{ env.K8S_NAMESPACE }} --timeout=300s
```

### B3: Set Up GitHub Secrets

```
1. Go to your GitHub repo → Settings → Secrets and variables → Actions
2. Add these secrets:
   - AZURE_CLIENT_ID     (from App Registration)
   - AZURE_TENANT_ID     (from Entra ID)
   - AZURE_SUBSCRIPTION_ID (from az account show)
```

Create a GitHub OIDC connection (recommended over client secrets):

```powershell
# Create App Registration for GitHub
az ad app create --display-name "github-actions-deploy"

# Get the App ID
$APP_ID = az ad app list --display-name "github-actions-deploy" --query [0].appId -o tsv

# Create federated credential for GitHub
az ad app federated-credential create `
  --id $APP_ID `
  --parameters '{
    \"name\": \"github-main-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }'

# Create Service Principal and assign role
az ad sp create --id $APP_ID
az role assignment create `
  --assignee $APP_ID `
  --role Contributor `
  --scope "/subscriptions/<sub-id>/resourceGroups/myproject-dev-rg"
```

---

## Pull Request Validation

Both Azure DevOps and GitHub Actions automatically build and test on pull requests. This means:

```
Developer creates PR:
1. Pipeline runs: restore → build → test
2. PR shows ✅ or ❌ based on result
3. Team reviews code + sees test results
4. PR merged → triggers deployment pipeline
```

---

## ✅ CI/CD Checklist

- [ ] Pipeline file created in repository
- [ ] Service connection / secrets configured
- [ ] Build stage: restore, build, test, publish
- [ ] Deploy stage: pushes to App Service or AKS
- [ ] PR validation: builds and tests on every PR
- [ ] Team notified on build failures

---

> **Next Step:** Set up monitoring → [06-MONITORING-AND-DEBUGGING.md](06-MONITORING-AND-DEBUGGING.md)
