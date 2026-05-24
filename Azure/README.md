## Enterprise Landing Zones (ELZ) 
Building "Big Company" Azure infrastructure with Terraform isn't about writing more code—it's about following Enterprise Landing Zones (ELZ) . Large firms enforce strict separation between network, security, and application teams
#### 🏗️ The "Big Company" Architecture
Instead of one big resource group, think Subscriptions and Landing Zones .

Connectivity Subscription: Houses the Hub VNet, Azure Firewall, and VPN Gateway for central network egress.

Management Subscription: Contains Log Analytics Workspaces, Automation Accounts, and Backup Vaults for global monitoring.

Identity Subscription: Stores Azure Key Vault and Private DNS Zones.

Application Landing Zone (Spoke): The actual workload (VMs, AKS, SQL). It is peered to the Hub to force traffic through the Firewall .

##### 📂 Folder Structure (The "Modular" Layout)
Avoid a single main.tf. Enterprise code looks like this
```
├── environments/          # Config per environment
│   ├── dev/               # dev.tfvars, dev.conf
│   └── prod/              # prod.tfvars, prod.conf
├── modules/               # Reusable components
│   ├── networking/
│   ├── aks/               # Kubernetes cluster
│   └── database/
└── infra/                 # Root configs calling modules
```
#### 🔐 1. State Management (Non-negotiable)
You cannot use local state files. You must use Azure Blob Storage with Entra ID Authentication (not access keys)
```
# backend-config file
resource_group_name  = "rg-platform-identity"
storage_account_name = "sttfstateprod01"
container_name       = "tfstate"
key                  = "prod/network/terraform.tfstate"
use_azuread_auth     = true # Enforces Entra ID
use_msi              = true # Uses Managed Identity in CI/CD
```
#### 🔒 2. Networking & Security
Enterprises use Private Endpoints for PaaS services (Storage, SQL, Key Vault) to keep traffic off the public internet .

Policy: Deny public network access on all resources.

DNS: Private DNS Zones linked to your Hub VNet resolve the private IPs automatically

#### 🔄 3. CI/CD & Automation
CI/CD is how big companies prevent "drift" (manual changes in the portal) .

Authentication: Use OpenID Connect (OIDC) from GitHub Actions or Azure DevOps. Never store service principal secrets .

Pipeline Flow:

PR Trigger: terraform plan runs automatically.
Merge to Main: terraform apply runs.
Scheduled Job: Nightly terraform plan -refresh-only to detect manual changes (drift)

#### 🧠 4. Handling Drift (The Pro Move)
In enterprises, things change outside of Terraform. Use terraform plan -refresh-only to sync state without changing infrastructure .

Use Case: If an ops team resizes a VM via the portal, -refresh-only updates your state file to match reality without destroying the VM.

Ignore Rules: Use lifecycle { ignore_changes = [tags] } for attributes that change automatically (like auto-scaling values)
#### 🚀 Action Plan for You
Start with the Backend: Manually create the Storage Account via Portal/CLI. Configure the azurerm backend with use_azuread_auth = true.

Build the "Hub": Write a module for the Hub VNet, Firewall, and Private DNS Zones.

Implement CI/CD: Set up OIDC in your GitHub repo. Create a pipeline that runs plan on PRs and apply on merge .

Add a "Spoke": Deploy a simple VM or AKS cluster in the Spoke, enforcing routing to the Hub Firewal

### Day 1. No modules, no CI/CD yet—just the foundational decisions that make everything else possible.

##### 📋 Step 1: The Governance Meeting (Before Any Code)
Before writing Terraform, the company decides:

Decision	Typical Enterprise Choice
Subscription Strategy	One "Platform" subscription for shared services, separate "Landing Zone" subscriptions per app team
State Location	Central storage account in Platform subscription, one container per environment
Naming Convention	[resourcetype]-[appname]-[environment]-[region][number] (e.g., rg-prod-network-weu-01)
Authentication	Service Principal with federated OIDC (never secrets in code)
Approval Process	PR to main → Plan → Approval → Apply
##### 🖥️ Step 2: Engineer's First Day - Setup the Backend
The senior engineer creates the remote state infrastructure (this is the only manual step):

```
# Log into Azure (using company Entra ID)
az login --tenant "company.onmicrosoft.com"

# Create the state storage (once, manually)
az group create --name "rg-platform-tfstate" --location "westeurope"
az storage account create \
  --name "stcompanytfp01" \
  --resource-group "rg-platform-tfstate" \
  --sku "Standard_ZRS" \
  --min-tls-version "TLS1_2" \
  --allow-blob-public-access false
az storage container create --name "tfstate" --account-name "stcompanytfp01"
```

##### 📁 Step 3: Repository Bootstrap
The engineer creates this exact folder structure:

