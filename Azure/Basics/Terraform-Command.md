### What Does terraform init Do?
terraform init initializes a Terraform working directory. It performs the following tasks:

- Downloads Providers  
   Fetches provider plugins specified in the provider block.Downloads them from the Terraform Registry and places them in the `.terraform` directory.
- Configures Backend  
  Sets up the backend (e.g., local, S3, Azure Storage, GCS) to store the Terraform state.
- Checks and Verifies Modules  
  Downloads external Terraform modules specified in the configuration.
- Prepares for Execution  
  Ensures everything is properly set up for running Terraform commands like plan and apply.

### What is .terraform Directory?
- It is a hidden directory created after running terraform init.
- Contains provider binaries, modules, and backend-related files.
- Helps Terraform execute commands faster without redownloading providers.
```
.terraform/
│── providers/
│   ├── registry.terraform.io/
│   │   ├── hashicorp/
│   │   │   ├── aws/
│   │   │   │   ├── 5.0.0/
│   │   │   │   │   ├── terraform-provider-aws_v5.0.0_x5
│── modules/
│── plugin-cache/
```

### What is terraform.lock.hcl?
It is an auto-generated lock file that ensures Terraform always uses the same provider versions.
Prevents accidental upgrades that might break infrastructure.
Stored in the root directory of the Terraform project.  
Example Content of `terraform.lock.hcl`:
```
provider "registry.terraform.io/hashicorp/aws" {
  version = "5.0.0"
  hashes = [
    "h1:xyzabc123456...",
    "zh:abcde12345..."
  ]
}
```

### ~> operator
- ~> `3.0.0 (Three digits): Allows patch updates only.`  
    * Allowed: 3.0.1, 3.0.2, 3.0.1
    * Blocked: 3.1.0 (minor change) and 4.0.0 (major change)
- ~>` 3.0 (Two digits): Allows minor and patch updates, but blocks major updates.`
    * Allowed: 3.1.0, 3.5.2, 3.9.0
    * Blocked: 4.0.0 (major change)
- ~>` 1.0: In your previous required_version context,`  
    * using ~> 1.0 would allow versions like 1.5.0 or 1.11.0,
    * but would completely block Terraform 2.0.0
---
### terraform plan (Preview step)
Think of this as a `dry run` or a `blueprint` review.
- `Refreshes State`: It queries your cloud provider (like Azure) to see the current real-world status of your existing infrastructure.
- `Compares Code to Reality`: It looks at your local .tf files and calculates the differences between your desired setup and what currently exists.
- `Generates a Report`: It lists exactly what actions it will take using three visual markers
- `Validates Syntax`
- What it shows:  
  🟢 + (Create): New resources that will be built.  
  🟡 ~ (Update): Existing resources that will be modified in place.  
  🔴 - (Destroy): Existing resources that will be deleted.  
      -/+ Replace → A resource will be destroyed and recreated (common when immutable fields change).


---
### terraform apply (The Execution Step)
- `Re-runs the Plan`: It automatically runs a fresh terraform plan first to show you the changes one final time
- `Prompts for Approva`
- `Calls Cloud APIs`
- `Handles Resource Ordering`: build dependency order
- `Updates the State File`: once resource gets created ,updates state file

---
## Step-by-Step Production Workflow
##### 1. Save the Plan as an Artifact
  ```
  terraform plan -out=tfplan
  ```
##### 2. Convert to JSON for Analysis
   Because the binary tfplan file is not human-readable or machine-scannable, you convert it into a structured JSON format 
  ```
   terraform show -json tfplan > tfplan.json
  ```
##### 3. Run Automated Security and Compliance Scans
- `Checkov / Trivy / tfsec`: code scan for security vulnerability
- `Open Policy Agent (OPA) / Conftest`:  Conftest: Evaluates your custom company policies (e.g., "Production VMs must use a specific size" or "No production deletions allowed without secondary approval")

##### 4. Peer Review and Manual Approval   
The plan is posted inside a Pull Request (PR) in tools like GitHub Actions. Senior engineers review it, keeping a strict lookout for The Red Flags:
- 🔴 - (Destroy): Is something being deleted that shouldn't be?
- 💥 -/+ (Replace): A tiny change to an immutable attribute can force a database or critical server to completely destroy and recreate itself, causing unexpected downtime

##### 5.  Apply the Exact Saved Plan File
Once approved, you execute the exact file you reviewed:
```
 terraform apply "tfplan"
```
