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
- Recognize why infrastructure as code improves reliability.

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

This lesson introduces Terraform through a practical, minimal example:
launching and managing an EC2 instance on AWS.

Terraform is a tool that lets you define cloud resources in code.  
Instead of clicking through the AWS console, you write configuration files that describe resources such as:

- EC2 instances  
- security groups  
- key pairs  
- storage volumes  

This approach makes your infrastructure repeatable, testable, and version-controlled.

We use Pandoc-flavored Markdown for lesson content. Code in this lesson is shown using Workbench-compatible formatting.

Three sections appear in every Carpentries episode:

1. **questions** — displayed at the top to focus the learner  
2. **objectives** — skills the learner should gain  
3. **keypoints** — quick reinforcement at the end  

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: instructor

This introduction works best when learners already have AWS CLI authentication set up.  
If not, suggest they review the Setup page before proceeding.

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

## Challenge 1: Why Use Terraform?

List one advantage of defining infrastructure in code instead of using the AWS console.

:::::::::::::::::::::::: solution

Versioning and reproducibility — code lets you recreate the same infrastructure any time.

::::::::::::::::::::::::::::::::::::::::::::::::

## Challenge 2: Thinking About the Workflow

Terraform uses a three-step workflow.  
What does `terraform plan` do?

:::::::::::::::::::::::: solution

It shows what Terraform *would* change, without applying the changes.

::::::::::::::::::::::::::::::::::::::::::::::::

## Callout: The Terraform Workflow

::::::::::::::::::::::::::::::::::::: callout

The three most important commands in Terraform are:

- `terraform init` — prepare the working directory  
- `terraform plan` — preview changes  
- `terraform apply` — create or update resources  

::::::::::::::::::::::::::::::::::::::::::::::::

## Figures

Terraform can produce architecture diagrams, but in this lesson we use simple conceptual images.

![High-level AWS Infrastructure](https://raw.githubusercontent.com/carpentries/logo/master/Badge_Carpentries.svg){alt='Placeholder diagram showing infrastructure as code.'}

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform describes infrastructure in code.  
- `init`, `plan`, and `apply` form the core workflow.  
- Code-based infrastructure is reproducible and easy to update.  
- This lesson will build a small AWS environment step by

::::::::::::::::::::::::::::::::::::::::