```
terraform-azure-company/
├── .github/
│   └── workflows/          # CI/CD (Step 6)
├── environments/
│   ├── dev/
│   │   ├── dev.tfvars      # Dev-specific values
│   │   └── providers.tf    # Backend config
│   ├── staging/
│   └── prod/
└── infrastructure/         # Shared infrastructure (not app code)
    └── networking/
```
### Architecture Diagram
```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              AZURE TENANT (company.onmicrosoft.com)                    │
│                                                                                        │
│  ┌─────────────────────────────────────┐      ┌─────────────────────────────────────┐  │
│  │   MANAGEMENT SUBSCRIPTION           │      │   IDENTITY SUBSCRIPTION             │  │
│  │   (Platform Shared Services)        │      │   (Security & Secrets)              │  │
│  │                                     │      │                                     │  │
│  │  ┌─────────────────────────────┐    │      │  ┌─────────────────────────────┐    │  │
│  │  │  Log Analytics Workspace    │    │      │  │  Key Vault                   │   │  │
│  │  │  Automation Account         │    │      │  │  (Private Endpoint)         │    │  │
│  │  └─────────────────────────────┘    │      │  └─────────────────────────────┘    │  │
│  └─────────────────────────────────────┘      └─────────────────────────────────────┘  │
│                                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                    CONNECTIVITY SUBSCRIPTION (Hub)                               │  │
│  │                                                                                  │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐    │  │
│  │  │                         Hub VNet (10.0.0.0/16)                           │    │  │
│  │  │                                                                          │    │  │
│  │  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                │    │  │
│  │  │  │ Subnet:      │    │ Subnet:      │    │ Subnet:      │                │    │  │
│  │  │  │ Firewall     │    │ Bastion      │    │ Gateway      │                │    │  │
│  │  │  │ 10.0.1.0/24  │    │ 10.0.2.0/24  │    │ 10.0.3.0/24  │                │    │  │
│  │  │  └──────┬───────┘    └──────────────┘    └──────────────┘                │    │  │
│  │  │         │                                                                │    │  │
│  │  │  ┌──────▼────────────────────────────────────────────────────────┐       │    │  │
│  │  │  │                    Azure Firewall                             │       │    │  │
│  │  │  │  • Threat Intelligence                                        │       │    │  │
│  │  │  │  • Application Rules (HTTPS filtering)                        │       │    │  │
│  │  │  │  • Network Rules (DNS/NTP)                                    │       │    │  │
│  │  │  │  • Public IP: pip-company-....                                │       │    │  │
│  │  │  └───────────────────────────────────────────────────────────────┘       │    │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘    │  │
│  │                                                                                  │  │
│  │         ▲ Peering with forwarded traffic        ▲ Peering with forwarded traffic │  │
│  │         │                                        │                               │  │
│  └─────────┼────────────────────────────────────────┼───────────────────────────────┘  │
│            │                                        │                                  │
│            │                                        │                                  │
│  ┌─────────┼────────────────────────────────────────┼──────────────────────────────┐   │
│  │         ▼                                        │                              │   │
│  │  LANDING ZONE SUBSCRIPTION (Spoke - App Team A)  │  LANDING ZONE SUBSCRIPTION ..│   │
│  │                                                  │                              │   │
│  │  ┌────────────────────────────────────────────┐  │                               │   │
│  │  │     Spoke VNet A (10.1.0.0/16)             │  │                                │   │
│  │  │                                            │  │                               │   │
│  │  │  ┌────────────────────────────────────┐    │  │                                │   │
│  │  │  │ Subnet: Workload (10.1.1.0/24)     │    │  │                                │   │
│  │  │  │                                    │    │  │                                │   │
│  │  │  │  ┌─────┐  ┌─────┐  ┌─────────┐     │    │  │                                │  │
│  │  │  │  │ VM  │  │ AKS │  │ App Svc │     │    │  │                                │  │
│  │  │  │  └──┬──┘  └──┬──┘  └────┬────┘     │    │  │                                │  │
│  │  │  │     │        │          │          │   │ │                                │  │
│  │  │  │     └────────┼──────────┘          │   │ │                                │  │
│  │  │  │              │                     │   │ │                                │  │
│  │  │  │     Route Table forces ALL         │   │ │                                │  │
│  │  │  │     egress (0.0.0.0/0) to          │   │ │                                │  │
│  │  │  │     Firewall's private IP          │   │ │                                │  │
│  │  │  └──────────────┼─────────────────────┘   │ │                                │  │
│  │  └─────────────────┼─────────────────────────┘ │                                │  │
│  └────────────────────┼───────────────────────────┘                                │  │
│                       │                                                            │  │
│              Traffic flows through Firewall                                         │  │
│              for inspection & logging                                              │  │
│                                                                                    │  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │  │
│  │                          CI/CD PIPELINE (GitHub Actions)                    │   │  │
│  │                                                                              │   │  │
│  │  GitHub Repo ──► OIDC Auth ──► terraform plan ──► Review ──► terraform apply│   │  │
│  │                                                                              │   │  │
│  └──────────────────────────────────────────────────────────────────────────────┘   │  │
└─────────────────────────────────────────────────────────────────────────────────────┘  │

                                    INTERNET
                                        ▲
                                        │
                              Filtered by Firewall
                                        │
                          ┌─────────────┴─────────────┐
                          │                           │
                    ┌─────▼─────┐               ┌─────▼─────┐
                    │ GitHub    │               │ Azure APIs│
                    │ *.github  │               │ *.azure   │
                    └───────────┘               └───────────┘
```
##### 📋 Visual Blueprint (Text Table)
Layer	Component	Purpose	Our Status
Day 1	Remote State (Storage Account)	Team collaboration, locking	✅ Done
Day 1	Hub VNet + Subnets	Network foundation	✅ Done
Day 2	Azure Firewall + Policy	Security inspection	✅ Done
Day 2	Route Table + Peering	Force traffic through firewall	✅ Done
Day 2	Spoke VNet	Application isolation	✅ Done
Day 3	Private Endpoints (Key Vault, Storage)	No public internet exposure	🚧 Next
Day 3	AKS or VM in Spoke	Actual workload	🚧 Next
Day 3	GitHub Actions CI/CD	Automated deployments	🚧 Next
Day 4	Azure Policy	Enforce company standards	🔮 Future
Day 4	Diagnostic Logging	Firewall analytics	🔮 Future

##### 🔄 Data Flow (What happens when a user visits the app)
```
1. User request ──► Internet ──► Azure Firewall Public IP (inspect inbound)
                          │
2. Firewall rules check ──► Allowed? 
                          │
3. ┌───────────────────────┘
   ▼
4. Forward to Internal Load Balancer (Spoke subnet)
   │
5. Load Balancer ──► AKS Pod / VM
   │
6. App needs to call GitHub API (outbound)
   │
7. Route table (0.0.0.0/0) ──► Firewall private IP
   │
8. Firewall checks app rule "AllowGithub" ──► Yes
   │
9. Request leaves from Firewall public IP
   │
10. Response returns via same path
```
##### 📁 What Your Terraform Code Controls (Colored by Layer)
```
infrastructure/
├── networking/
│   ├── main.tf           # 🟦 Hub VNet, Firewall, Route Table
│   ├── firewall_rules.tf # 🟥 Security rules (allow/deny)
│   ├── spoke.tf          # 🟩 Application VNets
│   └── peering.tf        # 🔗 Hub-Spoke connections
├── security/
│   ├── keyvault.tf       # 🔐 Private endpoint for secrets
│   └── privatelinks.tf   # 🔒 PaaS service isolation
├── workload/
│   ├── aks.tf            # ☸️ Kubernetes cluster in spoke
│   └── monitoring.tf     # 📊 Log Analytics
└── pipeline/
    └── github.tf         # ⚙️ OIDC + workflow config
```


##### 🔧 Step 4: The First Terraform File
Create environments/dev/providers.tf:
```
hcl
terraform {
  required_version = ">=1.5.0"
  
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate"
    storage_account_name = "stcompanytfp01"
    container_name       = "tfstate"
    key                  = "infrastructure/dev/network.tfstate"
    use_oidc             = true  # No secrets - uses GitHub/Azure DevOps identity
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>4.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true  # Enterprise protection
    }
  }
}
Create infrastructure/networking/main.tf (first real resource):

hcl
variable "environment" {
  description = "dev/staging/prod"
  type        = string
}

variable "location" {
  default = "westeurope"
}

locals {
  prefix = "comp-${var.environment}"
  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    CostCenter  = "Platform"
    Project     = "LandingZone"
  }
}

# First resource: Resource Group
resource "azurerm_resource_group" "network" {
  name     = "rg-${local.prefix}-network-01"
  location = var.location
  tags     = local.tags
}

# Second: Hub VNet
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-${local.prefix}-hub-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  address_space       = ["10.0.0.0/16"]
  tags                = local.tags
}

# Third: Subnet for Firewall
resource "azurerm_subnet" "firewall" {
  name                 = "snet-${local.prefix}-firewall-01"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Outputs for other teams
output "hub_vnet_id" {
  value = azurerm_virtual_network.hub.id
}
```
##### 🔐 Step 5: Service Principal with OIDC
The security team creates an identity that never has a password:

