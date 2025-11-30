---
title: "Using Variables in Terraform"
teaching: 15
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- How can we make Terraform configurations flexible instead of hard-coded?
- How do we define and use variables in Terraform?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Define input variables in a `variables.tf` file.
- Use variables inside Terraform resources.
- Override variables from the command line or with `terraform.tfvars`.

::::::::::::::::::::::::::::::::::::::::::::::::

## Why Variables Matter

In the previous episodes, we hard-coded details like the AMI ID, instance type, and allowed SSH CIDR. This is fine for experimentation, but not practical when:

- You deploy to different environments.
- You need to give learners flexibility.
- You plan to reuse your Terraform modules.

Terraform variables make your configuration more reusable and easier to maintain.

---

## Creating a `variables.tf` File

Create a new file:

```bash
touch variables.tf
```

Add the following:

variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "us-west-2"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "instance_name" {
  description = "Name tag for the EC2 instance"
  type        = string
  default     = "terraform-demo"
}

::::::::::::::::::::::::::::::::::::: callout

By convention, variables are collected in their own file. Terraform loads all .tf files automatically, so file names don’t affect execution order.

::::::::::::::::::::::::::::::::::::::::::::::::

Using Variables in main.tf

Update your EC2 resource to reference variables:

```bash
provider "aws" {
  region = var.aws_region
}

resource "aws_instance" "demo" {
  ami           = "ami-0adebe3adcec1fa34"
  instance_type = var.instance_type

  tags = {
    Name = var.instance_name
  }
}
```

## Overriding Variables

1. Override at apply time
terraform apply -var="instance_type=t3.small"
2. Use a .tfvars file

Create a `terraform.tfvars` file:

```bash
instance_type = "t3.large"
instance_name = "demo-from-tfvars"
```

Now apply normally:

```bash
terraform apply
```

Terraform automatically loads terraform.tfvars.

::::::::::::::::::::::::::::::::::::: challenge

Challenge: Create Your Own Variable

Add a new variable called ssh_cidr that lets the learner specify which IP is allowed to SSH into the instance.

Then update your Security Group to use it.

:::::::::::::::::::::::: solution

Add to variables.tf:

```bash
variable "ssh_cidr" {
  description = "CIDR allowed for SSH"
  type        = string
  default     = "0.0.0.0/0"
}
```

Update the security group:

```bash
ingress {
  description = "SSH"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = [var.ssh_cidr]
}
```

::::::::::::::::::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::::: keypoints

* Variables make Terraform code reusable and easier to maintain.
* Variables are defined once and used many times across configurations.
* You can override variables with -var or terraform.tfvars.

::::::::::::::::::::::::::::::::::::::::::::::::
