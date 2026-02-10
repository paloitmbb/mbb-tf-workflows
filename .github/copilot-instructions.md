# Terraform Reusable Workflows - AI Agent Instructions

## Project Overview

This repository provides enterprise-grade **reusable GitHub Actions workflows** for Terraform CI/CD with multi-layer security scanning, OIDC authentication, and plan attestation. These workflows are consumed by infrastructure repositories via `workflow_call` events.

**Core Architecture**: Modular workflows that delegate implementation to composite actions in `CDL-FirstOrg/mbb-tf-actions` repository, enabling parallel security scanning and DRY principles.

## Critical Architecture Patterns

### 1. Workflow Dependency Chain
```
Consumer Repo (.github/workflows/terraform.yml)
    ↓ uses: workflow_call
terraform-ci.yml OR terraform-security.yml (this repo)
    ↓ uses: composite actions
CDL-FirstOrg/mbb-tf-actions/actions/*/action.yml
```

**Key Principle**: Workflows in this repo are **orchestrators**, not implementers. They coordinate jobs/steps but delegate actual work to composite actions. When modifying logic, check if changes belong in `mbb-tf-actions` repo instead.

### 2. Parallel Security Scanning Pattern

Both workflows use **parallel jobs** for security scanners (tfsec, checkov, trivy), followed by an **aggregator job** that collects results:

```yaml
jobs:
  tfsec-scan:     # Runs in parallel
  checkov-scan:   # Runs in parallel  
  trivy-scan:     # Runs in parallel
  aggregate-security:  # Waits for all, uses needs[]
    needs: [tfsec-scan, checkov-scan, trivy-scan]
```

**Why**: Reduces CI time from ~9-12 mins (sequential) to ~4-6 mins. Each scan job uses `continue-on-error: true` and outputs `outcome`, which the aggregate job evaluates.

### 3. Composite Action Usage Convention

Every step that requires Terraform setup uses the **common-setup pattern**:

```yaml
- name: Common Setup
  uses: CDL-FirstOrg/mbb-tf-actions/actions/terraform-setup@main
  with:
    terraform-version: ${{ inputs.terraform-version }}
```

This action handles: checkout, Terraform installation, git config, and sets common outputs. **Never duplicate this logic** in workflows.

## Key Workflow Files

### `.github/workflows/terraform-ci.yml` (Full CI Pipeline)
- **Purpose**: Complete validation → security → plan flow with Azure OIDC
- **Jobs**: validate → [tfsec, checkov, trivy] → aggregate → plan
- **Unique Features**: 
  - Azure OIDC authentication for backend access
  - Plan generation with attestation signing
  - PR comment with plan summary
  - Uploads plan artifact with provenance

**When to modify**: Adding/removing security scanners, changing job dependencies, updating plan generation logic

### `.github/workflows/terraform-security.yml` (Security-Only)
- **Purpose**: Lightweight PR checks without backend access
- **Jobs**: validate → [tfsec, checkov, trivy] → aggregate
- **Key Difference**: No Azure credentials required, no plan generation
- **Use Case**: Fast PR validation before code review (~3-5 mins)

**When to modify**: Same scanner changes as CI, but remember it skips plan job

## Project-Specific Conventions

### Variable File Integration Pattern
Security scanners receive `terraform-var-file` input to evaluate actual resource values (not just variable declarations):

```yaml
terraform-var-file: 'terraform.tfvars'  # MUST be relative to working-directory
```

**Critical**: Scanners run with `terraform init -backend=false` to avoid backend credentials, then use tfvars during scan. Coverage improves from ~70% to ~95% with tfvars.

### Backend Toggle Pattern
```yaml
# Validation/Security: Backend disabled (no credentials needed)
enable-backend: 'false'

# Plan: Backend enabled (requires OIDC authentication first)
enable-backend: 'true'
```

**Why**: Security scans shouldn't require Azure access; plan generation needs real state for accuracy.

