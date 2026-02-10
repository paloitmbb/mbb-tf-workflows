# Terraform Workflow Examples

Production-ready examples demonstrating how to use the **CDL-FirstOrg/Terraform-reusable-workflows** in your Terraform infrastructure projects.

## 📚 Available Examples

| Example File | Purpose | When to Use |
|--------------|---------|-------------|
| **[terraform-ci-example.yml](terraform-ci-example.yml)** | Complete CI pipeline (validate + security + plan) | Post-merge validation, manual testing, comprehensive checks |
| **[pr-security-check-example.yml](pr-security-check-example.yml)** | Lightweight PR security scanning | Pull request validation before code review |
| **[ENVIRONMENT-STRUCTURE-GUIDE.md](ENVIRONMENT-STRUCTURE-GUIDE.md)** | Environment organization patterns | Planning directory structure |

---

## 🎯 Quick Start

### Step 1: Choose Your Workflow Pattern

#### Option A: CI Pipeline (Recommended for Post-Merge)
**Use when**: You want comprehensive validation after code is merged to `main`/`develop`

**Copy**:
```bash
cp terraform-ci-example.yml .github/workflows/terraform-ci.yml
```

**Features**:
- ✅ Validates Terraform syntax and formatting
- ✅ Runs security scans (tfsec, Checkov, Trivy)
- ✅ Generates Terraform plan
- ✅ Uploads SARIF to GitHub Security tab
- ✅ Creates plan attestation
- ✅ Environment-aware (auto-detects dev/stage/prod)

---

#### Option B: PR Security Check (Recommended for Pull Requests)
**Use when**: You want fast security feedback on PRs before code review

**Copy**:
```bash
cp pr-security-check-example.yml .github/workflows/pr-security-check.yml
```

**Features**:
- ✅ Lightweight (security scans only, no plan)
- ✅ Fast feedback (3-5 minutes)
- ✅ Blocks PR merge if security issues found
- ✅ SARIF upload to Security tab
- ✅ PR comments with scan results
- ✅ Environment-aware based on target branch

---

### Step 2: Update Organization and Repository Names

```bash
# Replace placeholder with your actual organization name
sed -i 's|CDL-FirstOrg/Terraform-reusable-workflows|YOUR_ORG/YOUR_WORKFLOWS_REPO|g' .github/workflows/*.yml
```

**Example**: If your reusable workflows are in `acme-corp/github-workflows`:
```bash
sed -i 's|CDL-FirstOrg/Terraform-reusable-workflows|acme-corp/github-workflows|g' .github/workflows/*.yml
```

---

### Step 3: Configure Repository Secrets

Navigate to **Settings → Secrets and variables → Actions** and add:

#### Required Secrets (Azure OIDC)
```
AZURE_CLIENT_ID: abc-123-def-456
AZURE_TENANT_ID: xyz-789-abc-123
AZURE_SUBSCRIPTION_ID: sub-456-def-789
```

**How to get these values**: See [Azure OIDC Setup Guide](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure)

#### Optional Secrets
```
GH_TOKEN: ghp_xxxxxxxxxxxx (for private module access)
```

---

### Step 4: Configure GitHub Environments

Create environments for approval workflow:

**Via GitHub UI**: Settings → Environments → New environment

| Environment | Protection Rules | Deployment Branches |
|-------------|------------------|---------------------|
| **dev** | None (auto-deploy) | `develop`, `feature/*` |
| **stage** | Required reviewers: 1 | `develop`, `main` |
| **prod** | Required reviewers: 2 | `main`, tags `v*` |

**Example Protection Rule (Production)**:
```
Environment name: prod
Deployment branches: main, release/*, v*
Required reviewers: john@example.com, jane@example.com
Wait timer: 0 minutes
```

---

### Step 5: Organize Your Terraform Directory

The workflows expect this structure (see [ENVIRONMENT-STRUCTURE-GUIDE.md](ENVIRONMENT-STRUCTURE-GUIDE.md) for details):

