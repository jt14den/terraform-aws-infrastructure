
---
title: "Setup"
---

:::::::::::::::::::::::::::::::::::::: questions

- What tools do I need installed before starting this lesson?
- How do I configure AWS, Terraform, and my local environment?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Install AWS CLI on macOS, Windows, and Linux
- Configure AWS credentials using AWS IAM Identity Center (SSO)
- Install Terraform
- Verify all tools are ready for the lesson

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

This page walks you through installing and configuring everything required to follow the lesson.  
By the end of this setup, you will be able to authenticate to AWS, run Terraform, and prepare a working directory.


## 1. AWS Account

You will need an AWS account you can log into.  
During the lesson you will use:

- EC2 (virtual machines)  
- IAM (identity and access)  
- S3 (storage)  


## 2. Install the AWS CLI

We use the AWS CLI throughout the lesson to authenticate and confirm credentials.

:::::::::::::::::::::::::::::::::::::: group-tab

### macOS

Install using Homebrew:

```bash
brew install awscli
````

Verify:

```bash
aws --version
```

### Windows

Use the official MSI installer:

1. Download:
   [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Run installer
3. Verify:

```powershell
aws --version
```

### Linux

On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y awscli
```

On Fedora/RHEL/Amazon Linux:

```bash
sudo dnf install -y awscli
```

Verify:

```bash
aws --version
```

:::::::::::::::::::::::::::::::::::::::::::::::


## 3. Configure AWS Credentials (IAM Identity Center / SSO)

This lesson uses **IAM Identity Center**, the modern AWS authentication system.

Run:

```bash
aws configure sso
```

Follow the browser prompts to authenticate.

Then confirm your identity:

```bash
aws sts get-caller-identity
```

You should see:

* Your AWS account ID
* Your IAM role ARN
* Your user ID


## 4. Install Terraform

:::::::::::::::::::::::::::::::::::::: group-tab

### macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### Windows

Using Chocolatey:

```powershell
choco install terraform
```

Using Scoop:

```powershell
scoop install terraform
```

### Linux

On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
| sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install terraform
```

On Fedora/RHEL:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install terraform
```
:::::::::::::::::::::::::::::::::::::::::::::::

Verify installation:

```bash
terraform version
```

## 5. Create a Working Directory

Create a folder where your Terraform configuration files will live:

```bash
mkdir terraform-dataverse
cd terraform-dataverse
```

## 6. (Optional) Clone the Lesson Repository

If you want the example files from this lesson:

```bash
git clone https://github.com/jt14den/terraform-aws-infrastructure
```

::::::::::::::::::::::::::::::::::::: keypoints

* Install AWS CLI, Terraform, and Git before beginning the lesson
* Use `aws configure sso` to authenticate with AWS Identity Center
* Use `terraform version` and `aws --version` to verify installation
* Create a dedicated working directory for Terraform files

::::::::::::::::::::::::::::::::::::::::::::::::