``` 
# Create Service Principal
az ad app create --display-name "tf-company-platform"

# Create federated credential for GitHub
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/applications/{app-id}/federatedIdentityCredentials" \
  --body '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:company/terraform-azure:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Assign RBAC to the storage account
az role assignment create \
  --assignee "{sp-client-id}" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{sub-id}/resourceGroups/rg-platform-tfstate"
``` 
##### 🚀 Step 6: The First Deployment
The engineer runs the first deploy manually (CI/CD comes later):

```
cd environments/dev

# Initialize (downloads providers, sets up backend)
terraform init

# Validate syntax
terraform validate

# Plan - what will change?
terraform plan -var-file="dev.tfvars" -out=tfplan

# Review the plan output CAREFULLY
terraform show tfplan

# Apply (only after senior review)
terraform apply tfplan
```
✅ What "Start as company starts" Looks Like After Day 1
Completed	Not Yet Done
Remote state in secure storage	CI/CD pipeline
OIDC authentication (no secrets)	Modules (reusability)
Naming convention enforced	Multiple subscriptions
Prevent deletion protection	Policy as Code (Azure Policy)
First VNet + Subnet	Application landing zones
🎯 Your Action Items Right Now
Run the manual state setup (Step 2) in your own Azure trial subscription

Create the repository structure (Step 3)

Write the providers.tf and main.tf (Step 4) - change company to your initials

Run terraform init then plan - verify it works before adding complexity

### 🔥 Day 2: Add Azure Firewall + Route Tables
First, let's expand your infrastructure/networking/main.tf with enterprise-grade networking:
```
# Add to existing main.tf - Azure Firewall resources

# Public IP for Firewall
resource "azurerm_public_ip" "firewall" {
  name                = "pip-${local.prefix}-firewall-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]  # Availability Zones for SLA
  tags                = local.tags
}

# Firewall Policy (central rules management)
resource "azurerm_firewall_policy" "main" {
  name                     = "fwpolicy-${local.prefix}-01"
  location                 = azurerm_resource_group.network.location
  resource_group_name      = azurerm_resource_group.network.name
  sku                      = "Standard"
  threat_intelligence_mode = "Alert"  # Alert on known bad IPs
  
  dns {
    proxy_enabled = true
    servers       = ["168.63.129.16"]  # Azure DNS
  }
  
  tags = local.tags
}

# Firewall instance
resource "azurerm_firewall" "main" {
  name                = "afw-${local.prefix}-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  
  firewall_policy_id = azurerm_firewall_policy.main.id
  
  ip_configuration {
    name                 = "configuration"
    subnet_id           = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.firewall.id
  }
  
  tags = local.tags
  
  depends_on = [azurerm_subnet.firewall]
}

# Route Table to force all traffic to Firewall
resource "azurerm_route_table" "spoke_to_firewall" {
  name                = "rt-${local.prefix}-spoke-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  
  route {
    name                   = "DefaultRouteToFirewall"
    address_prefix         = "0.0.0.0/0"  # ALL internet traffic
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = azurerm_firewall.main.ip_configuration[0].private_ip_address
  }
  
  # Route for hub-to-hub traffic (keep it internal)
  route {
    name                   = "HubToHub"
    address_prefix         = "10.0.0.0/16"
    next_hop_type          = "None"  # Stay within hub
  }
  
  tags = local.tags
}

# Output the firewall private IP for route tables
output "firewall_private_ip" {
  value = azurerm_firewall.main.ip_configuration[0].private_ip_address
  sensitive = true
}
```
#### 🚦 Add Firewall Rules (Application Teams request these)
Create infrastructure/networking/firewall_rules.tf:
```
# Application Rule Collection - Allow outbound HTTPS to specific sites
resource "azurerm_firewall_policy_application_rule_collection" "dev_allow" {
  name                = "app_rule_collection_dev"
  firewall_policy_id  = azurerm_firewall_policy.main.id
  priority            = 100
  action              = "Allow"
  
  rule {
    name = "AllowGithub"
    
    protocols {
      type = "Https"
      port = 443
    }
    
    source_addresses = ["10.0.0.0/8"]
    destination_fqdns = [
      "github.com",
      "*.github.com",
      "api.github.com"
    ]
  }
  
  rule {
    name = "AllowAzureAPIs"
    
    protocols {
      type = "Https"
      port = 443
    }
    
    source_addresses = ["10.0.0.0/8"]
    destination_fqdns = [
      "*.azure.com",
      "*.azure-api.net",
      "*.azmk8s.io"
    ]
  }
}

# Network Rule Collection - Allow DNS and NTP
resource "azurerm_firewall_policy_network_rule_collection" "infrastructure" {
  name                = "net_rule_collection_infra"
  firewall_policy_id  = azurerm_firewall_policy.main.id
  priority            = 100
  action              = "Allow"
  
  rule {
    name = "AllowDNS"
    protocols = ["UDP"]
    source_addresses = ["10.0.0.0/8"]
    destination_addresses = ["168.63.129.16"]
    destination_ports = ["53"]
  }
  
  rule {
    name = "AllowNTP"
    protocols = ["UDP"]
    source_addresses = ["10.0.0.0/8"]
    destination_addresses = ["*"]
    destination_ports = ["123"]
  }
}

```
#### 🏗️ Create a Spoke VNet (for actual workloads)
Create infrastructure/networking/spoke.tf:
```
# Spoke VNet for applications
resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-${local.prefix}-spoke-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  address_space       = ["10.1.0.0/16"]
  tags                = local.tags
}

# Subnet for AKS or VMs
resource "azurerm_subnet" "workload" {
  name                 = "snet-${local.prefix}-workload-01"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = ["10.1.1.0/24"]
}

# Associate route table to spoke subnet (force tunnel through firewall)
resource "azurerm_subnet_route_table_association" "spoke" {
  subnet_id      = azurerm_subnet.workload.id
  route_table_id = azurerm_route_table.spoke_to_firewall.id
}

# VNet Peering: Spoke -> Hub
resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                         = "peer-spoke-to-hub"
  resource_group_name          = azurerm_resource_group.network.name
  virtual_network_name         = azurerm_virtual_network.spoke.name
  remote_virtual_network_id    = azurerm_virtual_network.hub.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true  # CRITICAL: allows route table to work
}

# VNet Peering: Hub -> Spoke
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  name                         = "peer-hub-to-spoke"
  resource_group_name          = azurerm_resource_group.network.name
  virtual_network_name         = azurerm_virtual_network.hub.name
  remote_virtual_network_id    = azurerm_virtual_network.spoke.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}

# Output spoke network info for app teams
output "spoke_vnet_id" {
  value = azurerm_virtual_network.spoke.id
}

output "workload_subnet_id" {
  value = azurerm_subnet.workload.id
}
```
#### 🧪 Test the Setup
Deploy and verify traffic flows through firewall:
```
cd environments/dev

# Deploy the updated networking
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"

# Deploy a test VM to spoke subnet to verify routing
# (Add this temporarily - will remove after test)

```
Add this test VM to infrastructure/networking/test_vm.tf (temporary):