### Exit Code Workaround in Plan Job
The plan job includes **manual exit code detection** because Terraform's `-detailed-exitcode` has known bugs:

```bash
# Workaround: Parse plan output to detect changes
if echo "$PLAN_OUTPUT" | grep -q "Plan: [1-9][0-9]* to add"; then
  EXITCODE=2  # Override terraform exit code
fi
```

**Don't remove this logic** without verifying Terraform CLI fix (as of v1.7.0, still needed).

### Aggregate Pattern for Multi-Scanner Results

Security aggregate job uses composite action `security-aggregate@main`:

```yaml
with:
  tfsec-enabled: ${{ inputs.enable-tfsec }}
  tfsec-outcome: ${{ needs.tfsec-scan.outputs.outcome || 'skipped' }}
  # ... same for checkov, trivy
```

Outputs `all-passed: 'true'/'false'` based on enabled scanners. **Always use `|| 'skipped'`** to handle disabled scanners gracefully.

## Required Permissions

Consumer workflows MUST declare these permissions (reusable workflows inherit them):

```yaml
permissions:
  contents: read          # Checkout code
  security-events: write  # Upload SARIF to Security tab
  id-token: write         # Azure OIDC authentication
  pull-requests: write    # Comment on PRs
  actions: read           # SARIF upload telemetry requirement
  attestations: write     # Sign plan artifacts (terraform-ci only)
```

**Common Issue**: Missing `actions: read` causes cryptic SARIF upload errors.

## Testing Changes

### Local Validation (Limited)
```bash
# Validate workflow syntax
actionlint .github/workflows/*.yml

# Test composite action calls (requires local action repos)
act -W .github/workflows/terraform-ci.yml
```

**Limitation**: Cannot test actual workflow_call without consumer repo.

### Integration Testing Pattern
1. Create test branch in this repo
2. Update consumer repo to use: `uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-ci.yml@test-branch`
3. Trigger workflow in consumer repo
4. Verify outputs and SARIF uploads in GitHub Security tab

**Critical**: Test both workflows (terraform-ci.yml AND terraform-security.yml) before merging.

## Versioning and Release Strategy

**Current Practice**: Consumers use `@main` (seen in README examples)
**Recommended Pattern**: Use semantic versioning tags

```yaml
# Production usage (recommended)
uses: CDL-FirstOrg/Terraform-reusable-workflows/.github/workflows/terraform-ci.yml@v1.2.0

# Development usage (avoid in prod)
uses: ...@main  # Breaking changes possible
```

**When releasing**: Tag commits with `vX.Y.Z`, update README.md version history table.

## Common Pitfalls

1. **Don't parallelize scanner jobs unnecessarily**: They're already parallel within each workflow. Don't create matrix strategy over scanner names.

2. **SARIF categories are critical**: Each scanner MUST use unique category for GitHub Security tab:
   - tfsec: `tfsec-terraform-scan`  
   - Checkov: `checkov-iac-scan`
   - Trivy: `trivy-iac-scan`

3. **Attestation only on plan changes**: `if: steps.plan.outputs.exitcode == '2'` prevents empty attestations.

4. **Environment input is NOT optional**: Despite `required: true`, workflows don't enforce this. Consumers will fail if omitted.

## External Dependencies

- **Composite Actions**: `CDL-FirstOrg/mbb-tf-actions` (implementation details)
- **Third-party Actions**: Pinned with commit SHAs in README.md table (v4.1.1, etc.)
- **GitHub Features**: OIDC provider, Artifact attestations (beta), Security tab SARIF ingestion

**When updating dependencies**: Update both workflow files AND README.md action versions table.

## File Structure Reference

```
.github/workflows/
  terraform-ci.yml          # Full CI with plan generation
  terraform-security.yml    # Security-only for PRs
example/
  README.md                 # Consumer usage patterns and setup guide
README.md                   # Main documentation, architecture diagrams
```

**Documentation updates**: When changing inputs/outputs, update both workflow comments AND README.md inputs table.
