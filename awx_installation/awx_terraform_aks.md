# Enterprise Ansible AWX Deployment on Azure Kubernetes Service (AKS) via Terraform

This document details how to provision an **Azure Kubernetes Service (AKS)** cluster and automatically deploy production-grade **Ansible AWX** onto it using **Terraform** and the **Helm / Kubernetes Terraform Providers**.

---

## 1. High-Level Process Flow

```
+-----------------------------------------------------------------------------------+
|                                 HIGH-LEVEL FLOW                                   |
+-----------------------------------------------------------------------------------+
|  [Phase 1: Infrastructure Provisioning (Terraform AzureRM Provider)]              |
|   ├── Provision Azure Resource Group.                                             |
|   ├── Provision Virtual Network (VNet) & Dedicated Subnet.                        |
|   └── Provision Managed Azure Kubernetes Service (AKS) Cluster.                   |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Phase 2: Kubernetes & Helm Provider Initialization]                            |
|   └── Dynamically pull AKS client credentials (host, certs, token) from Phase 1.  |
|   └── Configure Helm Provider to target the newly provisioned AKS cluster.        |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Phase 3: AWX Operator Deployment via Helm]                                      |
|   └── Deploy AWX Operator Helm chart into namespace `awx`.                        |
|   └── Override `kube-rbac-proxy` image repository to Quay mirror.                 |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Phase 4: AWX Instance Provisioning (Kubernetes Manifest Provider)]             |
|   └── Apply `awx.ansible.com/v1beta1` custom resource manifest.                  |
|   └── Configure LoadBalancer / Ingress, PostgreSQL storage class, & replica specs.|
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Phase 5: Output & Post-Deployment]                                             |
|   └── Output Public LoadBalancer IP / FQDN and retrieve admin password.           |
+-----------------------------------------------------------------------------------+
```

---

## 2. Step-by-Step Implementation & Infrastructure Code

### Step 1: Define Terraform Code Architecture

Create a dedicated workspace directory for Terraform manifests:

```bash
mkdir -p terraform-awx-aks && cd terraform-awx-aks
```

Create the following files:
1. `main.tf` (Providers and Infrastructure definitions)
2. `variables.tf` (Configurable variables)
3. `outputs.tf` (Deployment outputs)

#### File: `variables.tf`
```hcl
variable "prefix" {
  type        = string
  default     = "awx-aks"
  description = "Prefix for all Azure resources"
}

variable "location" {
  type        = string
  default     = "East US"
  description = "Azure Region"
}

variable "node_count" {
  type        = number
  default     = 3
  description = "Number of AKS worker nodes"
}

variable "node_vm_size" {
  type        = string
  default     = "Standard_D4s_v5" # 4 vCPU, 16GB RAM for production AWX workloads
  description = "VM spec for worker nodes"
}
```

#### File: `main.tf`
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.90.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.26.0"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = "~> 1.14.0"
    }
  }
}

# ------------------------------------------------------------------------------
# Azure Provider Configuration
# ------------------------------------------------------------------------------
provider "azurerm" {
  features {}
}

# 1. Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "${var.prefix}-rg"
  location = var.location
}

# 2. Virtual Network & Subnet
resource "azurerm_virtual_network" "vnet" {
  name                = "${var.prefix}-vnet"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.240.0.0/16"]
}

resource "azurerm_subnet" "aks_subnet" {
  name                 = "${var.prefix}-aks-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.240.0.0/20"]
}

# 3. AKS Cluster Provisioning
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "${var.prefix}-cluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "${var.prefix}-k8s"

  default_node_pool {
    name           = "systempool"
    node_count     = var.node_count
    vm_size        = var.node_vm_size
    vnet_subnet_id = azurerm_subnet.aks_subnet.id
    os_disk_size_gb = 50
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }
}

# ------------------------------------------------------------------------------
# Dynamic Provider Credentials Setup (Connecting Helm & Kubernetes to AKS)
# ------------------------------------------------------------------------------
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
  }
}

provider "kubectl" {
  host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
  load_config_file       = false
}

# 4. Create Namespace for AWX
resource "kubernetes_namespace" "awx" {
  depends_on = [azurerm_kubernetes_cluster.aks]
  metadata {
    name = "awx"
  }
}