```
your-terraform-repo/
├── .github/
│   └── workflows/
│       ├── terraform-ci.yml                 # Post-merge CI pipeline
│       └── pr-security-check.yml            # PR security validation
│
├── env/                                     # Environment-specific Terraform
│   ├── dev/
│   │   ├── backend.tf                       # Azure Storage backend config
│   │   ├── provider.tf                      # Azure provider with OIDC
│   │   ├── main.tf                          # Call to ../modules/*
│   │   ├── variables.tf                     # Input variables
│   │   ├── outputs.tf                       # Output values
│   │   └── terraform.tfvars                 # Dev variable values
│   │
│   ├── stage/
│   │   ├── backend.tf
│   │   ├── provider.tf
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars                 # Stage variable values
│   │
│   └── prod/
│       ├── backend.tf
│       ├── provider.tf
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars                 # Prod variable values
│
└── modules/                                 # Reusable Terraform modules
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── network/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 🔒 Security Configuration

### Configurable Severity Thresholds

All three security scanners (tfsec, Checkov, Trivy) support configurable severity:

**Available Severity Levels**:
- `CRITICAL` - Only critical vulnerabilities fail the workflow
- `HIGH` - High and critical vulnerabilities fail (default)
- `MEDIUM` - Medium, high, and critical vulnerabilities fail
- `LOW` - All vulnerabilities fail (strictest)

**Example 1: Use HIGH threshold for all scanners**
```yaml
uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
with:
  tfsec-severity: 'HIGH'
  checkov-severity: 'HIGH'
  trivy-severity: 'HIGH'
```

**Example 2: Different thresholds per scanner**
```yaml
uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
with:
  tfsec-severity: 'CRITICAL'     # Only fail on critical tfsec issues
  checkov-severity: 'HIGH'       # Fail on high/critical Checkov issues
  trivy-severity: 'MEDIUM'       # Fail on medium/high/critical Trivy issues
```

**Example 3: Manual dispatch with user-selectable severity**
```yaml
workflow_dispatch:
  inputs:
    severity:
      type: choice
      options: [CRITICAL, HIGH, MEDIUM, LOW]
      default: HIGH

# Then in workflow:
with:
  tfsec-severity: ${{ github.event.inputs.severity }}
  checkov-severity: ${{ github.event.inputs.severity }}
  trivy-severity: ${{ github.event.inputs.severity }}
```

---

### terraform-var-file Integration

**Why**: Security scanners analyze Terraform configurations STATICALLY (without runtime values). This causes false negatives when:
- Resource configurations depend on `var.xxx` values
- Conditional resources use variables (`count = var.enabled ? 1 : 0`)
- Dynamic blocks reference variables

**Solution**: Pass `terraform.tfvars` to security scanners so they can evaluate configurations with actual values.

**Impact**: Increases scan coverage from ~70% to ~95% by enabling scanners to analyze:
- Conditionally created resources
- Dynamically configured security groups
- Variable-driven compliance rules

**Example Configuration**:
```yaml
uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
with:
  terraform-var-file: 'terraform.tfvars'  # Relative to working-directory
  enable-tfsec: true
  enable-checkov: true
  enable-trivy: true
```

**Before (70% coverage)**:
```hcl
resource "azurerm_storage_account" "example" {
  # Scanned ✅
  name                     = var.storage_account_name
  
  # ⚠️ Skipped (dynamic, scanner can't evaluate)
  enable_https_traffic_only = var.enable_https
}
```

**After with terraform.tfvars (95% coverage)**:
```hcl
# terraform.tfvars
enable_https = true

# Now scanner evaluates with actual value:
# ✅ enable_https_traffic_only = true (scanned properly)
```

---

## 🚀 Usage Examples

### Example 1: Basic CI Pipeline

```yaml
name: Terraform CI

on:
  push:
    branches: [main, develop]

permissions:
  contents: read
  security-events: write
  id-token: write

jobs:
  terraform-ci:
    uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
    with:
      terraform-version: '1.7.0'
      working-directory: './env/dev'
      environment: 'dev'
      terraform-var-file: 'terraform.tfvars'
      enable-tfsec: true
      enable-checkov: true
      enable-trivy: true
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### Example 2: Environment-Aware Workflow

```yaml
name: Multi-Environment CI

on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, stage, prod]

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - name: Determine Environment
        id: set-env
        run: |
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            ENV="${{ github.event.inputs.environment }}"
          elif [ "${{ github.ref }}" == "refs/heads/main" ]; then
            ENV="prod"
          else
            ENV="stage"
          fi
          echo "environment=${ENV}" >> $GITHUB_OUTPUT
  
  terraform-ci:
    needs: prepare
    uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
    with:
      working-directory: ./env/${{ needs.prepare.outputs.environment }}
      environment: ${{ needs.prepare.outputs.environment }}
      terraform-var-file: 'terraform.tfvars'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### Example 3: PR Security Check with Configurable Severity

```yaml
name: PR Security Check

on:
  pull_request:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      severity:
        type: choice
        options: [LOW, MEDIUM, HIGH, CRITICAL]
        default: HIGH

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  security-scan:
    uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/reusable-tf-security.yml@main
    with:
      working-directory: './env/dev'
      terraform-var-file: 'terraform.tfvars'
      enable-tfsec: true
      tfsec-severity: ${{ github.event.inputs.severity || 'HIGH' }}
      enable-checkov: true
      checkov-severity: ${{ github.event.inputs.severity || 'HIGH' }}
      enable-trivy: true
      trivy-severity: ${{ github.event.inputs.severity || 'HIGH' }}
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