```
# TEMPORARY TEST VM - Remove after verifying firewall routing
resource "azurerm_network_interface" "test" {
  name                = "nic-${local.prefix}-test-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.workload.id
    private_ip_address_allocation = "Dynamic"
  }
}

output "test_vm_nic_id" {
  value = azurerm_network_interface.test.id
}
```
#### 📊 What This Enforces
Traffic Type	Without Firewall	With Firewall
VM → Internet	Direct egress	Forced through inspection
VM → Azure APIs	Direct	Logged and filtered
VM → GitHub	Allowed (rule)	Only to *.github.com
VM → Malicious IP	Allowed	Blocked by threat intel
🚨 Common Enterprise Issues You'll Hit
Terraform timeout on Firewall creation: Firewall takes 30-45 minutes. Add timeouts { create = "60m" }

Route table doesn't force traffic: Check allow_forwarded_traffic = true on peering

Dev wants to bypass firewall: Security team requires exception ticket, then add specific rule

✅ Day 2 Summary
You now have:

✅ Firewall with threat intelligence

✅ Route table forcing all egress through firewall

✅ Spoke VNet ready for application teams

✅ Central control over outbound traffic


### 🔒 Day 3A: Private Endpoints for All PaaS Services
Create infrastructure/security/private_endpoints.tf:
```
# Create a dedicated subnet for Private Endpoints in the Spoke VNet
resource "azurerm_subnet" "private_endpoints" {
  name                 = "snet-${local.prefix}-private-endpoints-01"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = ["10.1.2.0/27"]  # Small subnet, only for ENIs
  
  # Prevent service policies from blocking Private Link
  private_endpoint_network_policies_enabled = false
}

# 1. KEY VAULT with Private Endpoint
resource "azurerm_key_vault" "platform" {
  name                       = "kv-${local.prefix}-platform-01"
  location                   = azurerm_resource_group.network.location
  resource_group_name        = azurerm_resource_group.network.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  
  # CRITICAL: Block public access
  public_network_access_enabled = false
  
  # Allow trusted Microsoft services (for backup, logging)
  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }
  
  # Enable purge protection for compliance
  purge_protection_enabled = true
  soft_delete_retention_days = 7
  
  tags = local.tags
}

# Private endpoint for Key Vault
resource "azurerm_private_endpoint" "keyvault" {
  name                = "pe-${local.prefix}-keyvault-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-keyvault"
    private_connection_resource_id = azurerm_key_vault.platform.id
    is_manual_connection           = false
    subresource_names              = ["vault"]
  }
  
  private_dns_zone_group {
    name                 = "keyvault-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.keyvault.id]
  }
  
  tags = local.tags
}

# Private DNS Zone for Key Vault (automatically resolves private IP)
resource "azurerm_private_dns_zone" "keyvault" {
  name                = "privatelink.vaultcore.azure.net"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  
  tags = local.tags
}

# Link DNS zone to Hub VNet (so resources in hub can resolve)
resource "azurerm_private_dns_zone_virtual_network_link" "keyvault_hub" {
  name                  = "link-keyvault-to-hub"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.keyvault.name
  virtual_network_id    = azurerm_virtual_network.hub.id
  registration_enabled  = false
}

# Link DNS zone to Spoke VNet
resource "azurerm_private_dns_zone_virtual_network_link" "keyvault_spoke" {
  name                  = "link-keyvault-to-spoke"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.keyvault.name
  virtual_network_id    = azurerm_virtual_network.spoke.id
  registration_enabled  = false
}

# 2. STORAGE ACCOUNT for app data (blobs, tables)
resource "azurerm_storage_account" "app_data" {
  name                     = "st${replace(local.prefix, "-", "")}data01"
  location                 = azurerm_resource_group.network.location
  resource_group_name      = azurerm_resource_group.network.name
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  # BLOCK PUBLIC ACCESS
  public_network_access_enabled = false
  
  # Require HTTPS
  enable_https_traffic_only = true
  
  # Minimum TLS 1.2
  min_tls_version = "TLS1_2"
  
  # Allow trusted Azure services
  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }
  
  tags = local.tags
}

# Private endpoint for Storage Account (blob)
resource "azurerm_private_endpoint" "storage_blob" {
  name                = "pe-${local.prefix}-storage-blob-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-storage-blob"
    private_connection_resource_id = azurerm_storage_account.app_data.id
    is_manual_connection           = false
    subresource_names              = ["blob"]
  }
  
  private_dns_zone_group {
    name                 = "storage-blob-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.storage_blob.id]
  }
  
  tags = local.tags
}

# Private DNS Zone for Storage Blob
resource "azurerm_private_dns_zone" "storage_blob" {
  name                = "privatelink.blob.core.windows.net"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  
  tags = local.tags
}

# Link storage DNS to Hub and Spoke
resource "azurerm_private_dns_zone_virtual_network_link" "storage_blob_hub" {
  name                  = "link-storage-to-hub"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.storage_blob.name
  virtual_network_id    = azurerm_virtual_network.hub.id
  registration_enabled  = false
}

resource "azurerm_private_dns_zone_virtual_network_link" "storage_blob_spoke" {
  name                  = "link-storage-to-spoke"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.storage_blob.name
  virtual_network_id    = azurerm_virtual_network.spoke.id
  registration_enabled  = false
}

# 3. CONTAINER REGISTRY (for AKS images)
resource "azurerm_container_registry" "platform" {
  name                = "cr${replace(local.prefix, "-", "")}01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  sku                 = "Premium"  # Required for private endpoints
  admin_enabled       = false      # Use Entra ID only
  
  # Block public access
  public_network_access_enabled = false
  
  # Allow trusted services
  network_rule_set {
    default_action = "Deny"
  }
  
  tags = local.tags
}

# Private endpoint for Container Registry
resource "azurerm_private_endpoint" "acr" {
  name                = "pe-${local.prefix}-acr-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-acr"
    private_connection_resource_id = azurerm_container_registry.platform.id
    is_manual_connection           = false
    subresource_names              = ["registry"]
  }
  
  private_dns_zone_group {
    name                 = "acr-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.acr.id]
  }
  
  tags = local.tags
}

# Private DNS Zone for ACR
resource "azurerm_private_dns_zone" "acr" {
  name                = "privatelink.azurecr.io"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  
  tags = local.tags
}

resource "azurerm_private_dns_zone_virtual_network_link" "acr_hub" {
  name                  = "link-acr-to-hub"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.hub.id
  registration_enabled  = false
}

resource "azurerm_private_dns_zone_virtual_network_link" "acr_spoke" {
  name                  = "link-acr-to-spoke"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.spoke.id
  registration_enabled  = false
}

# Data source for tenant ID
data "azurerm_client_config" "current" {}

# Output the private endpoints for reference
output "key_vault_private_endpoint_ip" {
  value = azurerm_private_endpoint.keyvault.private_service_connection[0].private_ip_address
  sensitive = true
}

output "storage_private_endpoint_ip" {
  value = azurerm_private_endpoint.storage_blob.private_service_connection[0].private_ip_address
}

output "acr_private_endpoint_ip" {
  value = azurerm_private_endpoint.acr.private_service_connection[0].private_ip_address
}

# Verify private connectivity (optional - add to outputs)
output "key_vault_uri" {
  value = azurerm_key_vault.platform.vault_uri
  sensitive = true
}
```
🧪 Test Private Endpoint Connectivity
Create scripts/test_private_connectivity.sh:
```
#!/bin/bash

# Deploy a temporary jumpbox VM in the spoke subnet to test
# (You'll need to deploy this VM via Terraform or Azure CLI)

# Get the private IP of Key Vault private endpoint
KV_PRIVATE_IP=$(terraform output -raw key_vault_private_endpoint_ip)

# Test DNS resolution from within the spoke VNet
echo "Testing DNS resolution for Key Vault:"
nslookup $(terraform output -raw key_vault_uri | sed 's|https://||' | sed 's|/||') 
# Should return the private IP, not public IP

# Test network connectivity
echo "Testing connectivity to Key Vault private endpoint:"
nc -zv $KV_PRIVATE_IP 443

# Test authentication (requires Azure CLI on jumpbox)
echo "Testing Key Vault access:"
az keyvault secret show --name "test-secret" --vault-name $(terraform output -raw key_vault_uri | sed 's|https://||' | sed 's|/||')
```
📊 What Changed After Private Endpoints
Before (Day 2)	After (Day 3A)
Key Vault accessible via public endpoint	Only accessible via private IP

