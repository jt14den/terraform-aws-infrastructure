---
title: "Creating Your First AWS EC2 Instance"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I define a cloud resource in Terraform?
- What is an `aws_instance` block?
- How does Terraform preview and apply changes?
- How can I safely destroy cloud resources?

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Add an EC2 instance resource to `main.tf`.
- Understand key arguments of the `aws_instance` block.
- Use `terraform plan` to preview changes.
- Use `terraform apply` to create infrastructure.
- Destroy resources safely with `terraform destroy`.

:::::::::::::::::::::::::::::::::::::::::::::::

## Adding an EC2 Instance to `main.tf`

We extend our configuration from Episode 2 by adding the `aws_instance` resource.

Add this block to your `main.tf`:

~~~hcl
resource "aws_instance" "example" {
  ami           = "ami-0adebe3adcec1fa34"   # Rocky Linux 9 AMI
  instance_type = "t3.micro"

  tags = {
    Name = "terraform-example"
  }
}
~~~

## Understanding the Key Arguments

- `resource "aws_instance" "example"`  
  Creates an EC2 instance and names it `example` in the Terraform state.

- `ami`  
  Identifies the Amazon Machine Image. Different AMIs represent different OSes.

- `instance_type`  
  Sets CPU & memory size.

- `tags`  
  Adds human-readable metadata in AWS.

## Previewing the Change

Run:

~~~bash
terraform plan
~~~

Terraform will show:

- one resource to add  
- full details of what will be created  
- no changes to existing resources

This preview step is important—nothing changes until we apply.

## Creating the EC2 Instance

Apply the plan:

~~~bash
terraform apply
~~~

You will be prompted:

Do you want to perform these actions?
Only 'yes' will be accepted.

Type:

yes


Within about 20–60 seconds the EC2 instance will launch.

Terraform will output:

- the instance ID  
- the public DNS  
- the public IP  

These values are now part of Terraform’s **state**, so it knows the instance exists.

## Verifying the Instance in AWS

Visit your AWS Console:

- EC2 → Instances  
- Confirm the instance named `terraform-example` is **running**

## Destroying the Instance (Optional)

Terraform can remove everything it created:

~~~bash
terraform destroy
~~~

This is useful while practicing to avoid unnecessary AWS charges.

::::::::::::::::::::::::::::::::::::: challenge

## Practice: Change the Instance Type

Modify this line:

~~~hcl
instance_type = "t3.micro"
~~~

Change it to:

~~~hcl
instance_type = "t3.small"
~~~

Run:

~~~bash
terraform plan
~~~

What does Terraform tell you it will do?

:::::::::::::::::::::::: solution

Terraform should report an **in-place update**, changing only the instance type:

~ instance_type: "t3.micro" -> "t3.small"

No resources are destroyed unless the change requires replacement.

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform uses resource blocks to define cloud infrastructure.
- The AWS provider includes many resource types; EC2 is one of them.
- `terraform plan` previews changes before they occur.
- `terraform apply` creates or updates cloud resources.
- Terraform maintains state so it always knows what exists.
- `terraform destroy` safely removes all managed resources.

::::::::::::::::::::::::::::::::::::::::::::::::
