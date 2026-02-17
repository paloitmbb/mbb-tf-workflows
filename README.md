# mbb-tf-workflows

Reusable GitHub Actions workflows for Terraform that combine zero-trust authentication, parallel security scanning, and repeatable plan handling for any infrastructure repository.

> [!TIP]
> These workflows orchestrate the composite actions published in [paloitmbb/mbb-tf-actions](https://github.com/paloitmbb/mbb-tf-actions). Keep both repositories in sync to ensure inputs and outputs stay aligned.

## Highlights

- **Security-first pipeline** — tfsec + Trivy run in parallel, upload SARIF, and roll into a single gate.
- **Zero static creds** — Azure OIDC for Terraform state, GitHub HTTP backend for GitHub provider use cases.
- **Artifact provenance** — Optional plan attestation, hashing, and uploads for downstream promotion.
- **DRY by design** — Consumer repos call one workflow and inherit validation, scanning, and plan UX.

```
Caller Repo Workflow
    ↓ workflow_call
mbb-tf-workflows (terraform-ci / terraform-security / terraform-gh-ci)
    ↓ uses
paloitmbb/mbb-tf-actions composite steps
```

## Workflow Catalog

| Workflow | Backend | Typical Trigger | Primary Use |
| --- | --- | --- | --- |
| `.github/workflows/terraform-ci.yml` | Azure (`azurerm`) | Push / nightly | Full CI: validate → scan → plan, upload artifacts, attest plans, PR comments |
| `.github/workflows/terraform-security.yml` | Azure (`azurerm`) | Pull requests | Reviewer-focused: validation, security scans, real plan preview (no artifacts) |
| `.github/workflows/terraform-gh-ci.yml` | GitHub HTTP | Push / PR for GitHub provider repos | Same pattern but optimized for mbb-iac (HTTP backend, no Azure secrets) |

> [!NOTE]
> The `terraform-gh-ci.yml` workflow swaps Azure OIDC for GitHub HTTP backend auth so GitHub organization Terraform (mbb-iac) can reuse the same CI pattern without cloud credentials.

## Quick Start (terraform-ci)

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

### Inputs & Secrets (common)

| Key | Required | Notes |
| --- | --- | --- |
| `terraform-version` | No | Defaults to `1.7.0`. Keep in sync with caller repos. |
| `working-directory` | No | Root of Terraform configuration. |
| `environment` | **Yes** | Drives backend selection, naming, and PR messaging. |
| `terraform-var-file` | No | Relative to `working-directory`; dramatically improves scan accuracy. |
| `enable-tflint`, `enable-tfsec`, `enable-trivy` | No | Toggle scanners individually. |
| `tfsec-severity`, `trivy-severity` | No | Default `HIGH`; lower for dev, raise for prod. |
| `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | Secrets | Required for Azure-backed workflows. Not used by `terraform-gh-ci`. |
| `ORG_GITHUB_TOKEN` | Secret | Needed only when using `terraform-gh-ci.yml` against the GitHub HTTP backend. |

### Required Permissions

- `terraform-ci.yml`: `contents`, `security-events`, `id-token`, `pull-requests`, `actions`, `attestations`
- `terraform-security.yml`: same minus `pull-requests` and `attestations`
- `terraform-gh-ci.yml`: `contents`, `security-events`, `pull-requests`, `actions` (no Azure OIDC)

## Security Architecture

### OIDC Everywhere
- Azure workflows rely on Workload Identity Federation (see [docs/AZURE-OIDC-QUICKSTART.md](docs/AZURE-OIDC-QUICKSTART.md)).
- GitHub-provider workflow uses the HTTP backend with `TF_HTTP_USERNAME/password` exported by mbb-iac scripts.

### Parallel Scanners

| Stage | Tool | Output |
| --- | --- | --- |
| Validation | `terraform fmt`, `terraform validate`, `tflint` | Logs + status outputs |
| Security | tfsec | `tfsec-results.sarif` (category `tfsec-{env}`) |
| Security | Trivy | `trivy-results.sarif` (category `trivy-{env}`) |
| Aggregate | security-aggregate | `all-passed`, individual statuses |

- Scanners run with `continue-on-error: true`; the aggregate job determines failure.
- Passing `terraform-var-file` enables ~95% rule coverage because conditional resources resolve properly.

### Plan Integrity (terraform-ci)

1. `terraform-plan` writes `tfplan` and summary.
2. `tfplan-metadata` hashes the file and emits `plan-metadata.json`.
3. `actions/attest-build-provenance` signs the plan when `generate-attestation: true`.
4. `pr-comment-plan` posts the plan, scanner statuses, and exit codes to the PR.

## Usage Recipes

### PR Validation (terraform-security)

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

### GitHub Org Infra (terraform-gh-ci)

```yaml
jobs:
  gh-infra:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-gh-ci.yml@v1
    with:
      environment: dev
      working-directory: ./mbb-iac
    secrets:
      ORG_GITHUB_TOKEN: ${{ secrets.ORG_GITHUB_TOKEN }}
```

### Severity by Branch

```yaml
with:
  environment: ${{ startsWith(github.ref, 'refs/heads/main') && 'prod' || 'dev' }}
  tfsec-severity: ${{ startsWith(github.ref, 'refs/heads/main') && 'HIGH' || 'MEDIUM' }}
  trivy-severity: ${{ startsWith(github.ref, 'refs/heads/main') && 'HIGH' || 'MEDIUM' }}
```

## Best Practices

1. **Pin versions** — reference tagged releases (`@v1.x`) in consumer repos to avoid breaking changes.
2. **Pass tfvars** — use environment-specific var files for accurate scans and plans.
3. **Scope triggers** — limit `on.push.paths` to Terraform directories so docs-only changes skip CI.
4. **Mirror permissions** — grant only what each workflow needs; remove unused scopes on PR-only runs.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `AADSTS700016` when initializing backend | Azure AD app or federated credential missing | Re-run steps from [AZURE-OIDC-QUICKSTART](docs/AZURE-OIDC-QUICKSTART.md) and confirm `az ad app federated-credential list --id $AZURE_CLIENT_ID` shows entries. |
| SARIF upload fails with “resource not accessible” | Missing `actions: read` permission | Add `actions: read` to workflow permissions block. |
| `terraform-var-file` not found | Path not relative to `working-directory` | Place the file inside the environment folder or adjust the input path. |
| Plan step stuck waiting for backend | Backend not initialized (skipped `terraform-init` in caller) | Ensure `terraform-setup` → `terraform-init` run in the same job before plan. |

## Documentation & References

- [docs/AZURE-OIDC-QUICKSTART.md](docs/AZURE-OIDC-QUICKSTART.md) — detailed Azure identity setup.
- [example/README.md](example/README.md) — sample consumer repo wiring plus env detection tips.
- [paloitmbb/mbb-tf-actions docs](https://github.com/paloitmbb/mbb-tf-actions/tree/main/docs) — action-level inputs/outputs.
- [mbb-tf-caller1/docs](https://github.com/paloitmbb/mbb-tf-caller1/tree/main/docs) — reference implementation using these workflows.