###### 🔐 Security Team Validation
Add infrastructure/security/validation.tf to verify security posture:
```
# Validate no public endpoints exist
resource "terraform_data" "validate_no_public_endpoints" {
  provisioner "local-exec" {
    command = <<-EOT
      echo "Validating no public endpoints..."
      
      # Check Key Vault public access
      PUBLIC_ACCESS=$(az keyvault show --name ${azurerm_key_vault.platform.name} --query "properties.publicNetworkAccess" -o tsv)
      if [ "$PUBLIC_ACCESS" != "Disabled" ]; then
        echo "❌ Key Vault has public access enabled!"
        exit 1
      fi
      
      # Check Storage Account public access
      STORAGE_ACCESS=$(az storage account show --name ${azurerm_storage_account.app_data.name} --query "publicNetworkAccess" -o tsv)
      if [ "$STORAGE_ACCESS" != "Disabled" ]; then
        echo "❌ Storage Account has public access enabled!"
        exit 1
      fi
      
      echo "✅ All PaaS services have public access disabled"
    EOT
  }
  
  depends_on = [
    azurerm_key_vault.platform,
    azurerm_storage_account.app_data
  ]
}
```
##### 🎯 Day 3A Complete Checklist
✅ Private endpoint subnet created in Spoke VNet

✅ Key Vault deployed with public access disabled

✅ Private DNS zones created and linked to Hub + Spoke

✅ Storage Account with private endpoint (blob)

✅ Container Registry with private endpoint

✅ Validation script confirms no public access

✅ All traffic to PaaS goes through Microsoft backbone

✅ Logging capability (add Diagnostic Settings next)

### 🚀 Deploy Day 3A
```
cd environments/dev

# Plan to see what will change
terraform plan -var-file="dev.tfvars"

# Apply (will take 5-10 minutes)
terraform apply -var-file="dev.tfvars"

# Verify private endpoints
terraform output key_vault_uri
terraform output key_vault_private_endpoint_ip

# The key vault URL will ONLY resolve inside your VNet now
```
⚠️ Common Issues & Fixes
Issue	                                      Solution
Cannot access Key Vault from AzurePortal	Portal accesses from public internet. Use Azure Bastion + VM inside VNet
DNS returns public IP	                    Private DNS zone not linked to your VNet. Check links
Private endpoint stuck "Updating"	        Delete and recreate. Sometimes takes 2 attempts
ACR Premium SKU required	                Premium is $/day - cheaper than data exfiltratio

