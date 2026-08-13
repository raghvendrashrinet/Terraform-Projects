## Terraform Providers and API Details
rraform uses providers to interact with various cloud platforms, SaaS services, and other APIs. A provider is a plugin that Terraform uses to manage infrastructure resources on a specific platform like AWS, Azure, GCP, Kubernetes, etc.

Each provider communicates with the respective cloud provider’s API to perform operations like creating, updating, and deleting resources.

```
 Terraform --> Provider --> Cloud API --> Manages Cloud Resource
```

#### 1.Azure Provider (hashicorp/azurerm)
```hcl
terraform{
    required_providers{
        azurerm = {
            source ="hashicorp/azurerm"
            version="5.0.0"
        }
    }
}

provider "azurerm" {
    features {}
}
```

#### Kubernetes Provider (hashicorp/kubernetes)

Uses Kubernetes API Server to manage resources like Pods, Deployments, Services, etc.
Authentication via kubeconfig
```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

### AWS Provider  (hashicorp/aws)
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

```
