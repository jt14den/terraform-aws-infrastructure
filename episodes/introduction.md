---
title: "Introduction to Terraform for AWS"
teaching: 10
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- What problem does Terraform solve?
- How does Terraform create and update cloud resources?
- Why is infrastructure as code useful for reproducibility?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Describe the purpose of Terraform.
- Explain the core workflow (`init`, `plan`, `apply`).
- Identify the AWS resources managed in this lesson.
- Recognize how infrastructure as code improves reliability.

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

This lesson introduces Terraform through a practical, minimal example:
launching and managing an EC2 instance on AWS.

Terraform is a tool that lets you define cloud resources in configuration files.
Instead of clicking through the AWS console, you describe resources such as:

- EC2 instances  
- security groups  
- key pairs  
- storage volumes  

Terraform then compares your configuration to what exists in AWS and makes the
changes needed to match the desired state.

By working this way, your infrastructure becomes:

- repeatable  
- easier to review  
- easier to share  
- easier to rebuild when something breaks  

This lesson is written in Pandoc-flavoured Markdown and built with The Carpentries Workbench.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: instructor

Encourage learners to say out loud (or write down) what they currently do to
create infrastructure (for example, “I click around in the AWS console”).
Use their answers to motivate why infrastructure as code is valuable.

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: callout

### The Terraform Workflow

We will use three core commands throughout the lesson:

- `terraform init` — prepare the working directory and download providers  
- `terraform plan` — show what Terraform will change  
- `terraform apply` — apply the changes to real infrastructure  

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform lets you describe infrastructure in code instead of clicking in a UI.  
- The main workflow is `init`, `plan`, and `apply`.  
- Infrastructure as code is more reproducible, reviewable, and maintainable.  
- This lesson will build up a small AWS environment step by step.

::::::::::::::::::::::::::::::::::::::::::::::::
