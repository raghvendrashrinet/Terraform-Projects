## Terraform Indexing Guide: Working with Lists, Maps, and Nested Blocks
#### 1. Overview
In Terraform (HCL), indexing allows you to access individual elements inside lists, maps, or nested configuration blocks exported by resources.

Because HCL uses 0-based indexing:
- Index [0] accesses the 1st element.
- Index [N] accesses the (N+1)th element.

#### Why Resource Blocks Use Indexing ([0])
In some providers (like the HashiCorp Kubernetes Provider), single nested blocks (such as metadata or spec) are represented under the hood as a 1-element list of maps.  
Even if a resource only allows one metadata block, Terraform treats it as a list with a single item at index 0.

1. Example: Single Block Access
```hcl
   # Creating a Kubernetes Namespace
resource "kubernetes_namespace" "example" {
  metadata {
    name = "my-app-namespace"
  }
}

# Referencing the name from the metadata block using [0]
resource "kubernetes_deployment" "nginx" {
  metadata {
    name      = "nginx-deployment"
    namespace = kubernetes_namespace.example.metadata[0].name
  }
}

```
> [!NOTE]
> Alternative Clean Syntax: You can also use Terraform's built-in one() function to simplify single-element list access:

```Terraform
namespace = one(kubernetes_namespace.example.metadata).name
```
2. Example: Multi-Container Pod
```hcl
resource "kubernetes_pod" "app" {
  metadata {
    name = "multi-container-pod"
  }

  spec {
    # Index [0]: Primary Container
    container {
      name  = "web-app"
      image = "nginx:latest"
    }

    # Index [1]: Sidecar Container
    container {
      name  = "sidecar-logger"
      image = "fluentd:latest"
    }
  }
}
```
- `kubernetes_pod.app.spec[0].container[0].name	---> "web-app`
-  `kubernetes_pod.app.spec[0].container[1].name ---> "sidecar-logger"`
