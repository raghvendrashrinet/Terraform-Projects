### terraform Block
The terraform block is used to configure settings that affect the entire Terraform configuration. This includes specifying the required Terraform version, setting up backend configurations, and managing state.
### required_providers Block
The required_providers block is used to specify which providers your configuration requires, along with their version constraints. 

```hcl
terraform{
    required_providers{
        azurerm = {
            source ="hashicorp/azurerm"
            version="5.0.0"
        }

```
### provider Block
The provider block configures the Azure provider (or any other provider). It defines the credentials and settings that Terraform uses to interact with Azure.

```hcl
provider "azurerm" {
  features {}
  tenant_id       = "your-tenant-id"
  subscription_id = "your-subscription-id"
  client_id       = "your-client-id"
  client_secret   = "your-client-secret"
}
```

---

###  resource Block
The resource block is used to define and manage a specific resource
```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  location            = "East US"
  resource_group_name = "my-resource-group"
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "example" {
  name                 = "example-subnet"
  resource_group_name  = "my-resource-group"
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}
```
---

### output Block
The output block is used to define values that should be displayed after running terraform apply. This is useful for returning important information like IP addresses, IDs, or other resource-specific values.
- Useful for returning dynamic values that are generated during terraform apply
```hcl
output "vm_public_ip" {
  value = azurerm_linux_virtual_machine.example.public_ip_address
}
```
---
###  variable Block
The variable block is used to define input variables, making your Terraform configuration more dynamic and reusable  
variables.yaml  
```hcl
variable "region" {
  type    = string
  default = "East US"
  description = " located in east us "
}

variable "resource_group_name" {
  type    = string
  default = "my-resource-group"
}
```
---
### locals Block
The locals block is used to define local values (expressions) that are evaluated once and used throughout the configuration. This helps reduce duplication and makes the code cleaner.

```hcl
locals {
  vm_size = "Standard_B1s"
}

resource "azurerm_linux_virtual_machine" "example" {
  name                = "example-vm"
  resource_group_name = var.resource_group_name
  location            = var.region
  size                = local.vm_size
  ...
```
---

### data Block
The data block allows you to query and retrieve data from existing Azure resources, without managing them directly.

```hcl
data "azurerm_image" "latest_ubuntu" {
  name                = "UbuntuServer"
  resource_group_name = "my-resource-group"
  filter {
    name = "sku"
    values = ["18.04-LTS"]
  }
}

resource "azurerm_linux_virtual_machine" "example" {
  name                = "example-vm"
  resource_group_name = "my-resource-group"
  location            = "East US"
  size                = "Standard_B1s"
  admin_username      = "adminuser"
  admin_password      = "P@ssw0rd1234!"
  network_interface_ids = [
    azurerm_network_interface.example.id,
  ]
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  source_image_reference {
    id = data.azurerm_image.latest_ubuntu.id
  }
}
```
---

###  module Block
The module block allows you to call reusable modules, making your code more modular and maintainable. Modules are a way to encapsulate and reuse configurations.
```hcl
module "vnet" {
  source              = "Azure/vnet/azurerm"
  resource_group_name = "my-resource-group"
  location            = "East US"
  address_space       = ["10.0.0.0/16"]
}
```






