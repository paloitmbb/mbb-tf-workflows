# Azure OIDC Setup for GitHub Actions

Complete guide to configure Azure AD App Registration with OIDC (OpenID Connect) for GitHub Actions Terraform workflows. This eliminates the need for static credentials and provides secure, short-lived token-based authentication.

## 🎯 Overview

**What is OIDC?** OpenID Connect allows GitHub Actions to authenticate to Azure using short-lived tokens instead of storing long-lived credentials as secrets.

**Benefits:**
- ✅ No static secrets to rotate or manage
- ✅ Automatic credential lifecycle (1-hour token expiration)
- ✅ Audit trail via Azure AD sign-in logs
- ✅ Scoped access per repository/environment
- ✅ Eliminates secret sprawl and credential leaks

**Architecture:**
```
GitHub Actions Workflow
    ↓ (OIDC token request)
GitHub OIDC Provider (token.actions.githubusercontent.com)
    ↓ (short-lived JWT token)
Azure AD (validates federated credential)
    ↓ (Azure access token)
Azure Resources (Terraform deployment)
```

---

## 📋 Prerequisites

- Azure subscription with appropriate permissions
- GitHub repository where workflows will run
- Azure CLI installed (`az --version` to verify)
- Permissions to:
  - Create Azure AD App Registrations
  - Create Service Principals
  - Assign Azure RBAC roles
  - Configure Federated Credentials

---

## 🚀 Complete Setup Guide

### Step 1: Set Configuration Variables

Define your configuration variables at the start to use throughout the setup:

```bash
# =============================================================================
# CONFIGURATION VARIABLES - Update these for your setup
# =============================================================================

# GitHub Configuration
export GITHUB_ORG="paloitmbb"                    # Your GitHub organization
export GITHUB_REPO="mbb-tf-caller1"              # Your repository name

# Azure Configuration
export APP_DISPLAY_NAME="GitHub-Actions-Terraform-OIDC"  # App registration name
export LOCATION="eastus"                         # Azure region

# Terraform State Storage Configuration
export TFSTATE_RG="rg-terraform-state"           # Resource group for state
export TFSTATE_STORAGE="sttfstatedev001"         # Storage account name (3-24 chars, lowercase, alphanumeric)
export TFSTATE_CONTAINER="tfstate"               # Container name

# Optional: Environment-specific suffix
export ENV_SUFFIX="dev"                          # dev, stage, prod

# =============================================================================
# Verify configuration
# =============================================================================
echo "GitHub: ${GITHUB_ORG}/${GITHUB_REPO}"
echo "App Name: ${APP_DISPLAY_NAME}"
echo "Location: ${LOCATION}"
echo "State Storage: ${TFSTATE_STORAGE}"
```

---

### Step 2: Azure Authentication

Login to Azure and set your subscription:

```bash
# Login to Azure
az login

# List available subscriptions
az account list --output table

# Set the subscription you want to use
az account set --subscription "YOUR-SUBSCRIPTION-NAME-OR-ID"

# Verify current subscription
az account show --output table

# Get and save subscription details
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)

echo "Subscription ID: ${AZURE_SUBSCRIPTION_ID}"
echo "Tenant ID: ${AZURE_TENANT_ID}"
```

---

### Step 3: Create Azure AD App Registration

Create an App Registration that will represent your GitHub Actions workflow:

```bash
# Create App Registration
az ad app create \
  --display-name "${APP_DISPLAY_NAME}" \
  --sign-in-audience AzureADMyOrg

# Get the Application (Client) ID
export AZURE_CLIENT_ID=$(az ad app list \
  --display-name "${APP_DISPLAY_NAME}" \
  --query "[0].appId" \
  -o tsv)

# Get the Object ID (needed for federated credentials)
export APP_OBJECT_ID=$(az ad app list \
  --display-name "${APP_DISPLAY_NAME}" \
  --query "[0].id" \
  -o tsv)

echo "✅ App Registration Created"
echo "   Application (Client) ID: ${AZURE_CLIENT_ID}"
echo "   Object ID: ${APP_OBJECT_ID}"

# Create Service Principal
az ad sp create --id "${AZURE_CLIENT_ID}"

echo "✅ Service Principal Created"
```

**Save these values** - you'll need them later for GitHub Secrets:
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

---

### Step 4: Configure Federated Credentials

Create federated credentials that tell Azure AD to trust tokens from your GitHub repository:

