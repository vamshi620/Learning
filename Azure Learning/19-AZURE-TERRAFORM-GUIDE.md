# 19 – Azure Terraform Guide

---

## Table of Contents
1. [Why Terraform on Azure](#1-why-terraform-on-azure)
2. [Installation & Setup](#2-installation--setup)
3. [Terraform Core Concepts](#3-terraform-core-concepts)
4. [Azure Provider (azurerm) Deep Dive](#4-azure-provider-azurerm-deep-dive)
5. [State Management](#5-state-management)
6. [Terraform Modules](#6-terraform-modules)
7. [Terraform Workflows](#7-terraform-workflows)
8. [Terraform in CI/CD](#8-terraform-in-cicd)
9. [Best Practices](#9-best-practices)
10. [Complete Real-World Example](#10-complete-real-world-example)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Why Terraform on Azure

### IaC Tool Comparison

| Feature | Terraform | Bicep | ARM Templates | Pulumi |
|---|---|---|---|---|
| Language | HCL | Bicep DSL | JSON | TypeScript/Python/Go |
| Multi-cloud | ✅ Yes | ❌ Azure-only | ❌ Azure-only | ✅ Yes |
| State management | ✅ Built-in | ❌ No state | ❌ No state | ✅ Built-in |
| Drift detection | ✅ terraform plan | ❌ | ❌ | ✅ |
| Module registry | ✅ registry.terraform.io | ✅ Bicep registry | Limited | ✅ Pulumi registry |
| Learning curve | Medium | Low (Azure devs) | High (verbose JSON) | Medium |
| Maturity | Very high | High | Very high | Medium |

**Use Terraform when:** multi-cloud, existing Terraform expertise, need state management, using third-party providers.
**Use Bicep when:** Azure-only, existing ARM template knowledge, deep Azure integration (deployment stacks).

---

## 2. Installation & Setup

```bash
# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/terraform

# Ubuntu/Debian
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Windows (winget)
winget install HashiCorp.Terraform

# Verify
terraform version
```

### Authenticate to Azure

```bash
# Option 1: Interactive login (local development)
az login
az account set --subscription "<subscription-id>"

# Option 2: Service Principal with client secret
export ARM_CLIENT_ID="<client-id>"
export ARM_CLIENT_SECRET="<client-secret>"
export ARM_TENANT_ID="<tenant-id>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"

# Option 3: Service Principal with certificate
export ARM_CLIENT_ID="<client-id>"
export ARM_CLIENT_CERTIFICATE_PATH="/path/to/cert.pfx"
export ARM_TENANT_ID="<tenant-id>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"

# Option 4: Managed Identity (when running on Azure VMs/GitHub Actions with OIDC)
export ARM_USE_OIDC=true
export ARM_TENANT_ID="<tenant-id>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"
```

---

## 3. Terraform Core Concepts

### File Structure

```
my-infra/
├── main.tf          # main resources
├── variables.tf     # input variable declarations
├── outputs.tf       # output values
├── providers.tf     # provider + backend config
├── locals.tf        # local computed values
├── terraform.tfvars # variable values (gitignored for secrets)
└── .terraform/      # downloaded providers (gitignored)
```

### providers.tf

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.50"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }

  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "mytfstatestorage"
    container_name       = "tfstate"
    key                  = "prod/myapp.terraform.tfstate"
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
    virtual_machine {
      delete_os_disk_on_deletion     = true
      graceful_shutdown              = false
    }
  }
}
```

### variables.tf

```hcl
variable "location" {
  type        = string
  description = "Azure region for all resources."
  default     = "eastus"

  validation {
    condition     = contains(["eastus", "westus2", "westeurope", "northeurope"], var.location)
    error_message = "Location must be one of: eastus, westus2, westeurope, northeurope."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "app_name" {
  type        = string
  description = "Application name prefix for all resources."
}

variable "db_admin_password" {
  type        = string
  description = "SQL admin password."
  sensitive   = true  # never shown in logs or plan output
}

variable "node_pool_vm_size" {
  type    = string
  default = "Standard_D4s_v3"
}
```

### locals.tf

```hcl
locals {
  name_prefix = "${var.app_name}-${var.environment}"

  common_tags = {
    Environment     = var.environment
    Application     = var.app_name
    ManagedBy       = "terraform"
    Owner           = "platform-team@contoso.com"
    CostCenter      = "CC-1234"
  }

  # Resource name with random suffix to ensure uniqueness
  storage_account_name = lower(replace("${local.name_prefix}sa${random_id.suffix.hex}", "-", ""))
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

### outputs.tf

```hcl
output "resource_group_name" {
  value       = azurerm_resource_group.main.name
  description = "Name of the main resource group."
}

output "aks_cluster_name" {
  value       = azurerm_kubernetes_cluster.main.name
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.main.kube_config_raw
  sensitive = true
}

output "acr_login_server" {
  value = azurerm_container_registry.main.login_server
}
```

---

## 4. Azure Provider (azurerm) Deep Dive

### Resource Group

```hcl
resource "azurerm_resource_group" "main" {
  name     = "${local.name_prefix}-rg"
  location = var.location
  tags     = local.common_tags
}
```

### Virtual Network and Subnets

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${local.name_prefix}-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = local.common_tags
}

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "app" {
  name                 = "app-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]

  delegation {
    name = "appservice-delegation"
    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}

resource "azurerm_subnet" "private_endpoints" {
  name                                          = "private-endpoints-subnet"
  resource_group_name                           = azurerm_resource_group.main.name
  virtual_network_name                          = azurerm_virtual_network.main.name
  address_prefixes                              = ["10.0.3.0/24"]
  private_endpoint_network_policies             = "Disabled"
}
```

### NSG

```hcl
resource "azurerm_network_security_group" "aks" {
  name                = "${local.name_prefix}-aks-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "allow-https"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = local.common_tags
}

resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks.id
}
```

### Linux VM

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  name                = "${local.name_prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_D2s_v3"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.app.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 64
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = local.common_tags
}
```

### AKS Cluster

```hcl
resource "azurerm_user_assigned_identity" "aks" {
  name                = "${local.name_prefix}-aks-identity"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "${local.name_prefix}-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "${local.name_prefix}-aks"
  kubernetes_version  = "1.29"

  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    vnet_subnet_id      = azurerm_subnet.aks.id
    zones               = ["1", "2", "3"]
    os_disk_size_gb     = 128
    os_disk_type        = "Ephemeral"
    type                = "VirtualMachineScaleSets"

    upgrade_settings {
      max_surge = "33%"
    }
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
    service_cidr      = "172.16.0.0/16"
    dns_service_ip    = "172.16.0.10"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  azure_active_directory_role_based_access_control {
    managed            = true
    azure_rbac_enabled = true
  }

  key_vault_secrets_provider {
    secret_rotation_enabled = true
  }

  tags = local.common_tags
}

# User node pool for workloads
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = var.node_pool_vm_size
  node_count            = 2
  min_count             = 2
  max_count             = 20
  enable_auto_scaling   = true
  vnet_subnet_id        = azurerm_subnet.aks.id
  zones                 = ["1", "2", "3"]
  mode                  = "User"

  node_labels = {
    "workload-type" = "user"
  }

  tags = local.common_tags
}
```

### Key Vault

```hcl
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "main" {
  name                       = "${local.name_prefix}-kv"  # max 24 chars
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  enable_rbac_authorization  = true
  soft_delete_retention_days = 90
  purge_protection_enabled   = true

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }

  tags = local.common_tags
}

# Grant current user Key Vault admin (for Terraform to manage secrets)
resource "azurerm_role_assignment" "kv_terraform" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Administrator"
  principal_id         = data.azurerm_client_config.current.object_id
}
```

---

## 5. State Management

### Remote State on Azure

```bash
# Create backend storage (one-time setup)
az group create --name tfstate-rg --location eastus

az storage account create \
  --name mytfstatestorage \
  --resource-group tfstate-rg \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

az storage container create \
  --name tfstate \
  --account-name mytfstatestorage

# Enable versioning for state history
az storage account blob-service-properties update \
  --account-name mytfstatestorage \
  --enable-versioning true
```

### State Commands

```bash
# List all resources in state
terraform state list

# Show details of a resource
terraform state show azurerm_kubernetes_cluster.main

# Move resource (rename or restructure)
terraform state mv azurerm_subnet.old_name azurerm_subnet.new_name

# Remove resource from state (without destroying)
terraform state rm azurerm_resource_group.legacy

# Import existing resource into state
terraform import azurerm_resource_group.main /subscriptions/<sub>/resourceGroups/existing-rg

# Pull current state
terraform state pull > current.tfstate

# Force-unlock stuck state (use with caution)
terraform force-unlock <lock-id>
```

### Workspaces

```bash
# Create environments as workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Select workspace
terraform workspace select prod

# List workspaces (* = current)
terraform workspace list

# Use workspace name in resources
resource "azurerm_resource_group" "main" {
  name = "${var.app_name}-${terraform.workspace}-rg"
  ...
}
```

---

## 6. Terraform Modules

### Module Structure

```
modules/
└── aks-cluster/
    ├── main.tf        # AKS resources
    ├── variables.tf   # inputs
    ├── outputs.tf     # outputs
    └── README.md      # documentation
```

### Using a Module

```hcl
module "aks" {
  source = "./modules/aks-cluster"      # local module
  # source = "Azure/aks/azurerm"         # registry module
  # source = "git::https://github.com/myorg/tf-modules.git//aks?ref=v1.2.0"

  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  cluster_name        = "${local.name_prefix}-aks"
  node_count          = 3
  vm_size             = "Standard_D4s_v3"
  vnet_subnet_id      = azurerm_subnet.aks.id
  tags                = local.common_tags
}

output "kube_config" {
  value     = module.aks.kube_config_raw
  sensitive = true
}
```

---

## 7. Terraform Workflows

```bash
# Initialize project (download providers, configure backend)
terraform init

# Upgrade provider versions
terraform init -upgrade

# Format all .tf files
terraform fmt -recursive

# Validate syntax and configuration
terraform validate

# Preview changes (save plan to file for safer apply)
terraform plan -out=tfplan

# Apply saved plan
terraform apply tfplan

# Apply with auto-approval (CI/CD only)
terraform apply -auto-approve

# Target a specific resource
terraform plan -target=azurerm_kubernetes_cluster.main
terraform apply -target=azurerm_kubernetes_cluster.main

# Destroy all resources
terraform destroy

# Destroy a specific resource
terraform destroy -target=azurerm_virtual_machine.test

# Replace (re-create) a resource
terraform apply -replace=azurerm_linux_virtual_machine.app

# Refresh state without applying changes
terraform apply -refresh-only
```

---

## 8. Terraform in CI/CD

### GitHub Actions (OIDC – No Secrets Needed)

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write    # required for OIDC
  contents: read
  pull-requests: write

env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC: "true"

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.7.5

    - name: Az Login via OIDC
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      id: plan
      run: terraform plan -out=tfplan -no-color
      continue-on-error: true

    # Post plan output as PR comment
    - name: Comment Plan on PR
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v7
      with:
        script: |
          const output = `#### Terraform Plan 📖
          \`\`\`
          ${{ steps.plan.outputs.stdout }}
          \`\`\``;
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          });

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve tfplan
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  - group: terraform-secrets   # contains SERVICE_CONNECTION, BACKEND_*

stages:
- stage: Validate
  jobs:
  - job: TerraformValidate
    steps:
    - task: TerraformInstaller@1
      inputs:
        terraformVersion: 1.7.5

    - task: TerraformTaskV4@4
      displayName: Init
      inputs:
        provider: azurerm
        command: init
        backendServiceArm: $(SERVICE_CONNECTION)
        backendAzureRmResourceGroupName: tfstate-rg
        backendAzureRmStorageAccountName: mytfstatestorage
        backendAzureRmContainerName: tfstate
        backendAzureRmKey: prod/myapp.tfstate

    - task: TerraformTaskV4@4
      displayName: Validate
      inputs:
        provider: azurerm
        command: validate

    - task: TerraformTaskV4@4
      displayName: Plan
      inputs:
        provider: azurerm
        command: plan
        environmentServiceNameAzureRM: $(SERVICE_CONNECTION)
        commandOptions: -out=tfplan

- stage: Apply
  dependsOn: Validate
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: TerraformApply
    environment: production   # requires manual approval in Azure DevOps
    strategy:
      runOnce:
        deploy:
          steps:
          - task: TerraformTaskV4@4
            displayName: Apply
            inputs:
              provider: azurerm
              command: apply
              environmentServiceNameAzureRM: $(SERVICE_CONNECTION)
              commandOptions: tfplan
```

---

## 9. Best Practices

```
1. PIN VERSIONS
   - Pin Terraform version with required_version
   - Pin provider versions with ~> (pessimistic constraint)
   - Pin module versions with git tag refs

2. REMOTE STATE
   - Always use remote state in Azure Blob
   - Enable blob versioning for state history
   - Separate state files per environment per application

3. NAMING CONVENTIONS
   - Use locals for name_prefix: "${app_name}-${environment}"
   - Follow Azure naming rules (max lengths, allowed chars)
   - Use random_id suffix for globally unique resources

4. TAGS
   - Define common_tags in locals, apply to all resources
   - Include: Environment, Application, ManagedBy=terraform, Owner, CostCenter

5. SECRETS
   - Use sensitive = true for all secret variables
   - Never put secrets in terraform.tfvars committed to git
   - Use Key Vault data sources to fetch secrets at runtime

6. .gitignore
   .terraform/
   .terraform.lock.hcl     # DO commit this (provider version lock)
   *.tfstate
   *.tfstate.backup
   tfplan
   *.tfvars                 # if contains secrets

7. MODULES
   - Create modules for reusable patterns (AKS, VNet, SQL)
   - Keep modules single-purpose (one module = one concern)
   - Version modules with git tags (v1.0.0, v1.1.0)

8. SECURITY SCANNING
   checkov -d .
   tfsec .
   terrascan scan -t azure
```

---

## 10. Complete Real-World Example

### File Layout

```
infra/
├── providers.tf
├── main.tf           # resource group, log analytics
├── networking.tf     # vnet, subnets, nsg, private dns
├── aks.tf            # aks cluster, node pools
├── security.tf       # key vault, acr
├── variables.tf
├── outputs.tf
└── locals.tf
```

### networking.tf

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${local.name_prefix}-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = local.common_tags
}

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "private_endpoints" {
  name                                          = "pe-subnet"
  resource_group_name                           = azurerm_resource_group.main.name
  virtual_network_name                          = azurerm_virtual_network.main.name
  address_prefixes                              = ["10.0.4.0/24"]
  private_endpoint_network_policies             = "Disabled"
}

# Private DNS zone for Key Vault
resource "azurerm_private_dns_zone" "kv" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "kv" {
  name                  = "kv-dns-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.kv.name
  virtual_network_id    = azurerm_virtual_network.main.id
}
```

### security.tf

```hcl
resource "azurerm_container_registry" "main" {
  name                = "${replace(local.name_prefix, "-", "")}acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Premium"
  admin_enabled       = false

  network_rule_policy_bypass = "AzureServices"

  tags = local.common_tags
}

# Allow AKS to pull from ACR
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main.id
  skip_service_principal_aad_check = true
}

resource "azurerm_key_vault" "main" {
  name                       = "${local.name_prefix}-kv"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  enable_rbac_authorization  = true
  soft_delete_retention_days = 90
  purge_protection_enabled   = true
  tags                       = local.common_tags
}

# Private endpoint for Key Vault
resource "azurerm_private_endpoint" "kv" {
  name                = "${local.name_prefix}-kv-pe"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "kv-connection"
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "kv-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.kv.id]
  }
}
```

---

## 11. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Error acquiring the state lock` | Another process holds the lock | `terraform force-unlock <lock-id>` |
| `Inconsistent dependency lock file` | Provider version changed | `terraform init -upgrade` |
| `A resource with the ID already exists` | Resource exists but not in state | `terraform import <resource> <azure-id>` |
| `403 Forbidden` | Insufficient RBAC permissions | Check role assignments for SP/MI |
| `Error: creating Resource Group: already exists` | Terraform tries to create existing RG | Import it: `terraform import azurerm_resource_group.main /subscriptions/.../resourceGroups/name` |
| `The name 'xxx' is already taken` | Globally unique name conflict | Add random suffix with `random_id` |
| `Cycle error` | Circular resource dependency | Use `depends_on` explicitly or refactor |
| Provider version conflict | Two modules need different versions | Use `required_providers` version constraints carefully |

```bash
# Enable detailed debug logging
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply

# Test if backend is reachable
terraform init -backend-config="key=test.tfstate"

# Visualize dependency graph
terraform graph | dot -Tpng > graph.png
```
