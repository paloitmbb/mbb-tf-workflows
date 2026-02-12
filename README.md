# mbb-tf-workflows

**Reusable GitHub Actions workflows** for Terraform CI/CD with parallel security scanning, OIDC authentication, and plan attestation.

## Quick Start

```yaml
# .github/workflows/terraform.yml
name: Terraform CI
on: [push, pull_request]

permissions:
  contents: read
  security-events: write
  id-token: write
  pull-requests: write
  actions: read
  attestations: write

jobs:
  terraform-ci:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      terraform-version: '1.7.0'
      working-directory: './env/prod'
      environment: 'prod'
      terraform-var-file: 'terraform.tfvars'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Architecture

This repository contains **2 reusable workflows** that orchestrate **11 composite actions** from [paloitmbb/mbb-tf-actions](https://github.com/paloitmbb/mbb-tf-actions), enabling:

- ✅ **Parallel security scanning** (reduces CI time from ~9-12 mins to ~4-6 mins)
- ✅ **DRY principles** (workflows shared across multiple consumer repositories)
- ✅ **Zero static credentials** (OIDC-only authentication)
- ✅ **Defense-in-depth** (multi-layer security validation)

```
Consumer Repo (.github/workflows/terraform.yml)
    ↓ workflow_call
terraform-ci.yml / terraform-security.yml (this repo)
    ↓ uses composite actions
paloitmbb/mbb-tf-actions (implementation)
```

**Data Flow**: Caller repo → Reusable workflow → Composite actions → Terraform execution

## Workflows

### 1. `terraform-ci.yml` - Full CI Pipeline

Complete validation → security → plan flow with Azure OIDC authentication.

**Jobs**: validate → [tfsec, checkov, trivy] → aggregate → plan

**Key Features**:
- ✅ Terraform validation (fmt, validate, tflint)
- ✅ Parallel security scanning (tfsec, Checkov, Trivy)
- ✅ Azure OIDC authentication (zero static credentials)
- ✅ Plan generation with attestation signing
- ✅ SARIF upload to GitHub Security tab
- ✅ PR comments with plan summary

**Use Case**: Full CI pipeline for plan generation and comprehensive validation

### 2. `terraform-security.yml` - Security + Plan Preview

PR validation with security scanning and plan preview for code review.

**Jobs**: validate → [tfsec, checkov, trivy] → aggregate → plan

**Key Features**:
- ✅ Terraform validation (fmt, validate, tflint)
- ✅ Parallel security scanning (tfsec, Checkov, Trivy)
- ✅ Azure OIDC authentication for plan generation
- ✅ Plan preview with backend access (shows actual infrastructure changes)
- ✅ SARIF upload to GitHub Security tab

**Key Differences from terraform-ci.yml**:
- ❌ No plan artifact upload (plan displayed in job summary only)
- ❌ No attestation signing
- ❌ No PR comments with plan summary
- ❌ Different `continue-on-error` handling on aggregate job

**Use Case**: PR validation with comprehensive security checks and plan preview for reviewers (~6-8 mins)

## Key Inputs

### Common Inputs (Both Workflows)

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `terraform-version` | No | `1.7.0` | Terraform CLI version |
| `working-directory` | No | `.` | Directory containing Terraform code |
| `environment` | **Yes** | - | Target environment (dev/stage/prod) |
| `terraform-var-file` | No | `''` | Variable file for accurate scans (e.g., `terraform.tfvars`) |
| `enable-tflint` | No | `true` | Enable tflint linting |
| `enable-tfsec` | No | `true` | Enable tfsec scanning |
| `enable-checkov` | No | `true` | Enable Checkov scanning |
| `enable-trivy` | No | `true` | Enable Trivy scanning |
| `tfsec-severity` | No | `HIGH` | Fail threshold (CRITICAL/HIGH/MEDIUM/LOW) |
| `checkov-severity` | No | `HIGH` | Fail threshold (CRITICAL/HIGH/MEDIUM/LOW) |
| `trivy-severity` | No | `HIGH` | Fail threshold (CRITICAL/HIGH/MEDIUM/LOW) |

### Required Secrets (Both Workflows)

Both `terraform-ci.yml` and `terraform-security.yml` require Azure OIDC secrets for plan generation:

```yaml
AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Required Permissions