---

## 📊 Workflow Outputs

After workflow completion, these outputs are available for downstream jobs:

| Output | Description | Example Value |
|--------|-------------|---------------|
| `plan-exitcode` | Terraform plan exit code | `0` (no changes), `2` (changes detected) |
| `plan-artifact-name` | Name of uploaded plan artifact | `terraform-plan-dev-20240115-abc123` |
| `plan-hash` | SHA256 hash of plan file | `sha256:a1b2c3d4...` |
| `security-scan-passed` | Whether all security scans passed | `true` / `false` |

**Example: Use plan output in downstream job**
```yaml
jobs:
  terraform-ci:
    uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-pipeline.yml@main
    # ... configuration ...
  
  notify-changes:
    needs: terraform-ci
    runs-on: ubuntu-latest
    if: needs.terraform-ci.outputs.plan-exitcode == '2'
    steps:
      - name: Notify Team
        run: |
          echo "Infrastructure changes detected!"
          echo "Plan: ${{ needs.terraform-ci.outputs.plan-artifact-name }}"
```

---

## 🛠️ Customization Options

### Disable Specific Security Scanners

```yaml
with:
  enable-tflint: true      # Enable code quality checks
  enable-tfsec: true       # Enable tfsec security scan
  enable-checkov: false    # Disable Checkov
  enable-trivy: false      # Disable Trivy
```

### Custom Terraform Version

```yaml
with:
  terraform-version: '1.7.0'  # Pin to specific version
```

### Custom Runner

```yaml
with:
  runner-os: 'ubuntu-latest'  # Or self-hosted runner label
```

### Custom Timeout

```yaml
with:
  timeout-minutes: 120  # Increase for large infrastructures
```

---

## 🔍 Troubleshooting

### Issue 1: "trivy-severity is not defined in the referenced workflow"

**Cause**: Your reusable workflows repository doesn't have the `trivy-severity` parameter yet (remote main branch is outdated).

**Solution**:
```bash
# 1. Push reusable workflows FIRST
cd Terraform-reusable-workflows
git add .github/workflows/
git commit -m "feat: add trivy-severity configurable threshold"
git push origin main

# 2. Wait 1-2 minutes for GitHub indexing

# 3. Then push caller workflows
cd ../Terraform-caller1
git push origin main
```

---

### Issue 2: SARIF Upload Fails

**Symptoms**: `Error: Advanced Security must be enabled for this repository`

**Solution**:
```bash
# Enable GitHub Advanced Security
# Settings → Code security and analysis → GitHub Advanced Security → Enable
```

**Alternative**: Remove SARIF upload steps if you don't have Advanced Security license:
```yaml
# Comment out in your workflow:
# - name: Upload tfsec SARIF
#   uses: github/codeql-action/upload-sarif@v4
```

---

### Issue 3: Checkov Fails with "No Terraform files found"

**Cause**: `working-directory` doesn't contain `.tf` files, or modules aren't initialized.

**Solution**: Ensure your directory structure matches the expected pattern:
```
env/dev/
├── backend.tf        # At least one .tf file must exist
├── provider.tf
├── main.tf
└── terraform.tfvars
```

---

### Issue 4: Azure OIDC Authentication Fails

**Symptoms**: `Error: AADSTS7000215: Invalid client secret provided`

**Cause**: Using service principal secret instead of OIDC, or incorrect federated credential configuration.

**Solution**: Verify OIDC setup:
```bash
# 1. Check federated credential exists
az ad app federated-credential list \
  --id <APP_OBJECT_ID> \
  --query "[?subject=='repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main']"

# 2. Verify subject matches your repository exactly
# Subject format: repo:ORG/REPO:ref:refs/heads/BRANCH
# Example: repo:CDL-FirstOrg/Terraform-caller1:ref:refs/heads/main
```

**Guide**: [Azure OIDC Step-by-Step Setup](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure)

---

### Issue 5: terraform.tfvars Not Found

**Symptoms**: Security scan warnings about missing variables, reduced coverage

**Cause**: `terraform-var-file` path is incorrect (relative to `working-directory`)

**Solution**:
```yaml
# Correct ✅
with:
  working-directory: './env/dev'
  terraform-var-file: 'terraform.tfvars'  # Resolves to ./env/dev/terraform.tfvars

# Incorrect ❌
with:
  working-directory: './env/dev'
  terraform-var-file: './env/dev/terraform.tfvars'  # Too deep, file not found
```

---
