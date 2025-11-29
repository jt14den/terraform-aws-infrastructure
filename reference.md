---
title: 'Reference'
---

## Glossary

AMI (Amazon Machine Image)
:   A template describing the OS and software configuration for launching an EC2 instance.

Apply (Terraform)
:   Terraform command that creates or updates cloud resources to match your configuration.

AWS Region
:   A geographic area where AWS operates data centers (e.g., `us-west-2`).  
    Terraform resources must all specify a region.

Backend (Terraform)
:   Where Terraform stores its state file (local file, S3, etc.).  
    Remote backends enable collaboration and reduce risk.

CIDR Block
:   A notation defining IP ranges, such as `203.0.113.24/32`.  
    Used for networking and security groups.

Configuration File (`*.tf`)
:   A Terraform file written in HCL that describes desired infrastructure.

Destroy (Terraform)
:   Command that removes all Terraform-managed resources.  
    Usage: `terraform destroy`.

EC2 Instance
:   A virtual server hosted in AWS.

HCL (HashiCorp Configuration Language)
:   The language used by Terraform for configuration.

Ingress Rule (Security Group)
:   A firewall rule allowing incoming traffic to an EC2 instance.

Instance Type
:   A hardware profile specifying CPU, memory, and network performance (e.g., `t3.micro`, `t3.large`).

Key Pair (AWS)
:   Public/private SSH key used to authenticate when logging into EC2 instances.

Module (Terraform)
:   A folder containing Terraform code that groups logical resources for reuse.

Output (Terraform)
:   A value returned by Terraform after an apply, such as an instance IP or DNS name.

Plan (Terraform)
:   Command that previews changes before Terraform applies them.  
    Usage: `terraform plan`.

Provider (Terraform)
:   A plugin that allows Terraform to interact with a platform such as AWS, Azure, GitHub, or Kubernetes.

Resource (Terraform)
:   A specific piece of infrastructure described in Terraform (EC2 instance, S3 bucket, etc.).

Security Group
:   An AWS virtual firewall controlling inbound and outbound traffic.

SSH
:   Secure Shell, a network protocol used to log into remote machines.

State File (`terraform.tfstate`)
:   The file Terraform uses to track all managed resources.  
    Can be stored locally or in a remote backend.
