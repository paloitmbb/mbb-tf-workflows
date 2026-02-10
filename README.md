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

The workflows delegate implementation to **composite actions** in [paloitmbb/mbb-tf-actions](https://github.com/paloitmbb/mbb-tf-actions), enabling parallel security scans and DRY principles:

```
Consumer Repo (.github/workflows/terraform.yml)
    ↓ workflow_call
terraform-ci.yml / terraform-security.yml (this repo)
    ↓ composite actions
paloitmbb/mbb-tf-actions (implementation)
```

**Parallel Security Scanning**: Reduces CI time from ~9-12 mins (sequential) to ~4-6 mins
```
validate → [tfsec, checkov, trivy] → aggregate → plan
           └─ Run in parallel ─┘
```

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

### 2. `terraform-security.yml` - Security-Only

Lightweight PR checks without backend access or plan generation.

**Jobs**: validate → [tfsec, checkov, trivy] → aggregate

**Key Difference**: No Azure credentials required, no plan job (~3-5 mins)

**Use Case**: Fast PR validation before code review

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

### Required Secrets (terraform-ci.yml only)

```yaml
AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Required Permissions

```yaml
permissions:
  contents: read          # Checkout code
  security-events: write  # Upload SARIF to Security tab
  id-token: write         # Azure OIDC (terraform-ci only)
  pull-requests: write    # Comment on PRs (terraform-ci only)
  actions: read           # SARIF upload telemetry
  attestations: write     # Sign plan artifacts (terraform-ci only)
```

---


## Security Features

### 🔒 OIDC Authentication (terraform-ci.yml)
Zero static credentials - Azure AD trusts GitHub's identity provider with short-lived tokens (1-hour expiration).

### 🛡️ Multi-Scanner Coverage

| Scanner | Purpose | Checks | SARIF Upload |
|---------|---------|--------|--------------|
| **tflint** | Code quality | Syntax, deprecated patterns, naming | ✅ |
| **tfsec** | Terraform security | 200+ cloud security rules | ✅ |
| **Checkov** | Policy compliance | 500+ policies (CIS, PCI-DSS, HIPAA) | ✅ |
| **Trivy** | Vulnerabilities | CVEs, secrets, misconfigurations | ✅ |

**SARIF Categories** (GitHub Security tab):
- `tflint-code-quality`
- `tfsec-terraform-scan`  
- `checkov-iac-scan`
- `trivy-iac-scan`

### ✅ Variable File Integration

**Critical**: Pass `terraform-var-file` to improve scan coverage from ~70% to ~95%

```yaml
terraform-var-file: 'terraform.tfvars'  # Relative to working-directory
```

**Why**: Scanners run with `-backend=false` (no credentials), then evaluate tfvars during scan to detect variable-dependent misconfigurations.

### ✍️ Plan Attestation (terraform-ci.yml)

Sigstore-based signing using GitHub's `actions/attest-build-provenance@v2.1.0`:
- SLSA Level 3 compliance
- Cryptographic signing with ephemeral keys
- Immutable provenance (workflow metadata, commit SHA, timestamp)

---

## Usage Examples

### Example 1: PR Security Check (terraform-security.yml)

Fast validation without Azure credentials:

```yaml
name: PR Security
on: pull_request

jobs:
  security:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-security.yml@main
    with:
      working-directory: './env/prod'
      environment: 'prod'
      terraform-var-file: 'terraform.tfvars'
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



