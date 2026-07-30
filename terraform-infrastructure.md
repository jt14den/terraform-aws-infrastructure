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
- Describe what an Elastic IP is and why the one defined here doesn't yet simplify the rebuild cycle.
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

This project uses **remote state** -- but each operator has their own bucket and key,
not a shared one:

```
Tim:   s3://ucla-tim-terraform-state/terraform-dataverse/tim/terraform.tfstate
Jamie: s3://ucla-library-terraform-state/terraform-dataverse/jamie/terraform.tfstate
```

Tim's `apply` and Jamie's `apply` cannot see or affect each other's state at all --
they are backed by different buckets. This is deliberate: it means destroying or
rebuilding one operator's environment can't touch the other's, which matters a lot
given how often `make rebuild` tears an environment down and recreates it (Episode 6).
Both backends still use a shared DynamoDB table (`terraform-locks`) for locking, which
prevents two `terraform apply` runs against the *same* state from racing each other --
but that only protects an operator against themselves (e.g. two terminal tabs), not
against each other.

`s3://ucla-dataverse-migration-assets/` is a different bucket entirely -- it holds
database dumps and migration assets, not Terraform state. Don't confuse the two.

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

In principle, an Elastic IP fixes this: reserve the address once, reassociate it with
whatever instance exists, and the address survives even when the instance doesn't.

**That's not what happens today, though.** The `aws_eip.dataverse` resource in
`modules/dataverse_ec2/main.tf` is defined in the *same module* as the EC2 instance,
associated directly to `aws_instance.dataverse.id`. When `make rebuild` runs
`terraform destroy`, it tears down the whole module -- instance and EIP together --
so the address does *not* survive a rebuild yet. Roadmap item `02-01` ("Elastic IP
resource... for both environments") is still unchecked for exactly this reason: having
an `aws_eip` resource isn't the same as having a *persistent* one. The real
`make rebuild` output today prints a new IP and pauses for you to update the DNS A
record by hand (`Makefile`, the `rebuild` target) -- which is the friction this feature
is meant to remove, and hasn't yet.

## Variables and tfvars

Terraform configurations use variables to avoid hardcoding values that differ between environments.

- **`variables.tf`**: declares the variables and their types (like a function signature)
- **`terraform.tfvars`**: provides values for those variables (like the function call)

The `terraform.tfvars` file for each environment is **gitignored, not committed** --
each operator creates their own from `terraform.tfvars.example` during setup (Episode 2).
That's intentional, because in practice it's not secret-free: Tim's real `tfvars` includes
a plaintext `db_password`. This is a known, flagged gap (the security audit calls it out
as finding F8) -- the intended design keeps secrets in Ansible Vault, not Terraform, but
`db_password` currently lives in both places, and the Terraform copy isn't encrypted.
Don't treat "it's gitignored" as equivalent to "it's safe" -- gitignore keeps a file out of
version control, it doesn't encrypt what's on disk.

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

::::::::::::::::::::::::::::::::::::: challenge

### State isolation and IP stability

Without checking the episode:

1. Can Tim accidentally destroy Jamie's environment by running `terraform destroy` in
   his own environment directory? Point to the specific piece of config that proves your answer.
2. Does today's public EC2 IP survive `make rebuild ENV=tim`? Why or why not?

:::::::::::::::::::::::::::::::::::: solution

1. No -- each environment's `main.tf` points at a different S3 bucket/key for its backend
   (`ucla-tim-terraform-state` vs. `ucla-library-terraform-state`, different keys). Terraform
   only knows about resources tracked in the state it's pointed at, so Tim's `destroy` has no
   way to reach anything in Jamie's state file.
2. No. `aws_eip.dataverse` lives in the same module as `aws_instance.dataverse` and is
   associated directly to it, so `terraform destroy` removes both together. The address
   changes on every rebuild until roadmap item `02-01` makes the EIP persistent.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform manages EC2, RDS, S3, security groups, an Elastic IP resource, and IAM for this project.
- State is stored remotely in S3, but Tim and Jamie each have their own bucket/key -- state is isolated per operator, not shared.
- Each operator has their own environment directory; both use shared modules.
- An Elastic IP resource exists, but it's tied to the instance's lifecycle, so it does **not** yet survive `make rebuild` -- that's still open work (roadmap `02-01`).
- `terraform.tfvars` is gitignored per environment, but is not currently secret-free in practice -- a known gap (audit F8).

::::::::::::::::::::::::::::::::::::::::::::::::
