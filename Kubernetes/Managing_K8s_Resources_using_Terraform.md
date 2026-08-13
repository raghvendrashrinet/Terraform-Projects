### Terraform Kubernetes Provider
enables you to manage native Kubernetes resources using HashiCorp Configuration Language (HCL) instead of raw YAML files or manual kubectl commands.
```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

# Configure the provider using local kubeconfig
provider "kubernetes" {
  config_path = "~/.kube/config"
}

# Create a Kubernetes Namespace
resource "kubernetes_namespace" "example" {
  metadata {
    name = "my-app-namespace"
  }
}

# Create an NGINX Deployment
resource "kubernetes_deployment" "nginx" {
  metadata {
    name      = "nginx-deployment"
    namespace = kubernetes_namespace.example.metadata[0].name
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "nginx"
      }
    }

    template {
      metadata {
        labels = {
          app = "nginx"
        }
      }

      spec {
        container {
          image = "nginx:latest"
          name  = "nginx"

          port {
            container_port = 80
          }
        }
      }
    }
  }
}
```