#### 4.1: Main Branch Credential

```bash
# Create federated credential for main branch deployments
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsMain\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/main\",
    \"description\": \"GitHub Actions deployment from main branch\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

echo "✅ Federated Credential: main branch"
```

#### 4.2: Develop Branch Credential

```bash
# Create federated credential for develop branch
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsDevelop\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/develop\",
    \"description\": \"GitHub Actions deployment from develop branch\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

echo "✅ Federated Credential: develop branch"
```

#### 4.3: Pull Request Credential

```bash
# Create federated credential for pull requests
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsPullRequest\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:pull_request\",
    \"description\": \"GitHub Actions for pull request validation\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

echo "✅ Federated Credential: pull requests"
```

#### 4.4: Environment-Specific Credentials (Optional)

For environment-specific deployments using GitHub Environments:

```bash
# Example: Production environment
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsEnvironmentProd\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:prod\",
    \"description\": \"GitHub Actions deployment to prod environment\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

# Example: Staging environment
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsEnvironmentStage\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:stage\",
    \"description\": \"GitHub Actions deployment to stage environment\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

# Example: Development environment
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsEnvironmentDev\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:environment:dev\",
    \"description\": \"GitHub Actions deployment to dev environment\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

```

#### 4.5: Verify Federated Credentials

```bash
# List all federated credentials
az ad app federated-credential list \
  --id "${AZURE_CLIENT_ID}" \
  --output table

# Verify specific credential
az ad app federated-credential list \
  --id "${AZURE_CLIENT_ID}" \
  --query "[?subject=='repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/main']"
```

---

### Step 5: Create Terraform State Storage

Create Azure Storage Account to store Terraform state files:

#### 5.1: Create Resource Group

```bash
az group create \
  --name "${TFSTATE_RG}" \
  --location "${LOCATION}"

echo "✅ Resource Group Created: ${TFSTATE_RG}"
```

#### 5.2: Create Storage Account

```bash
az storage account create \
  --name "${TFSTATE_STORAGE}" \
  --resource-group "${TFSTATE_RG}" \
  --location "${LOCATION}" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --encryption-services blob \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --enable-hierarchical-namespace false

echo "✅ Storage Account Created: ${TFSTATE_STORAGE}"
```

#### 5.3: Create State Container

```bash
az storage container create \
  --name "${TFSTATE_CONTAINER}" \
  --account-name "${TFSTATE_STORAGE}" \
  --auth-mode login

echo "✅ Container Created: ${TFSTATE_CONTAINER}"
```

#### 5.4: Enable Versioning (Recommended)

```bash
# Enable blob versioning for state file protection
az storage account blob-service-properties update \
  --account-name "${TFSTATE_STORAGE}" \
  --resource-group "${TFSTATE_RG}" \
  --enable-versioning true

# Enable soft delete for blob recovery
az storage account blob-service-properties update \
  --account-name "${TFSTATE_STORAGE}" \
  --resource-group "${TFSTATE_RG}" \
  --enable-delete-retention true \
  --delete-retention-days 30

```

---

### Step 6: Assign Azure RBAC Permissions

Grant the Service Principal necessary permissions:

#### 6.1: Subscription-Level Contributor Role

```bash
# Assign Contributor role for infrastructure deployment
az role assignment create \
  --assignee "${AZURE_CLIENT_ID}" \
  --role "Contributor" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}"

echo "✅ Contributor role assigned (subscription scope)"
```

#### 6.2: Storage Account Access

```bash
# Assign Storage Blob Data Contributor for state file access
az role assignment create \
  --assignee "${AZURE_CLIENT_ID}" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${TFSTATE_RG}/providers/Microsoft.Storage/storageAccounts/${TFSTATE_STORAGE}"

echo "✅ Storage Blob Data Contributor role assigned"
```

#### 6.3: Verify Role Assignments

```bash
# List all role assignments for the service principal
az role assignment list \
  --assignee "${AZURE_CLIENT_ID}" \
  --output table

# Verify specific storage account permissions
az role assignment list \
  --assignee "${AZURE_CLIENT_ID}" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${TFSTATE_RG}/providers/Microsoft.Storage/storageAccounts/${TFSTATE_STORAGE}" \
  --output table
```

---

### Step 7: Save Configuration Summary

Create a summary file with all the values you'll need:

```bash
# Create configuration summary
cat > azure-oidc-config.txt << CONFIG
# =============================================================================
# Azure OIDC Configuration Summary
# Generated: $(date)
# =============================================================================

# GitHub Repository
GITHUB_ORG="${GITHUB_ORG}"
GITHUB_REPO="${GITHUB_REPO}"

# Azure AD Configuration (Add these as GitHub Secrets)
AZURE_CLIENT_ID="${AZURE_CLIENT_ID}"
AZURE_TENANT_ID="${AZURE_TENANT_ID}"
AZURE_SUBSCRIPTION_ID="${AZURE_SUBSCRIPTION_ID}"

# Azure Resources
APP_DISPLAY_NAME="${APP_DISPLAY_NAME}"
APP_OBJECT_ID="${APP_OBJECT_ID}"

# Terraform State Storage (Update in backend.tf)
TFSTATE_RESOURCE_GROUP="${TFSTATE_RG}"
TFSTATE_STORAGE_ACCOUNT="${TFSTATE_STORAGE}"
TFSTATE_CONTAINER="${TFSTATE_CONTAINER}"
LOCATION="${LOCATION}"

# =============================================================================
# Next Steps
# =============================================================================
# 1. Add GitHub Secrets:
#    - Go to: https://github.com/${GITHUB_ORG}/${GITHUB_REPO}/settings/secrets/actions
#    - Add: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID
#
# 2. Update Terraform backend configuration (backend.tf):
#    resource_group_name  = "${TFSTATE_RG}"
#    storage_account_name = "${TFSTATE_STORAGE}"
#    container_name       = "${TFSTATE_CONTAINER}"
#    use_azuread_auth     = true
#
# 3. Test the workflow:
#    - Create a test branch and push changes
#    - Verify GitHub Actions can authenticate to Azure
# =============================================================================
CONFIG

echo "✅ Configuration saved to: azure-oidc-config.txt"
cat azure-oidc-config.txt
```

---

## 🔧 GitHub Repository Configuration

### Step 8: Add GitHub Repository Secrets

1. Navigate to your GitHub repository
2. Go to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add the following secrets:

| Secret Name | Value | Where to Find |
|-------------|-------|---------------|
| `AZURE_CLIENT_ID` | Application (Client) ID | From Step 3 (`echo $AZURE_CLIENT_ID`) |
| `AZURE_TENANT_ID` | Directory (Tenant) ID | From Step 2 (`echo $AZURE_TENANT_ID`) |
| `AZURE_SUBSCRIPTION_ID` | Subscription ID | From Step 2 (`echo $AZURE_SUBSCRIPTION_ID`) |

**Quick Command to Display Values:**
```bash
echo "Add these to GitHub Secrets:"
echo "AZURE_CLIENT_ID: ${AZURE_CLIENT_ID}"
echo "AZURE_TENANT_ID: ${AZURE_TENANT_ID}"
echo "AZURE_SUBSCRIPTION_ID: ${AZURE_SUBSCRIPTION_ID}"
```

### Step 9: Update Terraform Backend Configuration

Update your `backend.tf` file:

```hcl
# env/dev/backend.tf (or your environment path)
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"        # From $TFSTATE_RG
    storage_account_name = "sttfstatedev001"           # From $TFSTATE_STORAGE
    container_name       = "tfstate"                   # From $TFSTATE_CONTAINER
    key                  = "dev/terraform.tfstate"     # Environment-specific key
    use_azuread_auth     = true                        # Enable OIDC authentication
  }
}


---

## ✅ Verification & Testing

### Step 10: Verify Setup

#### 10.1: Test Azure Authentication Locally (Optional)

```bash
# Login as the service principal using OIDC (simulation)
az login --service-principal \
  --username "${AZURE_CLIENT_ID}" \
  --tenant "${AZURE_TENANT_ID}" \
  --allow-no-subscriptions

# List accessible resources
az resource list --output table
```

#### 10.2: Test Storage Access

```bash
# Test storage account access
az storage blob list \
  --account-name "${TFSTATE_STORAGE}" \
  --container-name "${TFSTATE_CONTAINER}" \
  --auth-mode login
```

#### 10.3: Verify Federated Credential Configuration

```bash
# Show federated credentials
az ad app federated-credential list \
  --id "${AZURE_CLIENT_ID}" \
  --query '[].{Name:name, Subject:subject}' \
  --output table