# 5. Deploy AWX Operator via Helm Chart
resource "helm_release" "awx_operator" {
  name             = "awx-operator"
  repository       = "https://ansible-community.github.io/awx-operator-helm/"
  chart            = "awx-operator"
  version          = "2.19.1"
  namespace        = kubernetes_namespace.awx.metadata[0].name
  create_namespace = false

  set {
    name  = "rbac_proxy.image.repository"
    value = "quay.io/brancz/kube-rbac-proxy"
  }

  set {
    name  = "rbac_proxy.image.tag"
    value = "v0.15.0"
  }
}

# 6. Deploy AWX Custom Resource Instance via kubectl manifest provider
resource "kubectl_manifest" "awx_instance" {
  depends_on = [helm_release.awx_operator]

  yaml_body = <<YAML
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-cloud
  namespace: awx
spec:
  service_type: LoadBalancer
  postgres_configuration_secret: awx-cloud-postgres-configuration
  projects_persistence: true
  projects_storage_access_mode: ReadWriteOnce
YAML
}
```

#### File: `outputs.tf`
```hcl
output "aks_cluster_name" {
  value       = azurerm_kubernetes_cluster.aks.name
  description = "The name of the provisioned AKS cluster"
}

output "get_credentials_command" {
  value       = "az aks get-credentials --resource-group ${azurerm_resource_group.rg.name} --name ${azurerm_kubernetes_cluster.aks.name}"
  description = "Azure CLI command to connect local kubectl to AKS"
}

output "get_admin_password_command" {
  value       = "kubectl get secret awx-cloud-admin-password -n awx -o jsonpath='{.data.password}' | base64 --decode; echo"
  description = "Command to retrieve the AWX superuser password"
}
```

---

### Step 2: Commands to Execute Terraform Deployment

Follow these sequential shell execution steps:

```bash
# Initialize Terraform providers, modules, and state backend
terraform init
```
* **Why run this?** Downloads required AzureRM, Helm, Kubernetes, and Kubectl providers to local cache.

```bash
# Validate code syntax and structural integrity
terraform validate
```
* **Why run this?** Catches configuration syntax errors prior to executing cloud API calls.

```bash
# Generate and inspect execution plan
terraform plan -out=tfplan
```
* **Why run this?** Shows exactly what cloud resources (VNet, Subnet, AKS, Helm releases) will be created without making changes yet.

```bash
# Apply infrastructure plan to Azure Cloud
terraform apply tfplan
```
* **Why run this?** Provisions infrastructure in Azure, provisions AKS, bootstraps AWX Operator via Helm, and instantiates the AWX cloud deployment.

---

### Step 3: Post-Deployment Verification & Access

Once `terraform apply` finishes:

```bash
# 1. Fetch AKS credentials into your local kubeconfig
az aks get-credentials --resource-group awx-aks-rg --name awx-aks-cluster

# 2. Check the status of the AWX pods in AKS
kubectl get pods -n awx -w

# 3. Retrieve the public LoadBalancer IP assigned to AWX by Azure
kubectl get svc awx-cloud-service -n awx

# 4. Retrieve the decoded admin password
kubectl get secret awx-cloud-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode; echo
```

---

## 3. Detailed Summary

* **Production Readiness:** Unlike local Kind environments, hosting AWX on AKS leveraging Terraform provides enterprise-grade scalability:
  - **Managed Kubernetes:** Azure manages control plane uptime, API availability, and master node patching.
  - **Automated Public Exposure:** Setting `service_type: LoadBalancer` automatically provisions a public Azure Standard Load Balancer with an external IP address.
  - **Storage Management:** AKS dynamically provisions Managed Disks for persistent storage (`postgres` data and project persistent volumes).
* **IaC Benefits:** Using Terraform ensures zero manual steps. Cluster creation, operator deployment, registry mirror fixes, and CR creation are completely declarative and version-controlled.
* **Cost & Sizing Considerations:** AWX PostgreSQL migrations and web tasks require substantial CPU/RAM. Pinned compute specifications using `Standard_D4s_v5` nodes ensure stable database execution and prevent pod termination under heavy job run loads.

---