**terraform-ci.yml** (Full CI):
```yaml
permissions:
  contents: read          # Checkout code
  security-events: write  # Upload SARIF to Security tab
  id-token: write         # Azure OIDC authentication
  pull-requests: write    # Comment on PRs
  actions: read           # SARIF upload telemetry
  attestations: write     # Sign plan artifacts
```

**terraform-security.yml** (Security + Plan Preview):
```yaml
permissions:
  contents: read          # Checkout code
  security-events: write  # Upload SARIF to Security tab
  id-token: write         # Azure OIDC authentication
  actions: read           # SARIF upload telemetry
  # Note: No pull-requests or attestations permissions needed
```

**Why Different?**
- `terraform-ci.yml` needs `pull-requests: write` for PR comments and `attestations: write` for plan signing
- `terraform-security.yml` is lighter - only validates and generates plan preview (no artifacts or comments)

---


## Security Features

### 🔒 OIDC Authentication (Both Workflows)
Zero static credentials - Azure AD trusts GitHub's identity provider via Workload Identity Federation with short-lived tokens (1-hour expiration). Both workflows use Azure OIDC for plan generation with backend access.

**Setup Guide**: See [docs/AZURE-OIDC-QUICKSTART.md](docs/AZURE-OIDC-QUICKSTART.md) for complete Azure AD App Registration setup with step-by-step commands.

**Benefits**:
- No service principal secrets to rotate or leak
- Automatic credential lifecycle management
- Audit trail via Azure AD sign-in logs
- Scoped access per environment/repository

### 🛡️ Multi-Scanner Coverage

| Scanner | Purpose | Checks | SARIF Upload |
|---------|---------|--------|--------------|
| **tflint** | Code quality | Syntax, deprecated patterns, naming | ✅ |
| **tfsec** | Terraform security | 200+ cloud security rules | ✅ |
| **Checkov** | Policy compliance | 500+ policies (CIS, PCI-DSS, HIPAA) | ✅ |
| **Trivy** | Vulnerabilities | CVEs, secrets, misconfigurations | ✅ |

**SARIF Categories** (visible in GitHub Security tab):
- `tflint-code-quality`
- `tfsec-terraform-scan`  
- `checkov-iac-scan`
- `trivy-iac-scan`

**Parallel Execution**: All enabled security scanners run concurrently with `continue-on-error: true` to ensure complete coverage even if one scanner fails. Jobs are conditionally executed based on their `enable-<scanner>` input flags, skipping disabled scanners entirely for improved efficiency. The `security-aggregate` action determines the final pass/fail status.

### ✅ Variable File Integration

**Critical**: Pass `terraform-var-file` to improve scan coverage from **~70% to ~95%**

```yaml
terraform-var-file: 'terraform.tfvars'  # Relative to working-directory
```

**Why**: Scanners run with `-backend=false` (no credentials needed for security scans), then evaluate tfvars during scan to detect variable-dependent misconfigurations that would otherwise be missed.

**What Gets Scanned Better**:
- Conditional resources (`count = var.enabled ? 1 : 0`)
- Variable-driven security settings (`enable_https = var.secure_transfer`)
- Dynamic security group rules
- Compliance checks dependent on variable values

### ✍️ Plan Attestation (terraform-ci.yml only)

Sigstore-based cryptographic signing using GitHub's `actions/attest-build-provenance` action:
- **SLSA Level 3** compliance for supply chain security
- **Cryptographic signing** with ephemeral keys (no key management)
- **Immutable provenance** (workflow metadata, commit SHA, timestamp, runner environment)
- **Verification** via GitHub CLI: `gh attestation verify`