### ☸️ Day 3B: AKS Cluster Behind Firewall with Private Endpoints
Create infrastructure/workload/aks.tf:
```
# Azure Kubernetes Service Cluster - Enterprise Grade
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${local.prefix}-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  dns_prefix          = "aks${replace(local.prefix, "-", "")}"
  
  # PRIVATE CLUSTER - No public endpoint at all
  private_cluster_enabled = true
  
  # Use local account for initial setup (will integrate Entra ID later)
  local_account_disabled = false
  
  # Role-based access control
  role_based_access_control_enabled = true
  
  # Network configuration - MUST use Azure CNI for forced tunneling
  network_profile {
    network_plugin = "azure"
    network_policy = "azure"
    outbound_type  = "userDefinedRouting"  # CRITICAL: Use route table to firewall
  }
  
  # Default node pool
  default_node_pool {
    name                 = "system"
    vm_size             = "Standard_D4s_v5"  # 4 vCPU, 16GB RAM - enterprise minimum
    vnet_subnet_id      = azurerm_subnet.workload.id
    enable_auto_scaling = true
    min_count           = 2
    max_count           = 5
    node_count          = 2
    
    # System pool should be on dedicated nodes
    node_labels = {
      "nodepool-type" = "system"
      "environment"   = var.environment
    }
    
    node_taints = ["CriticalAddonsOnly=true:NoSchedule"]
    
    # Max pods per node (Azure CNI limit)
    max_pods = 30
    
    # Availability Zones for high availability
    zones = ["1", "2", "3"]
  }
  
  # User node pool for workloads (separate from system)
  # Will be added after cluster creation
  
  # Identity for AKS to access other resources
  identity {
    type = "SystemAssigned"
  }
  
  # Monitor container insights
  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }
  
  # Private container registry integration
  private_cluster_public_fqdn_enabled = false
  
  # Maintenance window (avoid business hours)
  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [0, 1, 2, 3, 4, 5, 6]
    }
    
    not_allowed {
      start {
        date = "2024-12-24"
        time = "00:00"
      }
      end {
        date = "2024-12-26"
        time = "23:59"
      }
    }
  }
  
  tags = local.tags
  
  depends_on = [
    azurerm_route_table.spoke_to_firewall,
    azurerm_subnet_route_table_association.spoke
  ]
}

# User node pool for application workloads
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size              = "Standard_D8s_v5"  # 8 vCPU, 32GB RAM for apps
  node_count           = 2
  enable_auto_scaling  = true
  min_count            = 2
  max_count            = 10
  vnet_subnet_id       = azurerm_subnet.workload.id
  
  node_labels = {
    "nodepool-type" = "user"
    "environment"   = var.environment
  }
  
  node_taints = []
  
  zones = ["1", "2", "3"]
  
  # Allow scheduling of pods on this pool
  max_pods = 30
  
  tags = local.tags
}

# Create ACR pull role for AKS
resource "azurerm_role_assignment" "aks_to_acr" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.platform.id
  skip_service_principal_aad_check = true
}

# Role assignment for AKS to read Key Vault secrets
resource "azurerm_role_assignment" "aks_to_keyvault" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "Key Vault Secrets User"
  scope                            = azurerm_key_vault.platform.id
  skip_service_principal_aad_check = true
}
```
##### 📊 Create Log Analytics Workspace
Create infrastructure/workload/monitoring.tf:
```
# Log Analytics Workspace for AKS and Firewall logs
resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-${local.prefix}-01"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  sku                 = "PerGB2018"
  retention_in_days   = 30  # Compliance minimum, adjust based on needs
  
  tags = local.tags
}

# Diagnostic settings for AKS to send logs to Log Analytics
resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                       = "diag-aks-to-law"
  target_resource_id         = azurerm_kubernetes_cluster.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace_main.id
  
  # Send all logs and metrics
  enabled_log {
    category = "kube-apiserver"
  }
  enabled_log {
    category = "kube-controller-manager"
  }
  enabled_log {
    category = "kube-scheduler"
  }
  enabled_log {
    category = "cluster-autoscaler"
  }
  enabled_log {
    category = "guard"
  }
  enabled_log {
    category = "csi-azuredisk-controller"
  }
  enabled_log {
    category = "csi-azurefile-controller"
  }
  
  metric {
    category = "AllMetrics"
    enabled  = true
  }
}

# Diagnostic settings for Firewall logs
resource "azurerm_monitor_diagnostic_setting" "firewall" {
  name                       = "diag-firewall-to-law"
  target_resource_id         = azurerm_firewall.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  
  enabled_log {
    category = "AzureFirewallApplicationRule"
  }
  enabled_log {
    category = "AzureFirewallNetworkRule"
  }
  enabled_log {
    category = "AzureFirewallDnsProxy"
  }
  
  metric {
    category = "AllMetrics"
    enabled  = true
  }
}
```
##### 🔐 Store Kubernetes Secrets in Key Vault
Create infrastructure/workload/k8s_secrets.tf:
```
# Sample secret for AKS to use (database password, API keys, etc.)
resource "azurerm_key_vault_secret" "sample_app_secret" {
  name         = "sample-app-password"
  value        = random_password.app_secret.result
  key_vault_id = azurerm_key_vault.platform.id
  
  tags = local.tags
}

# Random password generator for demo
resource "random_password" "app_secret" {
  length  = 24
  special = true
}

# Provider for Kubernetes (to deploy the SecretProviderClass)
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.main.kube_config[0].host
  client_key             = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_key)
  client_certificate     = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_certificate)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].cluster_ca_certificate)
}

# SecretProviderClass to mount Key Vault secrets into pods
resource "kubernetes_manifest" "secret_provider_class" {
  manifest = {
    apiVersion = "secrets-store.csi.x-k8s.io/v1"
    kind       = "SecretProviderClass"
    metadata = {
      name      = "azure-kvname"
      namespace = "default"
    }
    spec = {
      provider = "azure"
      parameters = {
        usePodIdentity = "false"
        useVMManagedIdentity = "true"
        userAssignedIdentityID = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
        keyvaultName = azurerm_key_vault.platform.name
        objects = jsonencode({
          array = [
            {
              objectName = azurerm_key_vault_secret.sample_app_secret.name
              objectType = "secret"
              objectVersion = ""
            }
          ]
        })
        tenantId = data.azurerm_client_config.current.tenant_id
      }
    }
  }
  
  depends_on = [azurerm_kubernetes_cluster.main]
}
```
##### 🧪 Deploy a Test Application
Create infrastructure/workload/test_app.tf:
```
# Namespace for test app
resource "kubernetes_namespace" "test" {
  metadata {
    name = "test-app"
  }
  
  depends_on = [azurerm_kubernetes_cluster.main]
}

# Sample deployment that pulls from ACR and uses Key Vault secret
resource "kubernetes_deployment" "test_nginx" {
  metadata {
    name      = "test-nginx"
    namespace = kubernetes_namespace.test.metadata[0].name
    labels = {
      app = "test-nginx"
    }
  }
  
  spec {
    replicas = 2
    
    selector {
      match_labels = {
        app = "test-nginx"
      }
    }
    
    template {
      metadata {
        labels = {
          app = "test-nginx"
        }
      }
      
      spec {
        # Use managed identity for authentication
        node_selector = {
          "nodepool-type" = "user"
        }
        
        container {
          image = "nginx:latest"  # Replace with your private image
          name  = "nginx"
          
          port {
            container_port = 80
          }
          
          # Mount environment variables from Key Vault
          env {
            name = "APP_PASSWORD"
            value_from {
              secret_key_ref {
                name = "app-secrets"
                key  = "password"
              }
            }
          }
        }
        
        # Volume for Key Vault secrets
        volume {
          name = "secrets-store"
          csi {
            driver    = "secrets-store.csi.k8s.io"
            read_only = true
            volume_attributes = {
              secretProviderClass = kubernetes_manifest.secret_provider_class.manifest.metadata.name
            }
          }
        }
      }
    }
  }
}

# Service to expose the app internally
resource "kubernetes_service" "test_nginx" {
  metadata {
    name      = "test-nginx"
    namespace = kubernetes_namespace.test.metadata[0].name
  }
  
  spec {
    selector = {
      app = "test-nginx"
    }
    
    port {
      port        = 80
      target_port = 80
    }
    
    type = "ClusterIP"
  }
}

# Output the service endpoint
output "test_app_service_ip" {
  value = kubernetes_service.test_nginx.spec[0].cluster_ip
}
```
##### 📝 Add Required Providers
Update environments/dev/providers.tf to include Kubernetes:
```
terraform {
  required_version = ">=1.5.0"
  
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate"
    storage_account_name = "stcompanytfp01"
    container_name       = "tfstate"
    key                  = "infrastructure/dev/network.tfstate"
    use_oidc             = true
  }
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~>2.30"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.6"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
}
```
#### 🚀 Deploy AKS
```
cd environments/dev

# Initialize with new providers
terraform init

# Plan AKS deployment
terraform plan -var-file="dev.tfvars"

# Apply AKS (WARNING: Takes 15-20 minutes)
terraform apply -var-file="dev.tfvars"

# Get kubeconfig
az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw aks_name)

# Verify nodes
kubectl get nodes

# Verify pods in test namespace
kubectl get pods -n test-app

# Check if secrets are mounted
kubectl exec -it -n test-app $(kubectl get pods -n test-app -o jsonpath='{.items[0].metadata.name}') -- ls /mnt/secrets
```
#### 🛡️ Verify Forced Tunneling
```
# Deploy a test pod with networking tools
kubectl run nettest -it --rm --image=nicolaka/netshoot -- bash

# Inside the pod, check the route (should show 10.0.1.x firewall as default gateway)
ip route

# Test egress - should go through firewall
curl -v https://github.com

# Check firewall logs (in Azure portal or Log Analytics)
# You should see your pod's source IP (10.1.1.x) in the logs
```
#### 🎯 What's Happening Under the Hood
```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR COMPANY NETWORK                      │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  AKS Cluster (Private - No Public IP)               │    │
│  │                                                      │    │
│  │  ┌────────────┐        ┌────────────┐             │    │
│  │  │ Pod        │        │ Pod        │             │    │
│  │  │ (10.1.1.5) │        │ (10.1.1.6) │             │    │
│  │  └─────┬──────┘        └─────┬──────┘             │    │
│  │        │                      │                     │    │
│  │        └──────┬───────────────┘                     │    │
│  │               │                                      │    │
│  │    ┌──────────▼──────────┐                          │    │
│  │    │  Route Table        │                          │    │
│  │    │  0.0.0.0/0 → FW IP  │                          │    │
│  │    └──────────┬──────────┘                          │    │
│  └───────────────┼──────────────────────────────────────┘    │
│                  │                                            │
│         ┌────────▼────────┐                                  │
│         │ Azure Firewall  │◄──── All egress inspected        │
│         │ (10.0.1.4)      │                                  │
│         └────────┬────────┘                                  │
│                  │                                            │
│         ┌────────▼────────┐                                  │
│         │ Private DNS     │◄──── Resolves Key Vault          │
│         │ privatelink.*   │      to 10.1.2.x                 │
│         └─────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘

The public internet NEVER sees your AKS cluster or your secrets.
```
### 🔐 Day 3C: GitHub Actions OIDC Pipeline
First, create the GitHub OIDC identity in Azure:
 Create infrastructure/security/github_oidc.tf:
 ```
# Create Service Principal for GitHub OIDC
resource "azuread_application" "github_oidc" {
  display_name = "tf-github-oidc-${var.environment}"
  
  # No secrets - OIDC only
  web {
    redirect_uris = []
  }
}

resource "azuread_service_principal" "github_oidc" {
  client_id = azuread_application.github_oidc.client_id
}

# Create federated identity credential for GitHub
resource "azuread_application_federated_identity_credential" "github_main" {
  application_object_id = azuread_application.github_oidc.object_id
  display_name          = "github-main-branch"
  description           = "Deploy from GitHub main branch"
  audiences             = ["api://AzureADTokenExchange"]
  issuer                = "https://token.actions.githubusercontent.com"
  subject               = "repo:${var.github_repository}:ref:refs/heads/main"
}

# For pull requests (plan only, no apply)
resource "azuread_application_federated_identity_credential" "github_pr" {
  application_object_id = azuread_application.github_oidc.object_id
  display_name          = "github-pull-requests"
  description           = "Plan from GitHub PRs"
  audiences             = ["api://AzureADTokenExchange"]
  issuer                = "https://token.actions.githubusercontent.com"
  subject               = "repo:${var.github_repository}:pull_request"
}

# Assign Contributor role at subscription level (adjust scope as needed)
resource "azurerm_role_assignment" "github_contributor" {
  principal_id         = azuread_service_principal.github_oidc.object_id
  role_definition_name = "Contributor"
  scope                = "/subscriptions/${var.subscription_id}"
}

# Assign Key Vault Secrets User for accessing secrets
resource "azurerm_role_assignment" "github_keyvault" {
  principal_id         = azuread_service_principal.github_oidc.object_id
  role_definition_name = "Key Vault Secrets User"
  scope                = azurerm_key_vault.platform.id
}

# Output the client ID for GitHub
output "github_oidc_client_id" {
  value = azuread_application.github_oidc.client_id
  sensitive = true
}

output "github_oidc_tenant_id" {
  value = data.azurerm_client_config.current.tenant_id
  sensitive = true
}
```
##### 📁 Add GitHub Configuration Variables
Update environments/dev/dev.tfvars:
```
environment   = "dev"
location      = "westeurope"

# For OIDC configuration
github_repository = "your-company/terraform-azure"  # Change to your repo
subscription_id   = "00000000-0000-0000-0000-000000000000"  # Your subscription ID
```
#### 📂 Create GitHub Actions Workflow
Create .github/workflows/terraform.yml:
```
name: "Terraform Infrastructure"

on:
  pull_request:
    branches: [ main ]
    paths:
      - 'infrastructure/**'
      - 'environments/**'
      - 'modules/**'
  push:
    branches: [ main ]
    paths:
      - 'infrastructure/**'
      - 'environments/**'
      - 'modules/**'
  workflow_dispatch:  # Manual trigger

env:
  ARM_CLIENT_ID: "${{ secrets.AZURE_CLIENT_ID }}"
  ARM_SUBSCRIPTION_ID: "${{ secrets.AZURE_SUBSCRIPTION_ID }}"
  ARM_TENANT_ID: "${{ secrets.AZURE_TENANT_ID }}"
  ARM_USE_OIDC: true
  TERRAFORM_VERSION: "1.5.0"

permissions:
  id-token: write   # Required for OIDC
  contents: read
  pull-requests: write  # For PR comments

jobs:
  terraform-plan:
    name: "Terraform Plan"
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    environment: dev
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}
      
      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Terraform Init
        working-directory: ./environments/dev
        run: terraform init
      
      - name: Terraform Validate
        working-directory: ./environments/dev
        run: terraform validate
      
      - name: Terraform Plan
        id: plan
        working-directory: ./environments/dev
        run: |
          terraform plan -var-file="dev.tfvars" -out=tfplan
          
          # Format plan output for PR comment
          plan_output=$(terraform show -no-color tfplan)
          
          # Save plan output
          echo "plan_output<<EOF" >> $GITHUB_OUTPUT
          echo "$plan_output" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Comment PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const planOutput = `${{ steps.plan.outputs.plan_output }}`;
            const truncated = planOutput.length > 60000 ? 
              planOutput.substring(0, 60000) + "\n\n... (truncated)" : 
              planOutput;
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 📋 Terraform Plan Output\n\n\`\`\`\n${truncated}\n\`\`\``
            });
      
      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ./environments/dev/tfplan
          retention-days: 1

  terraform-apply:
    name: "Terraform Apply"
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: dev
    needs: terraform-plan  # Only run if plan succeeded
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}
      
      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Download Plan Artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ./environments/dev
      
      - name: Terraform Init
        working-directory: ./environments/dev
        run: terraform init
      
      - name: Terraform Apply
        working-directory: ./environments/dev
        run: |
          terraform apply tfplan
          
          # Capture outputs
          terraform output -json > outputs.json
      
      - name: Upload Outputs
        uses: actions/upload-artifact@v4
        with:
          name: terraform-outputs
          path: ./environments/dev/outputs.json
          retention-days: 7
      
      - name: Post Apply Summary
        run: |
          echo "## ✅ Terraform Apply Successful" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Environment:** dev" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "**Action Run:** ${{ github.run_id }}" >> $GITHUB_STEP_SUMMARY
