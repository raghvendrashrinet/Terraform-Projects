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
