# Azure OIDC Quickstart

> 15-minute checklist to wire GitHub Actions → Azure using OpenID Connect (OIDC) so Terraform workflows run without static secrets.

## 1. Prerequisites
- Azure subscription + `Owner` (or delegated) rights to create app registrations and role assignments.
- Azure CLI ≥ 2.56 (`az --version`).
- GitHub repository that consumes `paloitmbb/mbb-tf-workflows`.
- Terraform backend hosted on Azure Storage (one container per environment).

## 2. Set variables once
```bash
export GITHUB_ORG="paloitmbb"
export GITHUB_REPO="mbb-tf-caller1"
export APP_DISPLAY_NAME="${GITHUB_REPO}-oidc"
export LOCATION="eastus"
export TFSTATE_RG="rg-terraform-state"
export TFSTATE_STORAGE="sttfstate001"   # 3-24 chars, lowercase
export TFSTATE_CONTAINER="tfstate"
az account show --output table            # ensure logged in
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
```

## 3. Create the app + service principal
```bash
az ad app create \
  --display-name "$APP_DISPLAY_NAME" \
  --sign-in-audience AzureADMyOrg

export AZURE_CLIENT_ID=$(az ad app list \
  --display-name "$APP_DISPLAY_NAME" \
  --query '[0].appId' -o tsv)

az ad sp create --id "$AZURE_CLIENT_ID"
```
Keep `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` for GitHub secrets.

## 4. Add federated credentials
Create one subject per trust boundary. Replace the `subject` value as needed and rerun the command.
```bash
create_fed() {
  az ad app federated-credential create \
    --id "$AZURE_CLIENT_ID" \
    --parameters "{\"name\":\"$1\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"$2\",\"audiences\":[\"api://AzureADTokenExchange\"]}"
}

create_fed main        "repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/main"
create_fed develop     "repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/develop"
create_fed pr          "repo:${GITHUB_ORG}/${GITHUB_REPO}:pull_request"
create_fed env-dev     "repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:dev"      # optional
create_fed env-stage   "repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:stage"    # optional
create_fed env-prod    "repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:prod"     # optional
```
Verify:
```bash
az ad app federated-credential list --id "$AZURE_CLIENT_ID" --output table
```

## 5. Provision Terraform state storage
```bash
az group create --name "$TFSTATE_RG" --location "$LOCATION"
az storage account create \
  --name "$TFSTATE_STORAGE" --resource-group "$TFSTATE_RG" --location "$LOCATION" \
  --sku Standard_LRS --kind StorageV2 --https-only true --min-tls-version TLS1_2 \
  --allow-blob-public-access false --enable-hierarchical-namespace false
az storage container create --name "$TFSTATE_CONTAINER" --account-name "$TFSTATE_STORAGE" --auth-mode login
# Optional safety nets
az storage account blob-service-properties update --account-name "$TFSTATE_STORAGE" --resource-group "$TFSTATE_RG" --enable-versioning true
az storage account blob-service-properties update --account-name "$TFSTATE_STORAGE" --resource-group "$TFSTATE_RG" --enable-delete-retention true --delete-retention-days 30
```

## 6. Assign RBAC
```bash
az role assignment create \
  --assignee "$AZURE_CLIENT_ID" \
  --role "Contributor" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}"

az role assignment create \
  --assignee "$AZURE_CLIENT_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${TFSTATE_RG}/providers/Microsoft.Storage/storageAccounts/${TFSTATE_STORAGE}"
```

## 7. Wire GitHub and Terraform
1. **Repository secrets** → add `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` under *Settings → Secrets and variables → Actions*.
2. **Workflows** → ensure permissions block includes `id-token: write` and `actions: read` (for SARIF uploads) before calling the reusable workflows.
3. **Backend config (`env/<env>/backend.tf`)**:
   ```hcl
   terraform {
     backend "azurerm" {
       resource_group_name  = "${TFSTATE_RG}"
       storage_account_name = "${TFSTATE_STORAGE}"
       container_name       = "${TFSTATE_CONTAINER}"
       key                  = "dev/terraform.tfstate"
       use_oidc             = true
     }
   }
   ```
4. **Provider config (`env/<env>/provider.tf`)** → set `use_oidc = true` and map the three IDs via variables or `TF_VAR_*` env vars.

## 8. Smoke test the credential
```yaml
# .github/workflows/test-oidc.yml
name: Test Azure OIDC
on: workflow_dispatch
permissions:
  id-token: write
  contents: read
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - name: Azure login
        uses: azure/login@a65d910e8af852a8061fe27ac7e98f4e797865c0 # v2.3.0
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: az storage container list --account-name $TFSTATE_STORAGE --auth-mode login --output table
```

## 9. Troubleshooting
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `AADSTS700016` application not found | Wrong tenant or client ID | Re-run `az ad app list --display-name "$APP_DISPLAY_NAME"` and update secrets. |
| `federated credential subject mismatch` | Subject string typo | Ensure it matches `repo:<org>/<repo>:ref:refs/heads/<branch>` or `:pull_request`. |
| `Storage account access denied` | Missing `Storage Blob Data Contributor` role | Re-run role assignment scoped to the storage account. |
| Terraform backend prompts for credentials | `use_oidc` missing or workflow lacks `id-token: write` | Add `use_oidc = true` in backend/provider and ensure workflow permissions include `id-token: write`. |

## 10. Next steps
- Duplicate federated credentials for every environment that uses GitHub Environments (subject: `repo:ORG/REPO:environment:<name>`).
- Rotate nothing: OIDC tokens are short-lived and issued on demand.
- Reference `mbb-tf-caller1/docs/GITHUB-ACTIONS-SETUP.md` for a repo-specific view of the same checklist.