```
##### 🔧 Add Environment-Specific Workflows
Create .github/workflows/terraform-prod.yml for production:
```
name: "Terraform Production"

on:
  workflow_dispatch:  # Manual trigger only
    inputs:
      version:
        description: 'Version to deploy'
        required: true
        type: string

permissions:
  id-token: write
  contents: read

jobs:
  terraform-plan-prod:
    name: "Plan Production"
    runs-on: ubuntu-latest
    environment: prod  # Requires approval
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.5.0"
      
      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Terraform Init
        working-directory: ./environments/prod
        run: terraform init
      
      - name: Terraform Plan
        id: plan
        working-directory: ./environments/prod
        run: |
          terraform plan -var-file="prod.tfvars" -out=tfplan
      
      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: prod-tfplan
          path: ./environments/prod/tfplan
  
  terraform-apply-prod:
    name: "Apply Production"
    runs-on: ubuntu-latest
    needs: terraform-plan-prod
    environment: 
      name: prod
      url: https://portal.azure.com
    permissions:
      id-token: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: prod-tfplan
          path: ./environments/prod
      
      - name: Terraform Apply
        working-directory: ./environments/prod
        run: terraform apply tfplan
```
#### 🛡️ Drift Detection Workflow
Create .github/workflows/drift-detection.yml:
```
name: "Drift Detection"