**Note**: `terraform-security.yml` generates plans but does **not** create attestations (disabled for lightweight PR checks without artifacts)

---

## Usage Examples

### Example 1: PR Security + Plan Preview (terraform-security.yml)

PR validation with security scans and plan preview:

```yaml
name: PR Security & Plan
on: pull_request

permissions:
  contents: read
  security-events: write
  id-token: write
  actions: read

jobs:
  security-and-plan:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-security.yml@main
    with:
      working-directory: './env/prod'
      environment: 'prod'
      terraform-var-file: 'terraform.tfvars'
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Example 2: Environment-Specific Severity

Stricter checks for production branches:

```yaml
jobs:
  terraform-ci:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
      checkov-severity: ${{ github.ref == 'refs/heads/main' && 'HIGH' || 'MEDIUM' }}
      tfsec-severity: ${{ github.ref == 'refs/heads/main' && 'HIGH' || 'MEDIUM' }}
```

### Example 3: Disable Specific Scanners

```yaml
jobs:
  terraform-ci:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@main
    with:
      environment: 'dev'
      enable-trivy: false  # Disable Trivy
      enable-checkov: true # Keep Checkov
```

---

## Best Practices

### 1. Version Pinning
✅ **Recommended**: Pin to stable tag
```yaml
uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@v1.2.0
```

❌ **Avoid in production**: Using `@main` (breaking changes possible)

### 2. Always Pass Variable Files
```yaml
terraform-var-file: 'terraform.tfvars'  # Critical for 95% scan coverage
```

### 3. Severity by Environment

| Environment | Severity | Rationale |
|-------------|----------|-----------|
| Development | MEDIUM | Catch issues early, allow flexibility |
| Staging | HIGH | Pre-production quality gate |
| Production | HIGH/CRITICAL | Block security issues |

### 4. Conditional Execution
Only run on infrastructure changes:
```yaml
on:
  push:
    paths: ['env/**', 'modules/**', '*.tf']
```

---

## Troubleshooting

### OIDC Authentication Failed
**Error**: `AADSTS700016: Application not found`

**Fix**: Verify Azure App Registration and federated credentials:
```bash
az ad app show --id $AZURE_CLIENT_ID
az ad app federated-credential list --id $AZURE_CLIENT_ID
```

### SARIF Upload Failed
**Error**: `Resource not accessible by integration`

**Fix**: Add `actions: read` permission:
```yaml
permissions:
  actions: read  # Required for SARIF telemetry
  security-events: write
```

### Variable File Not Found
**Error**: `Variable file does not exist`

**Fix**: Ensure path is relative to `working-directory`:
```yaml
working-directory: './env/prod'
terraform-var-file: 'terraform.tfvars'  # Must exist in ./env/prod/terraform.tfvars
```

---

## Documentation

### Setup Guides
- **[Azure OIDC Setup](docs/AZURE-OIDC-QUICKSTART.md)** - Complete guide for Azure AD App Registration with OIDC authentication
  - Step-by-step commands with variables
  - Federated credentials configuration
  - Storage account setup for Terraform state
  - RBAC role assignments
  - Troubleshooting common issues

### Usage Examples
- **[Example Workflows](example/README.md)** - Consumer repository workflow examples
  - CI workflow setup
  - PR security validation
  - Environment-specific configurations

### Related Documentation
- **[Composite Actions Reference](https://github.com/paloitmbb/mbb-tf-actions/blob/main/docs/REFERENCE.md)** - Detailed action specifications
- **[Caller Repository Setup](https://github.com/paloitmbb/mbb-tf-caller1/blob/main/docs/GITHUB-ACTIONS-SETUP.md)** - Full CI/CD pipeline setup

---

## Project Structure

```
.github/
  workflows/
    terraform-ci.yml         # Full CI with plan generation
    terraform-security.yml   # Security-only for PRs
example/
  README.md                  # Consumer usage guide
README.md                    # This file
```



