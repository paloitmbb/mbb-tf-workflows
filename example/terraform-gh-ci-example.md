# Example: Terraform CI for GitHub Provider

Use this example when your Terraform project manages **GitHub resources** (repos, teams, org settings) and stores state in the **HTTP backend (GitHub Releases)**. The reusable workflow `terraform-gh-ci.yml` removes any Azure dependency while keeping the same validation + security posture.

## When to Use It
- GitHub organization IaC (e.g., `mbb-iac`)
- HTTP backend that relies on `TF_HTTP_PASSWORD`
- Need tfsec + Trivy scans plus a Terraform plan with PR comments
- Want GitHub-native authentication via a PAT (`ORG_GITHUB_TOKEN`)

## Prerequisites
- Repository structure similar to `mbb-iac` (YAML-driven Terraform root, HTTP backend config files per environment)
- GitHub Personal Access Token with `repo`, `read:org`, and `workflow` scopes stored as the secret `ORG_GITHUB_TOKEN`
- Backend `.tfvars` file (or inline `backend-config`) that points to your GitHub Release endpoints
- Optional: `terraform.tfvars` file per environment for richer scanner coverage

## Sample Workflow (`.github/workflows/github-provider-ci.yml`)
```yaml
name: GitHub Provider CI

on:
  pull_request:
    paths:
      - '**/*.tf'
      - 'data/**'
  push:
    branches: [main]
    paths:
      - '**/*.tf'
      - 'data/**'

permissions:
  contents: read
  security-events: write
  id-token: write
  pull-requests: write

jobs:
  github-provider-ci:
    uses: paloitmbb/mbb-tf-workflows/.github/workflows/terraform-gh-ci.yml@main
    with:
      terraform-version: '1.7.0'
      working-directory: '.'              # root module
      environment: 'dev'
      terraform-var-file: 'environments/dev/terraform.tfvars'
      backend-config-file: 'environments/dev/backend.tfvars'
      enable-tfsec: true
      enable-trivy: true
      tfsec-severity: 'HIGH'
      trivy-severity: 'HIGH'
      tfsec-continue-on-error: true
      trivy-continue-on-error: true
      aggregate-continue-on-error: false  # fail CI if scans fail
    secrets:
      ORG_GITHUB_TOKEN: ${{ secrets.ORG_GITHUB_TOKEN }}
```

## Inputs That Matter Most
| Input | Why it matters |
|-------|----------------|
| `working-directory` | Point to the folder containing `main.tf`. For `mbb-iac`, keep `.`. |
| `environment` | Used in summaries/plan comment; pick `dev`, `staging`, `production`, etc. |
| `terraform-var-file` | Pass real values to tfsec/Trivy for better coverage (optional but recommended). |
| `backend-config-file` or `backend-config` | Provide HTTP backend credentials (address, lock endpoints). Required for plan job. |
| `enable-pr-comment` | Defaults to `true`. Disable if you do not want plan output on PRs. |

## Secrets & Auth
- `ORG_GITHUB_TOKEN` is injected into both `GITHUB_TOKEN` and `TF_HTTP_PASSWORD` env vars inside the workflow.
- Keep the PAT scoped narrowly (ideally `repo`, `read:org`). Rotate periodically.

## Tips
- For multi-environment setups, pair this workflow with the `determine-environment` action to select the right backend/var files automatically.
- If you only want validation (no plan), set `enable-pr-comment: false` and skip `backend-config` inputs; the plan job will still run but without backend it will fail, so better keep backend config even for validation.
- Security scanners run in parallel and upload SARIF results to the repository Security tab. Use the severity inputs to tighten or relax enforcement.

Need an HTTP backend refresher? See `mbb-iac/HTTP_BACKEND_SETUP.md` or the comments inside `environments/<env>/backend.tfvars` in your GitHub provider repo.