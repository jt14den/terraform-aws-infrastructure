---
title: "Remote State with S3 and DynamoDB"
teaching: 15
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- Why store Terraform state remotely instead of locally?
- How do I configure an S3 bucket and DynamoDB table for remote state?

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain the purpose and benefits of remote state.
- Configure an S3 backend for state storage.
- Add a DynamoDB table to enable state locking.

:::::::::::::::::::::::::::::::::::::::::::::::

## Why Use Remote State?

By default, Terraform keeps its state in a local file named `terraform.tfstate`.  
This works for testing or single-user setups, but it breaks down when:

- multiple people are working on the same infrastructure
- you want versioned backups of your state
- your laptop is not a safe place for critical infrastructure metadata

A **remote backend** solves these problems by putting your state in AWS.

We use:

- **Amazon S3** — stores the state file
- **DynamoDB** — provides a locking mechanism to prevent concurrent writes

:::::::::::::::::::::::::::::::::::: instructor

If learners have not used S3 or DynamoDB before, provide a quick overview of
these AWS services, emphasizing that only minimal configuration is needed for
Terraform.

::::::::::::::::::::::::::::::::::::::::::::::::

## S3 Bucket Configuration

Earlier, we created a bucket manually:

`ucla-tim-terraform-state`


You can also declare the bucket in Terraform (optional for this lesson):

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = "ucla-tim-terraform-state"
}

## DynamoDB Table Configuration

Terraform locking requires a simple table with a primary key named LockID.

Example:

```bash
resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Backend Configuration

Add a `backend` block to your main Terraform configuration.

```bash
terraform {
  backend "s3" {
    bucket         = "ucla-tim-terraform-state"
    key            = "terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```
After adding the backend, reinitialize Terraform:

`terraform init` 

Terraform will detect the backend and move your local state into S3.

::::::::::::::::::::::::::::::::::::: challenge

Convert Your Project to Remote State

Using the bucket and DynamoDB table created above:

Add a backend block to your main terraform configuration.

Run terraform init.

Confirm that Terraform offers to migrate your state.

Verify that your S3 bucket now includes a .tfstate file.

::::::::::::::::::::::::::::::::::::: solution

Terraform prompts:

`Do you want to copy existing state to the new backend? (yes/no)`

Answer yes.

You can view the resulting file in the AWS S3 console.

:::::::::::::::::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

* Remote state centralizes infrastructure metadata in AWS.
* S3 stores the state, DynamoDB handles locking.
* terraform init migrates local state to the remote backend.
* Remote backends enable safe collaboration.

:::::::::::::::::::::::::::::::::::::::::::::::
