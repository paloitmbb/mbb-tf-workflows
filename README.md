# mbb-tf-workflows

Reusable GitHub Actions workflows for Terraform that combine zero-trust authentication, parallel security scanning, and repeatable plan handling for any infrastructure repository.

## Highlights

- 🛡️ **Security-first pipeline** — `tfsec` + `Trivy` run in parallel, upload SARIF, and roll into a single gate.
- 🔐 **Zero static creds** — Azure OIDC for Terraform state (including GitHub provider use cases).
- 📦 **Artifact provenance** — Optional plan attestation, hashing, and uploads for downstream promotion.
- ♻️ **DRY by design** — Consumer repos call one workflow and inherit validation, scanning, and plan UX.

## Workflow Catalog

| Workflow | Backend | Typical Trigger | Primary Use |
| :--- | :--- | :--- | :--- |
| [**terraform-ci**](.github/workflows/terraform-ci.yml) | Azure (`azurerm`) | Push / nightly | Full CI: validate → scan → plan, upload artifacts, attest plans, PR comments |
| [**terraform-security**](.github/workflows/terraform-security.yml) | Azure (`azurerm`) | Pull requests | Reviewer-focused: validation, security scans, real plan preview (no artifacts) |
| [**terraform-ci-github**](.github/workflows/terraform-ci-github.yml) | Azure (`azurerm`) | Push / PR | Optimized for GitHub organization Terraform (mbb-iac), using Azure storage backend (requires Azure secrets) |

> [!NOTE]
> These workflows orchestrate the composite actions published in [paloitmbb/mbb-tf-actions](https://github.com/paloitmbb/mbb-tf-actions). Keep both repositories in sync to ensure inputs and outputs stay aligned.

## Quick Start (terraform-ci)

To set up a full CI pipeline that validates, scans, and plans your Terraform configuration:

```yaml
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
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci.yml@v1
    with:
      terraform-version: '1.7.0'
      working-directory: ./env/dev
      environment: dev
      terraform-var-file: terraform.tfvars
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Inputs & Configuration

Most workflows share these common inputs:

| Key | Required | Description |
| :--- | :---: | :--- |
| `terraform-version` | No | Terraform version to use (default: `1.7.0`). |
| `working-directory` | No | Root of Terraform configuration. |
| `environment` | **Yes** | Drives backend selection, naming, and PR messaging. |
| `terraform-var-file` | No | Path relative to `working-directory`. Passing this dramatically improves scan accuracy. |
| `enable-tflint`, `enable-tfsec`, `enable-trivy` | No | Toggle individual scanners. |
| `tfsec-severity`, `trivy-severity` | No | Minimum severity to report (default `HIGH`). |

### Secrets

| Secret | Required by | Description |
| :--- | :--- | :--- |
| `AZURE_CLIENT_ID` | `terraform-ci`, `terraform-security` | The Client ID of the Azure AD application. |
| `AZURE_TENANT_ID` | `terraform-ci`, `terraform-security` | The Tenant ID of the Azure AD tenant. |
| `AZURE_SUBSCRIPTION_ID` | `terraform-ci`, `terraform-security` | The Subscription ID where resources are deployed. |
| `ARM_CLIENT_ID` | `terraform-ci-github` | The Client ID of the Azure AD application (for OIDC). |
| `ARM_TENANT_ID` | `terraform-ci-github` | The Tenant ID of the Azure AD tenant (for OIDC). |
| `ARM_SUBSCRIPTION_ID` | `terraform-ci-github` | The Subscription ID where resources are deployed (for OIDC). |
| `ORG_GITHUB_TOKEN` | `terraform-ci-github` | Token for GitHub provider authentication. |

## Usage Recipes

### PR Validation (terraform-security)

Ideal for faster feedback loops on Pull Requests without generating artifacts.

```yaml
name: PR Checks
on: pull_request

permissions:
  contents: read
  security-events: write
  id-token: write
  actions: read

jobs:
  security:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-security.yml@v1
    with:
      working-directory: ./env/prod
      environment: prod
      terraform-var-file: terraform.tfvars
```

### GitHub Org Infra (terraform-ci-github)

For repositories managing GitHub resources using the GitHub provider and Azure storage backend.

```yaml
jobs:
  gh-infra:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-ci-github.yml@v1
    with:
      environment: dev
      working-directory: ./mbb-iac
    secrets:
      ORG_GITHUB_TOKEN: ${{ secrets.ORG_GITHUB_TOKEN }}
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
```

### Dynamic Environments

Configure environment-specific inputs based on the branch name.

```yaml
with:
  environment: ${{ startsWith(github.ref, 'refs/heads/main') && 'prod' || 'dev' }}
  tfsec-severity: ${{ startsWith(github.ref, 'refs/heads/main') && 'HIGH' || 'MEDIUM' }}
```

## Security Architecture

### OIDC Authentication
Azure workflows leverage Workload Identity Federation, eliminating the need for long-lived secrets.
👉 [**Azure OIDC Setup Guide**](docs/AZURE-OIDC-QUICKSTART.md)

### Parallel Scanning
Security is enforced by running multiple scanners concurrently. The pipeline aggregates results and fails only if critical issues are found.

1. **Validation**: `terraform fmt`, `terraform validate`, `tflint`
2. **Security**: `tfsec`, `Trivy` (SARIF output)
3. **Aggregation**: Pass/fail determination based on severity thresholds.

### Plan Integrity
For the `terraform-ci` workflow:
1. `terraform-plan` generates the plan and summary.
2. `tfplan-metadata` hashes the plan for verification.
3. `actions/attest-build-provenance` signs the plan if enabled.
4. Results are posted to the Pull Request for review.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| :--- | :--- | :--- |
| `AADSTS700016` error | Missing federated credential | Check `az ad app federated-credential list` and review [OIDC guide](docs/AZURE-OIDC-QUICKSTART.md). |
| SARIF upload failure | Missing permissions | Ensure `actions: read` and `security-events: write` are in your workflow permissions. |
| `terraform-var-file` not found | Incorrect path | Verify the path is relative to `working-directory`. |
| Plan step hangs | Backend not initialized | Confirm `terraform-setup` and `terraform-init` run before the plan step. |

## Documentation

- [**Azure OIDC Quickstart**](docs/AZURE-OIDC-QUICKSTART.md)
- [**Example Consumer Repo**](example/README.md)
- [**Action Reference**](https://github.com/paloitmbb/mbb-tf-actions/tree/main/docs)
