---
title: "Variables, tfvars, and Modular Patterns"
teaching: 12
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- How do variables improve Terraform configuration?
- What is the role of terraform.tfvars?
- Why do modules make infrastructure clearer and easier to manage?

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Define Terraform variables and describe their purpose.
- Use terraform.tfvars to store environment-specific values.
- Understand how modules group related infrastructure.
- Interpret a Terraform plan that references module outputs.

:::::::::::::::::::::::::::::::::::::::::::::::

## Why Variables Matter

Terraform variables help you avoid hard-coding values.  
Instead of embedding values directly in a resource, you define them once in a separate file and reference them everywhere.

Example variable definition (do NOT include backticks when pasting into Workbench):

variable "instance_type" {
  description = "EC2 instance size"
  type        = string
  default     = "t3.large"
}

Referenced in a resource:

instance_type = var.instance_type

This keeps your configuration clean, reusable, and portable.

## Using terraform.tfvars

terraform.tfvars lets you override defaults without editing your Terraform source files.

Example contents:

instance_type = "t3.xlarge"
ssh_cidr      = "91.196.220.29/32"

Terraform automatically loads these values when you run:

terraform plan

:::::::::::::::::::::::::::::::::::: instructor

Invite learners to experiment by changing values in terraform.tfvars and re-running the plan to see the effect.

::::::::::::::::::::::::::::::::::::::::::::::::

## Why Use Modules?

As your infrastructure grows, a single main.tf becomes cluttered.  
Modules organize related resources into reusable folders.

Example structure:

modules/
  dataverse_ec2/
    main.tf
    variables.tf

Your root configuration stays clean:

module "dataverse_ec2" {
  source = "./modules/dataverse_ec2"
  instance_ami     = var.instance_ami
  instance_type    = var.instance_type
  key_pair_name    = var.key_pair_name
  ssh_cidr         = var.ssh_cidr
  instance_name    = var.instance_name
  root_volume_size = var.root_volume_size
}

Terraform passes all variables into the module at plan time.

## Putting It All Together

When terraform plan runs, Terraform:

1. Loads variables from variables.tf  
2. Loads overrides from terraform.tfvars  
3. Injects them into the module  
4. Builds a dependency graph  
5. Produces a plan showing expected changes  

This is the core of reproducible infrastructure.

::::::::::::::::::::::::::::::::::::: challenge

### Try It

1. Change your EC2 instance type in terraform.tfvars  
2. Run terraform plan  
3. Observe what Terraform proposes  
4. Revert your change when done  

::::::::::::::::::::::::::::::::::::: solution

Terraform should show an in-place update if the instance type supports resizing.  
If not, Terraform will propose a destroy-and-recreate operation.

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Variables avoid hard-coded values and keep Terraform flexible.
- terraform.tfvars stores environment-specific settings.
- Modules organize infrastructure into logical components.
- Terraform wires everything together at plan time.

:::::::::::::::::::::::::::::::::::::::::::::::
