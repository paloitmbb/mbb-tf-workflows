# Environment Structure Guide

This guide explains how to organize your Terraform infrastructure code for the **paloitmbb/mbb-tf-workflows** reusable workflows.

## Overview

The workflows expect a **multi-environment monorepo structure** where each environment (dev, stage, prod) has its own directory with environment-specific configurations, while shared logic lives in reusable modules.

## Recommended Directory Structure

```
your-terraform-repo/
├── .github/
│   └── workflows/
│       ├── terraform-ci.yml                 # Post-merge CI pipeline
│       └── pr-security-check.yml            # PR security validation
│
├── env/                                     # Environment-specific configurations
│   ├── dev/
│   │   ├── backend.tf                       # Azure Storage backend (dev)
│   │   ├── provider.tf                      # Azure provider with OIDC
│   │   ├── main.tf                          # Calls to ../modules/*
│   │   ├── variables.tf                     # Input variable declarations
│   │   ├── outputs.tf                       # Output value declarations
│   │   └── terraform.tfvars                 # Dev-specific variable values
│   │
│   ├── stage/
│   │   ├── backend.tf                       # Azure Storage backend (stage)
│   │   ├── provider.tf                      # Azure provider with OIDC
│   │   ├── main.tf                          # Calls to ../modules/*
│   │   ├── variables.tf                     # Input variable declarations
│   │   ├── outputs.tf                       # Output value declarations
│   │   └── terraform.tfvars                 # Stage-specific variable values
│   │
│   └── prod/
│       ├── backend.tf                       # Azure Storage backend (prod)
│       ├── provider.tf                      # Azure provider with OIDC
│       ├── main.tf                          # Calls to ../modules/*
│       ├── variables.tf                     # Input variable declarations
│       ├── outputs.tf                       # Output value declarations
│       └── terraform.tfvars                 # Prod-specific variable values
│
├── modules/                                 # Reusable Terraform modules
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── storage/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── docs/                                    # Documentation
│   ├── GITHUB-ACTIONS-SETUP.md             # Azure OIDC setup guide
│   └── WORKFLOW-REFERENCE.md               # Workflow parameter reference
│
└── README.md                                # Project overview

```

## File Purposes

### Environment Files (`env/{dev,stage,prod}/`)

#### `backend.tf`
Configures remote state storage. Uses Azure Storage Account with OIDC authentication.

```hcl
# env/dev/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state-dev"
    storage_account_name = "sttfstatedev12345"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
    
    # OIDC authentication (no static credentials)
    use_oidc = true
  }
}
```

#### `provider.tf`
Configures the Azure provider with OIDC authentication.

```hcl
# env/dev/provider.tf
terraform {
  required_version = ">= 1.7.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
}

provider "azurerm" {
  features {}
  
  # OIDC authentication (credentials from environment variables)
  use_oidc            = true
  subscription_id     = var.subscription_id
  tenant_id           = var.tenant_id
  client_id           = var.client_id
}
```

#### `main.tf`
Environment-specific infrastructure composition. Calls reusable modules with environment-specific values.

```hcl
# env/dev/main.tf
module "network" {
  source = "../../modules/network"
  
  environment         = var.environment
  location            = var.location
  address_space       = var.vnet_address_space
  subnet_prefixes     = var.subnet_prefixes
}

module "compute" {
  source = "../../modules/compute"
  
  environment         = var.environment
  location            = var.location
  subnet_id           = module.network.subnet_ids["app"]
  vm_size             = var.vm_size
}

module "storage" {
  source = "../../modules/storage"
  
  environment              = var.environment
  location                 = var.location
  account_tier             = var.storage_account_tier
  account_replication_type = var.storage_replication_type
}
```

#### `variables.tf`
Variable declarations (no values, just definitions).

```hcl
# env/dev/variables.tf
variable "environment" {
  description = "Environment name (dev, stage, prod)"
  type        = string
}

variable "location" {
  description = "Azure region for resources"
  type        = string
}

variable "subscription_id" {
  description = "Azure Subscription ID"
  type        = string
  sensitive   = true
}

variable "tenant_id" {
  description = "Azure Tenant ID"
  type        = string
  sensitive   = true
}

variable "client_id" {
  description = "Azure Client ID for OIDC"
  type        = string
  sensitive   = true
}

# ... other variables
```

#### `outputs.tf`
Output values for reference by other systems or environments.

```hcl
# env/dev/outputs.tf
output "network_id" {
  description = "Virtual Network ID"
  value       = module.network.vnet_id
}

output "subnet_ids" {
  description = "Map of subnet names to IDs"
  value       = module.network.subnet_ids
}

output "storage_account_name" {
  description = "Storage account name"
  value       = module.storage.storage_account_name
}
```

#### `terraform.tfvars`
Environment-specific variable values.

```hcl
# env/dev/terraform.tfvars
environment = "dev"
location    = "eastus"

# Network
vnet_address_space = ["10.0.0.0/16"]
subnet_prefixes    = {
  app = "10.0.1.0/24"
  db  = "10.0.2.0/24"
}

# Compute
vm_size = "Standard_B2s"

# Storage
storage_account_tier        = "Standard"
storage_replication_type    = "LRS"

# Azure OIDC (values from environment variables in CI/CD)
subscription_id = "00000000-0000-0000-0000-000000000000"  # Example only
tenant_id       = "00000000-0000-0000-0000-000000000000"  # Example only
client_id       = "00000000-0000-0000-0000-000000000000"  # Example only
```

**Note**: Sensitive values (subscription_id, tenant_id, client_id) are injected via environment variables in CI/CD workflows using GitHub secrets.