# Expected output should show:
# - GitHubActionsMain (repo:ORG/REPO:ref:refs/heads/main)
# - GitHubActionsDevelop (repo:ORG/REPO:ref:refs/heads/develop)
# - GitHubActionsPullRequest (repo:ORG/REPO:pull_request)
```

---

### Step 11: Test GitHub Actions Workflow

Create a test workflow to verify OIDC authentication:

```yaml
# .github/workflows/test-azure-oidc.yml
name: Test Azure OIDC

on:
  workflow_dispatch:
  push:
    branches: [main, develop]

permissions:
  id-token: write      # Required for OIDC token
  contents: read

jobs:
  test-azure-access:
    runs-on: ubuntu-latest
    steps:
      - name: Azure OIDC Login
        uses: azure/login@a65d910e8af852a8061fe27ac7e98f4e797865c0  # v2.3.0
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Verify Azure Access
        run: |
          echo "Testing Azure CLI access..."
          az account show
          az group list --output table
          
      - name: Test Storage Access
        run: |
          az storage blob list \
            --account-name sttfstatedev001 \
            --container-name tfstate \
            --auth-mode login \
            --output table
```

---

## 🐛 Troubleshooting

### Common Issues and Solutions

#### Issue 1: OIDC Token Validation Failed

**Error:**
```
Error: AADSTS700016: Application with identifier 'XXX' was not found in the directory
```

**Solution:**
```bash
# Verify app registration exists
az ad app show --id "${AZURE_CLIENT_ID}"

# Check service principal
az ad sp show --id "${AZURE_CLIENT_ID}"
```

#### Issue 2: Invalid Federated Credential Subject

**Error:**
```
Error: AADSTS700238: The federated credential subject does not match
```

**Solution:**
```bash
# Verify subject format exactly matches
# Format: repo:{ORG}/{REPO}:ref:refs/heads/{BRANCH}
# Example: repo:paloitmbb/mbb-tf-caller1:ref:refs/heads/main

# List and verify subjects
az ad app federated-credential list \
  --id "${AZURE_CLIENT_ID}" \
  --query '[].{Name:name, Subject:subject}' \
  --output table

# Delete incorrect credential
az ad app federated-credential delete \
  --id "${AZURE_CLIENT_ID}" \
  --federated-credential-id "CREDENTIAL_NAME"

# Recreate with correct subject
az ad app federated-credential create \
  --id "${AZURE_CLIENT_ID}" \
  --parameters "{
    \"name\": \"GitHubActionsMain\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_ORG}/${GITHUB_REPO}:ref:refs/heads/main\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
```

#### Issue 3: Missing id-token Permission

**Error:**
```
Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable
```

**Solution:**
Ensure workflow has `id-token: write` permission:
```yaml
permissions:
  id-token: write
  contents: read
```

#### Issue 4: Storage Access Denied

**Error:**
```
Error: storage: service returned error: StatusCode=403
```

**Solution:**
```bash
# Verify Storage Blob Data Contributor role
az role assignment list \
  --assignee "${AZURE_CLIENT_ID}" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${TFSTATE_RG}/providers/Microsoft.Storage/storageAccounts/${TFSTATE_STORAGE}"

# Reassign if missing
az role assignment create \
  --assignee "${AZURE_CLIENT_ID}" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${TFSTATE_RG}/providers/Microsoft.Storage/storageAccounts/${TFSTATE_STORAGE}"
```

#### Issue 5: Role Assignment Propagation Delay

**Error:**
```
Error: The client does not have authorization to perform action
```

**Solution:**
Azure RBAC role assignments can take 5-10 minutes to propagate. Wait and retry.

```bash
# Check role assignment status
az role assignment list \
  --assignee "${AZURE_CLIENT_ID}" \
  --all \
  --output table

