---
title: "Your First Terraform Configuration"
teaching: 15
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I create a Terraform project?
- What is a `main.tf` file and what belongs in it?
- How does Terraform know what cloud resources to manage?

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Create a new Terraform working directory.
- Write a minimal `main.tf` file.
- Initialize Terraform and interpret the output.
- Explain how providers connect Terraform to AWS.

:::::::::::::::::::::::::::::::::::::::::::::::

## Creating the Terraform Project Directory

Terraform works inside a folder that contains configuration files, usually
ending in `.tf`. Start by creating a clean directory:

~~~bash
mkdir terraform-dataverse
cd terraform-dataverse
~~~

Inside that folder, we create our first configuration file.

## Writing a Minimal `main.tf`

A minimal Terraform configuration needs two blocks:

1. A required provider declaration  
2. A provider configuration (here, AWS)

~~~hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}
~~~

### What this does

- The `terraform` block tells Terraform which provider plugin to use.
- The `provider "aws"` block sets your AWS region.
- Terraform does *not* create any infrastructure yet. It only knows how to talk to AWS.

## Initialize Terraform

Before Terraform can do anything, it needs to download provider plugins:

~~~bash
terraform init
~~~

You should see output confirming that the AWS provider was installed.

## Running Your First Plan

Even though we haven't defined resources yet, it’s safe to run:

~~~bash
terraform plan
~~~

Expected output:

No changes. Your infrastructure matches the configuration.


This means:

- Terraform is working correctly
- Terraform can talk to AWS
- Terraform sees no resources yet (because we haven’t defined any)

::::::::::::::::::::::::::::::::::::: callout

**Why is this useful?**

This episode establishes the foundation.  
If `init` and `plan` work now, troubleshooting future problems becomes much easier.  
Most Terraform issues start with authentication or provider errors.

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform configurations live in directories containing `.tf` files.  
- The `terraform` and `provider` blocks define how Terraform interacts with AWS.  
- `terraform init` downloads provider plugins.  
- `terraform plan` previews changes, even when the configuration is empty.  
- A clean plan confirms your AWS authentication is working.

:::::::::::::::::::::::::::::::::::::::::::::::