### Module Files (`modules/*/`)

Each module encapsulates reusable infrastructure components.

```hcl
# modules/network/main.tf
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.address_space
}

resource "azurerm_subnet" "subnets" {
  for_each = var.subnet_prefixes
  
  name                 = "snet-${each.key}-${var.environment}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [each.value]
}
```

## Alternative Structures

### Option 1: Workspace-Based (Single Directory)

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── dev.tfvars
├── stage.tfvars
└── prod.tfvars
```

**Workflow configuration**:
```yaml
with:
  working-directory: './terraform'
  environment: 'dev'
  terraform-var-file: 'dev.tfvars'
```

**Pros**: Simpler structure
**Cons**: Shared state, harder to separate environments, difficult to manage environment-specific backends

### Option 2: Nested Environments (Deep Hierarchy)

```
infrastructure/
└── azure/
    └── regions/
        └── eastus/
            └── environments/
                ├── dev/
                ├── stage/
                └── prod/
```

**Workflow configuration**:
```yaml
with:
  working-directory: './infrastructure/azure/regions/eastus/environments/dev'
  environment: 'dev'
```

**Pros**: Very organized for complex multi-region setups
**Cons**: Deep paths can be cumbersome, more complexity than needed for simple projects

## Workflow Integration

### Environment Detection

The workflows can auto-detect the target environment based on file paths or branch names:

```yaml
# .github/workflows/terraform-ci.yml
jobs:
  detect-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.detect.outputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - id: detect
        run: |
          # Detect from changed files
          if git diff --name-only origin/main | grep -q "^env/prod/"; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          elif git diff --name-only origin/main | grep -q "^env/stage/"; then
            echo "environment=stage" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi
```

### Dynamic Working Directory

```yaml
jobs:
  terraform-ci:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      working-directory: ./env/${{ needs.detect.outputs.environment }}
      environment: ${{ needs.detect.outputs.environment }}
```

## Best Practices

### 1. Separate Backend Configurations
Each environment should have its own backend configuration to prevent state collision.

✅ **Good**: Separate state files
```hcl
# env/dev/backend.tf
key = "dev.terraform.tfstate"

# env/prod/backend.tf
key = "prod.terraform.tfstate"
```

❌ **Bad**: Shared state file
```hcl
# All environments use the same key
key = "terraform.tfstate"
```

### 2. Use OIDC for Authentication
Never commit static credentials. Use OIDC with GitHub secrets.

✅ **Good**: OIDC authentication
```hcl
provider "azurerm" {
  use_oidc        = true
  subscription_id = var.subscription_id  # From environment variable in CI/CD
}
```

❌ **Bad**: Static credentials
```hcl
provider "azurerm" {
  client_secret = "super-secret-value"  # NEVER DO THIS
}
```

### 3. DRY Modules
Keep environment-specific code minimal. Most logic should be in reusable modules.

✅ **Good**: Thin environment layer
```hcl
# env/dev/main.tf (10 lines)
module "network" {
  source = "../../modules/network"
  environment = "dev"
}
```

❌ **Bad**: Duplicated code per environment
```hcl
# env/dev/main.tf (500 lines of duplicated resource definitions)
```

### 4. Version Control for tfvars
Commit non-sensitive tfvars to version control. Use GitHub secrets for sensitive values.

```hcl
# terraform.tfvars (safe to commit)
environment = "dev"
vm_size     = "Standard_B2s"

# Sensitive values injected via environment variables in CI/CD
# subscription_id from ${{ secrets.AZURE_SUBSCRIPTION_ID }}
# tenant_id from ${{ secrets.AZURE_TENANT_ID }}
# client_id from ${{ secrets.AZURE_CLIENT_ID }}
```

### 5. Consistent Naming
Use consistent naming patterns across environments.

```
resource_group_name: rg-{purpose}-{environment}
storage_account:     st{purpose}{environment}{random}
virtual_network:     vnet-{environment}
```

## Integration with mbb-tf-workflows

The recommended structure integrates seamlessly with the workflows:

```yaml
# .github/workflows/terraform-ci.yml
jobs:
  dev:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      working-directory: './env/dev'
      environment: 'dev'
      terraform-var-file: 'terraform.tfvars'
  
  stage:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      working-directory: './env/stage'
      environment: 'stage'
      terraform-var-file: 'terraform.tfvars'
  
  prod:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      working-directory: './env/prod'
      environment: 'prod'
      terraform-var-file: 'terraform.tfvars'
```

## Migration from Existing Structure

If you have an existing Terraform project, here's how to migrate:

### Step 1: Create Environment Directories
```bash
mkdir -p env/{dev,stage,prod}
```

### Step 2: Move Existing Code
```bash
# Assuming current code is for "dev"
cp *.tf env/dev/
```

### Step 3: Extract Modules
```bash
mkdir -p modules/compute modules/network modules/storage
# Move reusable resource definitions to modules
```

### Step 4: Create Environment-Specific Backends
```bash
# Update env/dev/backend.tf with dev-specific state key
# Update env/stage/backend.tf with stage-specific state key
# Update env/prod/backend.tf with prod-specific state key
```

### Step 5: Migrate State (if needed)
```bash
cd env/dev
terraform init -migrate-state
```

## See Also

- [Workflow Examples](README.md) - How to use the reusable workflows
- [Azure OIDC Setup Guide](../../mbb-tf-caller1/docs/GITHUB-ACTIONS-SETUP.md) - Setting up OIDC authentication
- [Terraform Best Practices](https://www.terraform-best-practices.com/) - General Terraform guidelines
