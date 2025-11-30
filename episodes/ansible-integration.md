---
title: "Integrating Terraform with Ansible"
teaching: 15
exercises: 8
---

:::::::::::::::::::::::::::::::::::::: questions
- How do Terraform and Ansible work together?
- What information must Terraform output for Ansible to use?
- How can Ansible provision software on a server Terraform created?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain the division of responsibilities between Terraform and Ansible.
- Generate Terraform outputs for Ansible inventory.
- Run Ansible against a Terraform-created EC2 instance.
- Describe when to run Terraform again versus when to run Ansible.

::::::::::::::::::::::::::::::::::::::::::::::::

## Why Terraform + Ansible?

Terraform builds infrastructure.  
Ansible configures what is *on* that infrastructure.

Think of it as:

- Terraform creates the EC2 instance, networking, storage, and access.
- Ansible installs Dataverse, Payara, PostgreSQL settings, and more.

## What Terraform Must Output

Ansible needs:

- Public IP or DNS  
- SSH username  
- SSH key path  

Example outputs:

    ```hcl
    output "instance_public_ip" {
      value = aws_instance.dataverse.public_ip
    }

    output "instance_public_dns" {
      value = aws_instance.dataverse.public_dns
    }
    ```

Show them with:

```bash
terraform output
```

## Creating a Simple Ansible Inventory

```bash
[dataverse]
ec2-1-2-3-4.us-west-2.compute.amazonaws.com

[dataverse:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/dataverse-ansible.ed25519
```

:::::::::::::::::::::::::::::::::::: instructor
Have learners SSH manually into the instance before running Ansible.
::::::::::::::::::::::::::::::::::::::::::::::::

## Running an Ansible Playbook

`ansible-playbook -i inventory.ini dataverse.yml`

Ansible will:

1. Connect over SSH
2. Install packages
3. Configure services
4. Deploy Dataverse

Terraform is not needed again unless infrastructure changes.

## When to Use Terraform vs Ansible

|  Task |  Terraform |  Ansible |  
| ---- | ---- | ----  |
|  Create or destroy EC2 instances |  ✅ |  ❌ |  
|  Install Dataverse |  ❌ |  ✅ |  
|  Configure Payara |  ❌ |  ✅ |  
|  Change instance type or disk size |  ✅ |  ❌ |  
|  Restart services / roll out config |  ❌ |  ✅ | 

::::::::::::::::::::::::::::::::::::: challenge

## Exercise: Test Connectivity
```bash
ansible dataverse -i inventory.ini -m ping
```
:::::::::::::::::::::::: solution

```text
dataverse | SUCCESS => {
  "changed": false,
  "ping": "pong"
}
```

::::::::::::::::::::::::::::::::::::::::::::::::
:::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Terraform = infrastructure
- Ansible = configuration
- Terraform outputs feed into Ansible
- Use Terraform for servers and networking
- Use Ansible for app installation and updates

::::::::::::::::::::::::::::::::::::::::::::::::


