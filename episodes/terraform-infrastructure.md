---
title: "Terraform: The Infrastructure Layer"
teaching: 25
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What AWS resources does Terraform manage for this project?
- How is Terraform state stored and shared between operators?
- How are Tim's and Jamie's environments organized?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Read a Terraform config and identify the AWS resource types it defines.
- Explain what remote state is and why it matters for a multi-operator project.
- Describe what an Elastic IP is and why it simplifies the rebuild cycle.
- Run `terraform plan` and interpret the output.

::::::::::::::::::::::::::::::::::::::::::::::::

## What Terraform manages

Terraform is responsible for the AWS resources that exist -- the "what" of the infrastructure.
For this project, those resources are:

- **EC2 instance**: the virtual machine where Dataverse and its supporting services run
- **RDS instance**: the managed PostgreSQL database
- **S3 buckets**: file storage for Dataverse, plus a separate bucket for migration assets and backups
- **Security groups**: firewall rules controlling what traffic reaches the EC2 instance
- **Elastic IP**: a static IP address attached to the EC2 instance (more on this below)
- **IAM roles and policies**: permissions for the EC2 instance to read and write S3

Terraform does not install software. It creates the resources. Once the EC2 instance exists,
Ansible takes over and configures it.

## Repository layout

The `terraform-dataverse` repo is organized by operator environment:

```
terraform-dataverse/
  environments/
    tim/        <- Tim's dev environment
    jamie/      <- Jamie's dev environment
  modules/      <- shared resource definitions used by both environments
```

Each environment directory has its own `main.tf`, `variables.tf`, and `terraform.tfvars`.
The environments are mostly identical -- they share modules -- but use different resource names
and sizes so Tim and Jamie can work independently without affecting each other.

## Remote state

By default, Terraform stores its state in a local file called `terraform.tfstate`.
This is a problem for a team: if two people run Terraform from different machines,
their state files diverge and Terraform loses track of what actually exists in AWS.

This project uses **remote state** -- the state file is stored in an S3 bucket:

```
s3://ucla-dataverse-migration-assets/terraform/state/
```

Both operators read from and write to the same state file.
Terraform uses DynamoDB for state locking, which prevents two people from running
`terraform apply` at the same time and corrupting the state.

::::::::::::::::::::::::::::::::::::: callout

### Never edit the state file directly

The state file is JSON and technically editable, but Terraform is the only thing
that should write to it. Editing it by hand can put Terraform in an inconsistent
state that is painful to recover from. If state gets out of sync, use
`terraform state` subcommands to inspect and repair it.

::::::::::::::::::::::::::::::::::::::::::::::::

## Elastic IP

An Elastic IP (EIP) is a static IP address you reserve in AWS and attach to an EC2 instance.

Without an Elastic IP, every time you run `make rebuild` -- which destroys and recreates
the EC2 instance -- the instance gets a new public IP address. That means:

- The Ansible inventory file needs to be updated before Ansible can run
- Any DNS records pointing at the old IP are wrong
- You spend time tracking down the current IP instead of doing work

With an Elastic IP attached, the public IP stays the same across rebuilds.
The instance changes; the address does not.

This was added in Phase 2 of the migration plan specifically to reduce friction in the
development cycle.

## Variables and tfvars

Terraform configurations use variables to avoid hardcoding values that differ between environments.

- **`variables.tf`**: declares the variables and their types (like a function signature)
- **`terraform.tfvars`**: provides values for those variables (like the function call)

The `terraform.tfvars` file for each environment is committed to the repo but contains
no secrets -- just resource sizes, region, names, and tags.
Secrets (database passwords, API keys) are kept out of Terraform and passed via Ansible Vault.

::::::::::::::::::::::::::::::::::::: challenge

### Read a Terraform plan

From `terraform-dataverse/environments/tim`, run `terraform plan`.

Answer these questions from the output:

1. How many resources does Terraform plan to create, change, or destroy?
2. Which resource type appears the most?
3. Find the security group resource. What ports does it open, and to what CIDR range?

:::::::::::::::::::::::::::::::::: solution

The exact output will depend on current state. Things to look for:

- `aws_instance` (the EC2 instance)
- `aws_security_group` and `aws_security_group_rule` (firewall rules)
- `aws_eip` and `aws_eip_association` (Elastic IP)
- `aws_db_instance` (RDS)
- Port 22 (SSH), 80 (HTTP), 443 (HTTPS), and 8080/4848 (Payara) in the security group rules

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform manages EC2, RDS, S3, security groups, Elastic IP, and IAM for this project.
- State is stored remotely in S3 so both operators share the same view of infrastructure.
- Each operator has their own environment directory; both use shared modules.
- Elastic IP keeps the public IP stable across rebuilds, which simplifies Ansible and DNS.
- Variables in `terraform.tfvars` configure environments; secrets stay out of Terraform entirely.

::::::::::::::::::::::::::::::::::::::::::::::::
