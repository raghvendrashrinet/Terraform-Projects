## 🚀 Demo Project: Terraform + Azure CI/CD Pipeline
#### 1. Create a New Repository
```bash
# Initialize a new repo locally
mkdir terraform-azure-demo
cd terraform-azure-demo
git init

# Add a README
echo "# Terraform Azure Demo Project" > README.md
git add README.md
git commit -m "Initial commit"

# Create remote repo on GitHub (via UI or CLI)
git remote add origin git@github.com:<your-org>/terraform-azure-demo.git
git push -u origin main
```
#### 2. Clone Repository
```bash
git clone git@github.com:<your-org>/terraform-azure-demo.git
cd terraform-azure-demo
```
#### 3. Create Feature Branch
```bash
git checkout -b feature/add-storage-account
```
- Branch name convention: `feature/<short-description>`
- Example: `feature/add-storage-account`

#### 4. Add Terraform Code
Create` main.tf` with Azure resource definition:

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "demo" {
  name     = "rg-demo"
  location = "eastus"
}

resource "azurerm_storage_account" "example" {
  name                     = "tfstorageacctdemo"
  resource_group_name      = azurerm_resource_group.demo.name
  location                 = azurerm_resource_group.demo.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```
**add Yaml Pipeline file ,see bottom of the doc**

#### 5. Commit and Push Feature Branch
```bash
git add main.tf
git commit -m "Add Azure Storage Account resource"
git push origin feature/add-storage-account
```

#### 6. Raise Pull Request (PR)
- Go to GitHub → open PR from feature/add-storage-account → main.
- PR template includes:
   * Summary: Added Azure Storage Account resource.
   * Plan Output: Attach terraform plan diff.
   * JSON Artifact: Attach tfplan.json for automated scans.
   * Checks: CI/CD runs security/compliance scans.

#### 7. Review Process
Developer: Confirms plan diff matches intent.

Security Tools: Run tfsec, Checkov, Trivy.

Compliance Tools: Run Conftest with Azure policies.

Approver: Reviews plan diff + scan reports, signs off.

#### 8. Merge and Apply
Once approved, PR is merged.

Pipeline runs:

```bash
terraform apply tfplan
```
Resources deployed to Azure.

✅ Summary
- Repo created → terraform-azure-demo.
- Feature branch → feature/add-storage-account.
- Terraform code added → Azure Storage Account.
- PR raised → includes plan + JSON + scans.
- Review & approval → ensures security/compliance.

---
### 📘 GitHub Actions Workflow 
Add below yaml file at  --> `.github/workflows/terraform.yml`
```yaml
name: Terraform Azure Pipeline

on:
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # Install Terraform
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Convert Plan to JSON
        run: terraform show -json tfplan > tfplan.json

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: tfplan.json

  security-scan:
    needs: terraform-plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: tfplan

      - name: Install Checkov
        run: pip install checkov

      - name: Install tfsec
        run: curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install.sh | bash

      - name: Install Trivy
        run: sudo apt-get update && sudo apt-get install -y trivy

      - name: Run Checkov
        run: checkov -f tfplan.json

      - name: Run tfsec
        run: tfsec .

      - name: Run Trivy
        run: trivy config .

  compliance-scan:
    needs: terraform-plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: tfplan

      - name: Install Conftest
        run: |
          sudo apt-get update
          sudo apt-get install -y conftest

      - name: Run Conftest
        run: conftest test tfplan.json

  approval-and-apply:
    needs: [security-scan, compliance-scan]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://portal.azure.com
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: tfplan

      - name: Terraform Apply
        run: terraform apply tfplan
```

Apply → deploys only the reviewed plan.  
