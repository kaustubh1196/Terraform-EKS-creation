
# Creating EKS cluster using Terraform

A brief description of what this project does and who it's for

# Prerequisite
AWS Account: You should have an active AWS account with the necessary permissions to create and manage resources.

Terraform: Terraform is an infrastructure provisioning tool that you’ll need to install on your local machine.

Below are the steps for the Provision of the Amazon EKS Cluster using Terraform:                      
## 1.Provider

The provider is responsible for authenticating and establishing a connection with your AWS account. Begin by configuring the AWS provider to authenticate with your AWS account and give it a name ```provider.tf.```

```bash
  
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.5.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = ">= 2.6.0"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.7.0"
    }
  }
  required_version = ">= 0.13"
}


provider "aws" {
  region = var.region
  #   access_key = var.aws_access_key
  #   secret_key = var.aws_secret_key
}
```
## 2. Network Infrastructure Setup:
 Define the Virtual Private Cloud (VPC) resources required for the EKS cluster. VPC with the specified CIDR block and a subnet within that VPC will be created. Let’s name it ```vpc.tf.```
Module- terraform-aws-module/vpc

```bash
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = var.vpc_name
  cidr = "10.0.0.0/16"

  azs             = var.azs
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  enable_nat_gateway     = var.enable_nat_gateway
  enable_vpn_gateway     = var.enable_vpn_gateway
  one_nat_gateway_per_az = var.one_nat_gateway_per_az

  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = var.vpc_tags
  public_subnet_tags = {
    Name                                        = "Public Subnets"
    "kubernetes.io/role/elb"                    = 1
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
  private_subnet_tags = {
    Name                                        = "private-subnets"
    "kubernetes.io/role/internal-elb"           = 1
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
}
```
Deploying Kubernetes workers in the private subnets with a default route to NAT Gateway.
To expose the application to the Internet, you would need public subnets with a default route to the Internet Gateway.

## 3. EKS Cluster Setup:
EKS module in charge of Creating the control plane, and instance groups ( EKS-managed nodes, self-managed groups, Fargate profile). Let’s name it ```eks.tf.```

``` bash
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.27"

  cluster_endpoint_private_access = var.cluster_endpoint_private_access
  cluster_endpoint_public_access  = var.cluster_endpoint_public_access

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  enable_irsa = var.enable_irsa


  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }

  eks_managed_node_group_defaults = {
    disk_size = 30
  }

  eks_managed_node_groups = {
    general = {
      desired_size = 1
      min_size     = 1
      max_size     = 10

      labels = {
        role = "general"
      }

      instance_types = ["t3.micro"]
      capacity_type  = "ON_DEMAND"
    }

    spot = {
      desired_size = 1
      min_size     = 1
      max_size     = 10 #change as required

      labels = {
        role = "dev"
      }

      #   taints = [{
      #     key    = "dedicated"
      #     value  = "spot"
      #     effect = "NO_SCHEDULE"
      #   }]
      taints = {
        dedicated = {
          key    = "dedicated"
          value  = "spot"
          effect = "NO_SCHEDULE"
        }
      }
      instance_types = ["t3.micro"]
      capacity_type  = "SPOT"
    }
  }
  # manage_aws_auth_configmap = true
  # aws_auth_roles = [
  #   {
  #     rolearn  = module.eks_terraform_admin_iam_assumable_role.iam_role_arn
  #     username = module.eks_terraform_admin_iam_assumable_role.iam_role_name
  #     groups   = ["system:masters"]
  #   },
  # ]

  tags = var.eks_tags
}




# data "aws_eks_cluster" "cluster" {
#   name = module.eks.cluster_name
# }

# data "aws_eks_cluster_auth" "cluster" {
#   name = module.eks.cluster_name
# }

# provider "kubernetes" {
#   host                   = data.aws_eks_cluster.cluster.endpoint
#   cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
#   token                  = data.aws_eks_cluster_auth.cluster.token
#   exec {
#     api_version = "client.authentication.k8s.io/v1beta1"
#     args        = ["eks", "get-token", "--cluster-name", data.aws_eks_cluster.cluster.id]
#     command     = "aws"
#   }
# }
```
## 4. Terraform Variables:
Define Terraform variables for AWS resources. Name it ``` variables.tf.```