# If needed, wait and verify again
sleep 300  # Wait 5 minutes
```

---

## 📖 Reference

### Subject Format Guide

| Credential Type | Subject Format | Example |
|----------------|----------------|---------|
| **Specific Branch** | `repo:ORG/REPO:ref:refs/heads/BRANCH` | `repo:paloitmbb/mbb-tf-caller1:ref:refs/heads/main` |
| **Pull Request** | `repo:ORG/REPO:pull_request` | `repo:paloitmbb/mbb-tf-caller1:pull_request` |
| **Environment** | `repo:ORG/REPO:environment:ENV_NAME` | `repo:paloitmbb/mbb-tf-caller1:environment:prod` |
| **Any Branch** | `repo:ORG/REPO:ref:refs/heads/*` | `repo:paloitmbb/mbb-tf-caller1:ref:refs/heads/*` |
| **Tag** | `repo:ORG/REPO:ref:refs/tags/TAG` | `repo:paloitmbb/mbb-tf-caller1:ref:refs/tags/v1.0.0` |

### Required Azure Roles

| Role | Scope | Purpose |
|------|-------|---------|
| **Contributor** | Subscription or Resource Group | Deploy and manage infrastructure resources |
| **Storage Blob Data Contributor** | Storage Account | Read/write Terraform state files |
| **User Access Administrator** | (Optional) Subscription | Manage role assignments in deployments |

### GitHub Actions Permissions

```yaml
permissions:
  id-token: write        # Required for OIDC token
  contents: read         # Read repository code
  pull-requests: write   # Comment on PRs (optional)
  security-events: write # Upload SARIF (optional)
```

---

## 🔗 Additional Resources

- **Official Microsoft Documentation:**
  - [Configure OpenID Connect in Azure](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure)
  - [Azure RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/)
  - [Terraform azurerm Backend](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)

- **GitHub Actions:**
  - [Security Hardening with OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
  - [Azure Login Action](https://github.com/Azure/login)

- **Security Best Practices:**
  - [Azure AD Security Best Practices](https://learn.microsoft.com/en-us/azure/active-directory/develop/security-best-practices-for-app-registration)
  - [GitHub Actions Security](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

---

## 📝 Quick Reference Commands

### View Current Configuration

```bash
# Display all configuration
echo "Client ID: ${AZURE_CLIENT_ID}"
echo "Tenant ID: ${AZURE_TENANT_ID}"
echo "Subscription ID: ${AZURE_SUBSCRIPTION_ID}"
echo "Storage Account: ${TFSTATE_STORAGE}"
echo "Resource Group: ${TFSTATE_RG}"

# List federated credentials
az ad app federated-credential list --id "${AZURE_CLIENT_ID}" --output table

# List role assignments
az role assignment list --assignee "${AZURE_CLIENT_ID}" --output table
```

### Clean Up (If Needed)

```bash
# Delete federated credential
az ad app federated-credential delete \
  --id "${AZURE_CLIENT_ID}" \
  --federated-credential-id "CREDENTIAL_NAME"

# Delete role assignment
az role assignment delete \
  --assignee "${AZURE_CLIENT_ID}" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}"

# Delete service principal
az ad sp delete --id "${AZURE_CLIENT_ID}"

# Delete app registration
az ad app delete --id "${AZURE_CLIENT_ID}"

# Delete storage resources (DANGER: This will delete state files!)
az storage account delete \
  --name "${TFSTATE_STORAGE}" \
  --resource-group "${TFSTATE_RG}" \
  --yes

az group delete --name "${TFSTATE_RG}" --yes
```

---

## ✅ Setup Checklist

Use this checklist to ensure complete setup:

- [ ] Azure CLI installed and authenticated
- [ ] Configuration variables set (Step 1)
- [ ] Azure subscription selected (Step 2)
- [ ] App Registration created (Step 3)
- [ ] Service Principal created (Step 3)
- [ ] Federated credentials configured (Step 4)
  - [ ] Main branch credential
  - [ ] Develop branch credential
  - [ ] Pull request credential
  - [ ] Environment-specific credentials (if needed)
- [ ] Terraform state storage created (Step 5)
  - [ ] Resource group created
  - [ ] Storage account created
  - [ ] Container created
  - [ ] Versioning/soft delete enabled
- [ ] Azure RBAC roles assigned (Step 6)
  - [ ] Contributor role (subscription)
  - [ ] Storage Blob Data Contributor role
- [ ] Configuration summary saved (Step 7)
- [ ] GitHub secrets added (Step 8)
  - [ ] AZURE_CLIENT_ID
  - [ ] AZURE_TENANT_ID
  - [ ] AZURE_SUBSCRIPTION_ID
- [ ] Terraform backend.tf updated (Step 9)
- [ ] Setup verified (Step 10)
- [ ] Test workflow executed successfully (Step 11)

---

**You're all set!** 🎉 Your GitHub Actions workflows can now securely authenticate to Azure using OIDC without storing any static credentials.
