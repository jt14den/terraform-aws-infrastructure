---
title: "Adding Security Groups to Your EC2 Instance"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What is an AWS security group?
- How do I create a security group with Terraform?
- How do I allow SSH and web traffic to an EC2 instance?
- How do I attach a security group to an existing resource?

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain the purpose of AWS security groups.
- Create a security group using Terraform.
- Add rules that allow SSH, HTTP, and HTTPS.
- Attach the security group to an EC2 instance.
- Preview and apply changes safely.

:::::::::::::::::::::::::::::::::::::::::::::::

## What Is a Security Group?

A security group acts as a virtual firewall for your EC2 instance.  
It controls which network traffic is allowed **in** and **out**.

Common inbound rules:

- **22** — SSH
- **80** — HTTP
- **443** — HTTPS

Outbound rules are typically open by default.

## Creating a Security Group in Terraform

Add this block to your `main.tf`:

~~~hcl
resource "aws_security_group" "web_sg" {
  name        = "terraform-web-sg"
  description = "Allow SSH, HTTP, and HTTPS"

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
~~~

## Attaching the Security Group to Your EC2 Instance

Modify the EC2 resource:

~~~hcl
resource "aws_instance" "example" {
  ami           = "ami-0adebe3adcec1fa34"
  instance_type = "t3.micro"

  vpc_security_group_ids = [
    aws_security_group.web_sg.id
  ]

  tags = {
    Name = "terraform-example"
  }
}
~~~

This tells AWS:

> “Launch this EC2 instance with the `web_sg` firewall rules.”

## Previewing the Change

Run:

~~~bash
terraform plan
~~~

Terraform should show:

- creation of a new security group  
- updates to the instance to attach that group  

## Applying the Change

~~~bash
terraform apply
~~~

Answer:

yes 


Once complete:

- Look under **EC2 → Security Groups** for `terraform-web-sg`
- Confirm your EC2 instance now lists that security group

::::::::::::::::::::::::::::::::::::: callout

Security groups are **stateful**.  
If inbound traffic is allowed, the response automatically flows back out.  
You don’t need explicit outbound rules to return data.

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

## Challenge: Restrict SSH to Your IP Only

Replace this line:

~~~hcl
cidr_blocks = ["0.0.0.0/0"]
~~~

With your IP in CIDR notation (example):

~~~hcl
cidr_blocks = ["203.0.113.24/32"]
~~~

Run:

~~~bash
terraform plan
~~~

What change does Terraform show?

:::::::::::::::::::::::: solution

Terraform should detect and apply an **in-place update** to the SSH ingress rule:

~ cidr_blocks: ["0.0.0.0/0"] -> ["203.0.113.24/32"]


Only the security group rule changes.
The EC2 instance does not need to be recreated.

:::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Security groups act as AWS firewalls.
- You define inbound and outbound rules in resource blocks.
- EC2 instances attach one or more security groups.
- Terraform updates security group rules without recreating the instance.
- Restricting SSH to your IP improves security.

::::::::::::::::::::::::::::::::::::::::::::::::
