### Example Scenario
- You need to create a storage account (tfstorageacct) in Azure that enforces HTTPS and uses Local Redundancy (LRS).

#### Step 1: Local Development & PR Creation
1. Developer Writes Code: You create a new branch (feature/add-storage) and add your Terraform code (main.tf):

```Terraform
resource "azurerm_storage_account" "example" {
  name                     = "tfstorageacct"
  resource_group_name      = "tf-rg"
  location                 = "East US"
  account_tier             = "Standard"
  account_replication_type = "LRS"
  enable_https_traffic_only = true
}
```
2.  `Push Branch`: You push your changes to GitHub / Azure DevOps:

```Bash
git add main.tf
git commit -m "add storage account"
git push origin feature/add-storage
```

3. `Open PR`: You open a Pull Request targeting the main branch.

#### Step 2: PR Pipeline Triggers (Automation / CI)
As soon as the PR is opened, the CI pipeline runs automatically on an isolated build agent (runner):

1. Validation & Formatting Check:

```Bash
terraform fmt -check
terraform init
terraform validate
```
2. Generate Plan Artifacts:
  - Generates binary plan:

```Bash
terraform plan -out=tfplan
```
  - Converts plan to JSON so security scanners can parse it:

```Bash
terraform show -json tfplan > tfplan.json
```
3. Security & Compliance Scanning:

 - `tfsec / Checkov`: Scans tfplan.json for security misconfigurations.
    * Example check: Ensures enable_https_traffic_only is set to true.
 -  Conftest (OPA): Evaluates organizational policies.
    * Example policy: Ensures standard naming conventions (tfstorageacct...) or restricts allowed cloud regions.

4. Post Results to PR:
The runner converts the human-readable diff into a Markdown comment on your PR:

```Diff
+ resource "azurerm_storage_account" "example" {
+     account_replication_type = "LRS"
+     account_tier             = "Standard"
+     enable_https_traffic_only = true
+     name                     = "tfstorageacct"
+ }

--- Scans Status ---
✅ tfsec: 0 High, 0 Medium issues
✅ Checkov: Passed (HTTPS enforced)
✅ Conftest: Policy checks passed
```
####  Step 3: Peer & Security Review (Human Oversight)
1. Reviewer Inspects: A Senior Engineer / Tech Lead reviews the PR.

2. Verification:
  * They look at the code changes in the PR.
  * They review the Plan Diff in the PR comments to make sure no unexpected deletions (-) or unwanted modifications (~) are occurring.
  * They check the automated security/compliance report status.

3. Approval: If everything looks good, the reviewer clicks Approve.

####  Step 4: PR Merge & CD Execution (Deploy)
1. Merge: The PR is merged into main.

2. Continuous Deployment (CD) Pipeline Triggers:
   * The pipeline fetches the merged code.
   * (Optional/Standard) An Approval Gate prompts an Ops lead / Senior Admin to sign off before impacting production.

3. Apply: Once approved, the runner executes the apply stage:

```Bash
terraform apply -auto-approve tfplan
```
4. Verification: Azure provisions the storage account, and state is updated securely in remote state storage (e.g., Azure Blob / Terraform Cloud).

---
**Two common ways to structure the YAML**
#### Option A: GitHub Actions Workflow
`(.github/workflows/terraform.yml)`
```yaml
name: "Terraform PR & Deploy Pipeline"

on:
  pull_request:
    branches: [ "main" ]
  push:
    branches: [ "main" ]

jobs:
  # ------------------------------------------------------------------
  # Stage 1: Pull Request CI (Lint, Plan, Scan & Report)
  # ------------------------------------------------------------------
  pr-validation:
    name: "PR Validation & Security Scans"
    # if Prevents PR validation from re-running during the post-merge push event
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Format & Init
        run: |
          terraform fmt -check
          terraform init

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json
          terraform show tfplan > plan_output.txt

      - name: Run Security Scans (tfsec & Checkov)
        run: |
          # Run tfsec on the plan
          # Mounts repo directory $(pwd) to /src inside the container,for docker to read files,download tfsec/tfsec and last /src is an asrg to tfsec
          docker run --rm -v $(pwd):/src tfsec/tfsec /src

          # Run Checkov on the JSON plan
          docker run --rm -v $(pwd):/tf bridgecrew/checkov -f /tf/tfplan.json

      - name: Post Plan & Scan Summary to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planText = fs.readFileSync('plan_output.txt', 'utf8');
            const output = `#### Terraform Plan Output 📖
            \`\`\`hcl
            ${planText.substring(0, 6000)}
            \`\`\`
            *Note: Full plan attached to job logs.*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  # ------------------------------------------------------------------
  # Stage 2: CD / Apply Stage (Main Branch Post-Merge)
  # ------------------------------------------------------------------
  deploy:
    name: "Terraform Apply"
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production # Triggers manual approval gate in GitHub Settings

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init & Apply
        run: |
          terraform init
          terraform apply -auto-approve
```

#### Option B: Azure DevOps Pipeline 
`(azure-pipelines.yml)`
```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

stages:
  # ------------------------------------------------------------------
  # Stage 1: Validation & Security Scans (Triggered on PR)
  # ------------------------------------------------------------------
  - stage: PR_Validation
    displayName: "PR Validation & Compliance"
    jobs:
      - job: ValidateAndScan
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: 'latest'

          - script: |
              terraform init
              terraform plan -out=tfplan
              terraform show -json tfplan > tfplan.json
            displayName: 'Terraform Init & Plan'

          - script: |
              # Execute Checkov scan on the JSON plan artifact
              docker run --rm -v $(System.DefaultWorkingDirectory):/tf bridgecrew/checkov -f /tf/tfplan.json
            displayName: 'Run Checkov Security Scan'

          - script: |
              # Execute tfsec scan
              docker run --rm -v $(System.DefaultWorkingDirectory):/src tfsec/tfsec /src
            displayName: 'Run tfsec Scan'

  # ------------------------------------------------------------------
  # Stage 2: Deploy / Apply (Triggered on Merge to Main)
  # ------------------------------------------------------------------
  - stage: Deploy_Production
    displayName: "Apply Terraform Changes"
    dependsOn: PR_Validation
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Apply
        environment: 'production' # Connects to Azure DevOps Environment with Approval Gates
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: 'latest'

                - script: |
                    terraform init
                    terraform apply -auto-approve
                  displayName: 'Terraform Apply'
```



