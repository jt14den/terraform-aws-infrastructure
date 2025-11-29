---
title: Setup
---

To take this lesson, you will need the following tools installed and configured.

## 1. AWS Account

You should have an AWS account you can log into.  
We will use AWS services such as EC2, IAM, and S3 during the lesson.

## 2. Install the AWS CLI

### macOS

Install the AWS CLI using Homebrew:

``` bash
brew install awscli
```

Verify the installation:

``` bash
aws --version

```

## 3. Configure AWS Credentials

Use AWS IAM Identity Center (formerly AWS SSO):

``` bash

aws configure sso
```

Follow the browser prompt, then confirm your identity:

``` bash
aws sts get-caller-identity
```

You should see your AWS account ID and ARN.

## 4. Install Terraform

Download Terraform from the official website or install with Homebrew:

    brew tap hashicorp/tap
    brew install hashicorp/tap/terraform

Verify the installation:

    terraform version

## 5. Create a Working Directory

Set up a directory for your Terraform files:

    mkdir terraform-dataverse
    cd terraform-dataverse

## 6. (Optional) Clone the Lesson Repository

If you want the example files used in this lesson:

    git clone https://github.com/jt14den/terraform-aws-infrastructure

You are now ready to begin the lesson.
