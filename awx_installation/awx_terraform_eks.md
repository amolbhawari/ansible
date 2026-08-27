# Automated Ansible AWX Deployment on Amazon EKS using Terraform

This guide provides a comprehensive, production-grade approach to provisioning an **Amazon Elastic Kubernetes Service (Amazon EKS)** cluster and deploying **Ansible AWX** onto it using **Terraform**, **Helm**, and the **Kubernetes/Kubectl** providers.

---

## 1. High-Level Process Flow

+-----------------------------------------------------------------------------------+
|                                 HIGH-LEVEL FLOW                                   |
+-----------------------------------------------------------------------------------+
|  [Phase 1: Infrastructure Provisioning (Terraform AWS Provider)]                  |
|   ├── Provision VPC, Public/Private Subnets, & Internet/NAT Gateways.             |
|   ├── Provision Amazon EKS Control Plane (Cluster).                               |
|   └── Provision EKS Managed Node Groups & EBS CSI Driver Add-on.                  |
+-----------------------------------------------------------------------------------+
│
▼
+-----------------------------------------------------------------------------------+
|  [Phase 2: Kubernetes & Helm Provider Initialization]                            |
|   └── Dynamically pull EKS client credentials & auth token via AWS IAM.           |
|   └── Configure Helm, Kubernetes, and Kubectl Providers targeting EKS endpoint.   |
+-----------------------------------------------------------------------------------+
│
▼
+-----------------------------------------------------------------------------------+
|  [Phase 3: AWX Operator Deployment via Helm]                                      |
|   └── Deploy AWX Operator Helm chart into namespace awx.                        |
|   └── Configure Operator image repositories and RBAC roles.                       |
+-----------------------------------------------------------------------------------+
│
▼
+-----------------------------------------------------------------------------------+
|  [Phase 4: AWX Instance Provisioning (Kubernetes Manifest Provider)]             |
|   └── Apply [awx.ansible.com/v1beta1](https://awx.ansible.com/v1beta1) Custom Resource Definition (CRD).           |
|   └── Provision AWS Network LoadBalancer service & dynamic EBS PV storage.        |
+-----------------------------------------------------------------------------------+
│
▼
+-----------------------------------------------------------------------------------+
|  [Phase 5: Post-Deployment & Verification]                                        |
|   └── Fetch EKS Kubeconfig, retrieve AWX LoadBalancer URL & Admin Password.       |
+-----------------------------------------------------------------------------------+


---

## 2. Infrastructure Architecture & Terraform Configuration

### Step 1: Directory Setup & File Architecture

Create a dedicated workspace directory:

```bash
mkdir -p terraform-awx-eks && cd terraform-awx-eks
Organize your solution into three core files:

variables.tf — Parameterized inputs (AWS region, cluster name, node sizing)

main.tf — AWS infrastructure, EKS cluster, Helm release, and AWX CRD

outputs.tf — Post-deployment access details and operational commands

Step 2: Code Implementation
File 1: variables.tf
Terraform
variable "aws_region" {
  type        = string
  default     = "us-east-1"
  description = "Target AWS Region for deployment"
}

variable "cluster_name" {
  type        = string
  default     = "awx-eks-cluster"
  description = "Name of the Amazon EKS cluster"
}

variable "vpc_cidr" {
  type        = string
  default     = "10.0.0.0/16"
  description = "Base CIDR block for the dedicated AWS VPC"
}

variable "node_instance_types" {
  type        = list(string)
  default     = ["t3.xlarge"]
  description = "EC2 instance types for EKS worker nodes (Recommended: 4 vCPU, 16GB RAM)"
}

variable "awx_namespace" {
  type        = string
  default     = "awx"
  description = "Kubernetes namespace dedicated to AWX"
}
File 2: main.tf
Terraform
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
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
# AWS Provider Setup
# ------------------------------------------------------------------------------
provider "aws" {
  region = var.aws_region
}

# ------------------------------------------------------------------------------
# 1. AWS VPC Module Configuration
# ------------------------------------------------------------------------------
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  # Tag subnets for AWS Load Balancer Auto-Discovery
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}

# ------------------------------------------------------------------------------
# 2. Amazon EKS Cluster Module
# ------------------------------------------------------------------------------
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.29"

  cluster_endpoint_public_access = true

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    awx_worker_nodes = {
      min_size     = 2
      max_size     = 4
      desired_size = 3

      instance_types = var.node_instance_types
      capacity_type  = "ON_DEMAND"
    }
  }

  # Enable AWS EBS CSI Driver Addon for dynamic persistent storage provisioning
  cluster_addons = {
    aws-ebs-csi-driver = {
      most_recent = true
    }
  }
}

# ------------------------------------------------------------------------------
# Dynamic Authentication for Kubernetes & Helm Providers
# ------------------------------------------------------------------------------
data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_name
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    token                  = data.aws_eks_cluster_auth.cluster.token
  }
}

provider "kubectl" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.cluster.token
  load_config_file       = false
}

# ------------------------------------------------------------------------------
# 3. Dedicated Kubernetes Namespace for AWX
# ------------------------------------------------------------------------------
resource "kubernetes_namespace" "awx" {
  depends_on = [module.eks]
  metadata {
    name = var.awx_namespace
  }
}

# ------------------------------------------------------------------------------
# 4. Deploy AWX Operator via Helm
# ------------------------------------------------------------------------------
resource "helm_release" "awx_operator" {
  name             = "awx-operator"
  repository       = "[https://ansible-community.github.io/awx-operator-helm/](https://ansible-community.github.io/awx-operator-helm/)"
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

# ------------------------------------------------------------------------------
# 5. Instantiate AWX Deployment via Kubectl Manifest
# ------------------------------------------------------------------------------
resource "kubectl_manifest" "awx_instance" {
  depends_on = [helm_release.awx_operator]

  yaml_body = <<YAML "aws_region" "configure_kubectl_command" "eks_cluster_name" "get_admin_passw