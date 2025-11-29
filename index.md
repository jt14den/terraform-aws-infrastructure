---
site: sandpaper::sandpaper_site
---

This lesson introduces Terraform as a tool for creating and managing AWS infrastructure.  
Learners start with simple EC2 deployments and build toward modular, repeatable patterns that support real services such as Dataverse.

The lesson uses a step-by-step, hands-on approach. Each task builds on the previous one, keeping cognitive load low and giving time to practice core ideas. Learners work directly in their AWS accounts and write Terraform configurations from scratch.

:::::::::::::::: prerequisites

Learners should:

- know basic command-line navigation  
- have an AWS account they can use for testing  
- be comfortable editing text files  
- have no prior Terraform experience

:::::::::::::::::::::::

## Learning Objectives

By the end of this lesson, learners will be able to:

- describe what Infrastructure-as-Code means  
- use Terraform to create, update, and destroy AWS resources  
- configure providers, variables, and outputs  
- launch an EC2 instance with a security group and key pair  
- understand how Terraform state works  
- refactor a flat configuration into modules  
- prepare an AWS environment for Ansible-based deployments  
