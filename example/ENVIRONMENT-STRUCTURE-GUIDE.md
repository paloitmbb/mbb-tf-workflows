# Environment Structure Guide

Concise patterns for structuring Terraform repositories that consume the **paloitmbb/mbb-tf-workflows** reusable workflows.

## Recommended Layout

```
your-terraform-repo/
├── .github/workflows/
│   ├── terraform-ci.yml          # Post-merge pipeline (calls terraform-ci.yml)
│   └── pr-security-check.yml     # PR validation (calls terraform-security.yml)
│
├── env/
│   ├── dev/
│   │   ├── backend.tf            # Azure Storage backend (dev)
│   │   ├── provider.tf           # AzureRM provider w/ OIDC
│   │   ├── main.tf               # Calls ../../modules/*
│   │   ├── variables.tf          # Input declarations
│   │   ├── outputs.tf            # Environment outputs
│   │   └── terraform.tfvars      # Non-sensitive values
│   ├── stage/
│   └── prod/                     # Same file set per environment
│
├── modules/                      # Reusable building blocks
│   ├── network/
│   ├── compute/
│   └── storage/
│
├── docs/
│   ├── GITHUB-ACTIONS-SETUP.md   # OIDC checklist
│   └── WORKFLOW-REFERENCE.md     # Workflow input matrix
└── README.md
```

## File Responsibilities

### `env/<env>/backend.tf`
Remote state per environment. Always set `use_oidc = true` so the backend authenticates with the same federated credential as the workflow.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state-dev"
    storage_account_name = "sttfstate001"
    container_name       = "tfstate"
    key                  = "dev/terraform.tfstate"
    use_oidc             = true
  }
}
```

### `env/<env>/provider.tf`
Centralises Terraform/AzureRM version pins and enables OIDC-based auth.

```hcl
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
  use_oidc        = true
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
  client_id       = var.client_id
}
```

### `env/<env>/main.tf`
Keeps environments thin by delegating to modules. Reference module outputs rather than duplicating resources.

```hcl
module "network" {
  source       = "../../modules/network"
  environment  = var.environment
  location     = var.location
  address_space = var.vnet_address_space
}

module "storage" {
  source                  = "../../modules/storage"
  environment             = var.environment
  location                = var.location
  account_tier            = var.storage_account_tier
  account_replication_type = var.storage_replication_type
}
```

### `variables.tf`, `outputs.tf`, `terraform.tfvars`
- **variables.tf**: only declarations (mark subscription/tenant/client as `sensitive`).
- **outputs.tf**: expose IDs the platform team needs (e.g., storage account name, subnet IDs).
- **terraform.tfvars**: commit non-secret values; inject secrets via `TF_VAR_*` environment variables or GitHub environment secrets.

## Modules Folder

Group reusable resources under `modules/` and keep inputs consistent across environments (e.g., `environment`, `location`, `tags`). This keeps `env/<env>/main.tf` focused on wiring rather than implementation.

## Workflow Integration

Pair the layout above with the shared workflows:

1. Detect the environment automatically.
2. Call the reusable workflow with the detected directory, `terraform.tfvars`, and OIDC secrets.

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.detect.outputs.environment }}
      working-directory: ${{ steps.detect.outputs.working-directory }}
    steps:
      - uses: actions/checkout@v4
      - id: detect
        uses: paloitmbb/mbb-tf-actions/actions/determine-environment@main
        with:
          fallback-environment: 'dev'

  terraform-ci:
    needs: prepare
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      working-directory: ${{ needs.prepare.outputs.working-directory }}
      environment: ${{ needs.prepare.outputs.environment }}
      terraform-var-file: 'terraform.tfvars'
      enable-tfsec: true
      enable-trivy: true
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Alternative Topologies

| Layout | When to use | Notes |
|--------|-------------|-------|
| **Single directory + tfvars** | Small proofs of concept | Keep `backend.tf` + `main.tf` at repo root, point workflows to `./terraform` and switch `terraform-var-file`. State isolation relies on different keys. |
| **Nested regions** | Multi-region orgs | e.g., `infrastructure/azure/<region>/env/dev`. Update `working-directory` accordingly. Deeper trees mean longer plan paths but mirror real org charts. |

## Best Practices

- Separate backend keys per environment (`dev/terraform.tfstate`, `stage/terraform.tfstate`, ...).
- Use OIDC everywhere (backend + provider) and never store client secrets.
- Keep environment layers declarative—modules own resource definitions.
- Commit non-sensitive tfvars; push secrets via repository/environment secrets.
- Reuse Terraform wrapper modules instead of copying resources between environments.
- Stick to consistent naming (`rg-<app>-<env>`, `st<app><env>###`) so automation can map resources easily.

## Related Docs

- `docs/AZURE-OIDC-QUICKSTART.md` – full OIDC setup.
- `example/terraform-ci-example.yml` – pipeline reference.
- `example/pr-security-check-example.yml` – PR validation pattern.

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