```bash 
#### provider Variables defined #######
variable "region" {
  type        = string
  description = "Name of the region to select"
  default     = "eu-west-2"
}

##### VPC Variables defined #######

variable "vpc_name" {
  type        = string
  description = "Name to be used on all the resources as identifier"
}
variable "public_subnets" {
  type        = list(string)
  description = "A list of public subnets inside the VPC"
  default     = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}
variable "private_subnets" {
  type        = list(string)
  description = "A list of private subnets inside the VPC"
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "azs" {
  type        = list(string)
  description = "A list of availability zones specified as argument to this module"
  default     = ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
}
variable "enable_nat_gateway" {
  type        = bool
  description = "Should be true if you want to provision NAT Gateways for each of your private networks"
  default     = "false"
}
variable "enable_vpn_gateway" {
  type        = bool
  description = "Should be true if you want to create a new VPN Gateway resource and attach it to the VPC"
  default     = "false"
}

variable "one_nat_gateway_per_az" {
  type        = bool
  description = "Should be true if you want only one NAT Gateway per availability zone"
  default     = "false"
}
variable "enable_dns_hostnames" {
  type        = bool
  description = "Should be true to enable DNS hostnames in the VPC"
  default     = "true"
}
variable "enable_dns_support" {
  type        = bool
  description = "Should be true to enable DNS support in the VPC"
  default     = "true"
}
variable "vpc_tags" {
  type = map(string)
  default = {
    Terraform   = "true"
    Environment = "dev"
  }
}


##### EkS Cluster Variables defined #######
variable "cluster_name" {
  type        = string
  description = "Name of the EKS cluster"
}

variable "cluster_endpoint_private_access" {
  type        = bool
  description = "Indicates whether or not the Amazon EKS private API server endpoint is enabled"
  default     = "true"
}
variable "cluster_endpoint_public_access" {
  type        = bool
  description = "Indicates whether or not the Amazon EKS public API server endpoint is enabled"
  default     = "false"
}
variable "enable_irsa" {
  type        = bool
  description = "Determines whether to create an OpenID Connect Provider for EKS to enable IRSA"
  default     = "true"
}
variable "eks_tags" {
  type = map(string)
  default = {
    Environment = "dev"
  }
}
```
## 5. terraform.tfvars:
 To define input variables and their values for vpc and cluster. Name as ```terraform.tfvars.```

```bash
#### VPC ###
vpc_name = "eks-terraform-vpc"
enable_nat_gateway = true
enable_vpn_gateway = true


### Cluster ##
cluster_name = "eks-terraform-project"
cluster_endpoint_private_access = true
cluster_endpoint_public_access  = true
```
## 6. Output.tf: 
To define outputs that can be retrieved after applying your Terraform configuration. Name as output.tf.
```bash
################################################################################
# VPC
################################################################################

output "vpc_id" {
  value       = module.vpc.vpc_id
  description = "VPC ID"
}
output "public_subnets" {
  value       = module.vpc.public_subnets
  description = "VPC public subnets' IDs list"
}
output "private_subnets" {
  value       = module.vpc.private_subnets
  description = "VPC private subnets' IDs list"
}

################################################################################
# Cluster
################################################################################
output "cluster_arn" {
  description = "The Amazon Resource Name (ARN) of the cluster"
  value       = module.eks.cluster_arn
}
output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data required to communicate with the cluster"
  value       = module.eks.cluster_certificate_authority_data
}

output "cluster_endpoint" {
  description = "Endpoint for your Kubernetes API server"
  value       = module.eks.cluster_endpoint
}

output "cluster_name" {
  description = "The name of the EKS cluster"
  value       = module.eks.cluster_name
}

output "cluster_oidc_issuer_url" {
  description = "The URL on the EKS cluster for the OpenID Connect identity provider"
  value       = module.eks.cluster_oidc_issuer_url
}

output "cluster_platform_version" {
  description = "Platform version for the cluster"
  value       = module.eks.cluster_platform_version
}

output "cluster_status" {
  description = "Status of the EKS cluster. One of `CREATING`, `ACTIVE`, `DELETING`, `FAILED`"
  value       = module.eks.cluster_status
}

output "cluster_security_group_id" {
  description = "Cluster security group that was created by Amazon EKS for the cluster. Managed node groups use this security group for control-plane-to-data-plane communication. Referred to as 'Cluster security group' in the EKS console"
  value       = module.eks.cluster_security_group_id
}

################################################################################
# IRSA
################################################################################

output "oidc_provider" {
  description = "The OpenID Connect identity provider (issuer URL without leading `https://`)"
  value       = module.eks.oidc_provider
}

output "oidc_provider_arn" {
  description = "The ARN of the OIDC Provider if `enable_irsa = true`"
  value       = module.eks.oidc_provider_arn
}

output "cluster_tls_certificate_sha1_fingerprint" {
  description = "The SHA1 fingerprint of the public key of the cluster's certificate"
  value       = module.eks.cluster_tls_certificate_sha1_fingerprint
}
```
let’s go to the terminal and run Terraform. Initialize first and then apply.
```
$ terraform init
$ terraform apply -auto-approve
```
Before connecting to the cluster, update the Kubernetes context.
```
$ aws eks update-kubeconfig --name eks-terraform-project --region eu-west-2
Updated context arn:aws:eks:eu-west-2:<ACCOUNT_NUMBER>:cluster/eks-terraform-project in /Users/.kube/config
```