on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  detect-drift:
    name: "Detect Infrastructure Drift"
    runs-on: ubuntu-latest
    environment: dev
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Terraform Init
        working-directory: ./environments/dev
        run: terraform init
      
      - name: Refresh State
        id: refresh
        working-directory: ./environments/dev
        run: |
          terraform plan -refresh-only -detailed-exitcode -out=drift-tfplan
          exit_code=$?
          
          if [ $exit_code -eq 2 ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            echo "## ⚠️ Drift Detected!" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "Infrastructure has changed outside of Terraform." >> $GITHUB_STEP_SUMMARY
            echo "Run 'terraform apply refresh-only' to sync state." >> $GITHUB_STEP_SUMMARY
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
            echo "## ✅ No Drift Detected" >> $GITHUB_STEP_SUMMARY
            echo "All infrastructure matches Terraform state." >> $GITHUB_STEP_SUMMARY
          fi
      
      - name: Create Issue on Drift
        if: steps.refresh.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🚨 Infrastructure Drift Detected',
              body: 'Manual changes detected in the infrastructure. Please investigate and sync state.\n\nRun: `terraform apply -refresh-only`\n\nEnvironment: dev',
              labels: ['drift', 'bug']
            });



```
#### 📋 GitHub Repository Setup
Create a README.md in your repo root:
```
# Azure Terraform Infrastructure

## 🚀 Quick Start

### Prerequisites
- Azure subscription
- GitHub repository with OIDC configured

### First Time Setup

1. **Configure OIDC in Azure:**
```bash
# Apply OIDC configuration first (this requires manual Azure login)
az login
terraform apply -target=azuread_application.github_oidc
```
2. Add GitHub Secrets:
After first apply, add these to your GitHub repo (Settings → Secrets → Actions):

AZURE_CLIENT_ID - from terraform output github_oidc_client_id

AZURE_TENANT_ID - from terraform output github_oidc_tenant_id

AZURE_SUBSCRIPTION_ID - your subscription ID

3. Push to main:
```
git push origin main
```
##### Development Workflow
- Create a feature branch
- Make changes to Terraform code
- Open a Pull Request
- Review the Terraform Plan in PR comments
- Merge to main → Auto-apply

##### Manual Commands
```
# Plan with refresh only (detect drift)
terraform plan -refresh-only

# Apply local changes (emergency only)
terraform apply -var-file="dev.tfvars"

# Destroy (with caution!)
terraform destroy -var-file="dev.tfvars"


```
##### Architecture
Hub-Spoke networking with Azure Firewall

Private endpoints for all PaaS

AKS cluster with forced tunneling

GitHub Actions OIDC (no secrets!)
```

## 🔧 Create Environment Folders

Create `environments/prod/prod.tfvars`:

```hcl
environment   = "prod"
location      = "westeurope"
github_repository = "your-company/terraform-azure"

# Production-specific values
node_count_system   = 3
node_count_user     = 5
min_user_nodes      = 3
max_user_nodes      = 20
```
#### ✅ Deploy CI/CD Pipeline
```
# 1. First, apply OIDC configuration (manual initial deploy)
cd environments/dev

# Deploy OIDC identity first
terraform apply -target=azuread_application.github_oidc -target=azuread_service_principal.github_oidc -target=azuread_application_federated_identity_credential.github_main

# Get the OIDC values for GitHub secrets
terraform output github_oidc_client_id
terraform output github_oidc_tenant_id

# 2. Add these as secrets in GitHub:
# Go to: https://github.com/YOUR-ORG/YOUR-REPO/settings/secrets/actions
# Add:
# - AZURE_CLIENT_ID (from output)
# - AZURE_TENANT_ID (from output)  
# - AZURE_SUBSCRIPTION_ID (your sub ID)

# 3. Push to GitHub
git add .
git commit -m "Add CI/CD pipeline with OIDC"
git push origin main

# The workflow will automatically trigger and deploy everything!
```
####  📊 CI/CD Pipeline Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                     DEVELOPER WORKFLOW                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Create branch & make changes                                 │
│    git checkout -b feature/new-firewall-rule                   │
│    vi infrastructure/networking/firewall_rules.tf              │
│    git push origin feature/new-firewall-rule                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Open Pull Request to main                                    │
│    GitHub → Pull Requests → New                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. GitHub Actions runs:                                         │
│    ✓ Azure Login (OIDC - no secrets)                           │
│    ✓ terraform init                                            │
│    ✓ terraform validate                                        │
│    ✓ terraform plan                                            │
│    ✓ Comments PR with plan output                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Senior Engineer reviews PR + plan                           │
│    Sees what will change:                                      │
│    + azurerm_firewall_policy_application_rule_collection.dev  │
│    ~ azurerm_firewall.main                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Merge PR to main                                             │
│    Click "Merge pull request"                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. GitHub Actions runs apply:                                   │
│    ✓ terraform apply tfplan (from PR)                          │
│    ✓ Infrastructure updated                                    │
│    ✓ Posts summary to Actions run                             │
└─────────────────────────────────────────────────────────────────┘
```
##### 🔒 Security Enforcement
Add branch protection rules in GitHub:
```
# .github/branch-protection.yml (via GitHub UI)
- Require pull request reviews before merging (2 approvers)
- Dismiss stale reviews when new commits pushed
- Require status checks to pass (terraform-plan)
- Require conversation resolution before merging
- Require linear history
```
🎉 You've Built an Enterprise System!
Your project now has:

Day	Component	Enterprise Value
1	Remote State + Hub VNet	Team collaboration
2	Firewall + Forced Tunneling	Security compliance
3A	Private Endpoints	Data exfiltration protection
3B	AKS Cluster	Application platform
3C	CI/CD with OIDC	No secrets, full automation
🚨 Production Readiness Questions
Before going to prod, answer these:

Disaster Recovery: Do you have Terraform state backed up? (It's in Azure Storage with geo-redundancy ✅)

Secrets Rotation: How often does Key Vault rotate secrets? (Set rotation_policy in Key Vault)

Cost Controls: Are budgets set? (Add azurerm_consumption_budget_resource_group)

Compliance: Are you logging all actions? (Azure Policy + Diagnostic Settings ✅)
