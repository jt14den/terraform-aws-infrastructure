---
title: "Introduction: Why We Built This Way"
teaching: 15
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- What is this lesson and who is it for?
- What infrastructure does Dataverse need to run?
- How are Terraform, Ansible, and the Dataverse application connected?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Describe the UCLA Dataverse infrastructure stack and what each component does.
- Explain the division of responsibility between Terraform and Ansible.
- Navigate the three repositories that make up this infrastructure.
- Understand what the 5.14 to 6.8 migration involves and why it shaped these decisions.

::::::::::::::::::::::::::::::::::::::::::::::::

## What this lesson is

This lesson traces the infrastructure that runs the UCLA Library Dataverse instance.
It was written during the migration from Dataverse 5.14 to 6.8 -- not as abstract documentation,
but as a way to make explicit what was built, why each decision was made, and how the pieces fit together.

The audience is people doing the work or being onboarded to it: DSC staff, DataSquad students,
and anyone who will operate or hand off this system. It assumes comfort with the command line
and some exposure to cloud services or configuration management, but not infrastructure expertise.

## The stack at a glance

Running Dataverse requires several components working together:

```
+------------------------------------------------------------+
|                           AWS                              |
|                                                            |
|  +------------------------------------------------------+  |
|  |                    EC2 instance                      |  |
|  |                                                      |  |
|  |  +----------------+    +--------------------------+  |  |
|  |  |  Apache httpd  |    |  Payara (app server)     |  |  |
|  |  |  reverse proxy +---->                          |  |  |
|  |  |  + SSL         |    |  Dataverse (WAR file)    |  |  |
|  |  +----------------+    +-------------+------------+  |  |
|  |                                      |               |  |
|  |                        +-------------v------------+  |  |
|  |                        |  Solr (search index)     |  |  |
|  |                        +--------------------------+  |  |
|  +------------------------------------------------------+  |
|                                                            |
|  +---------------------+   +----------------------------+ |
|  |  RDS (PostgreSQL)   |   |  S3 (file storage)         | |
|  +---------------------+   +----------------------------+ |
+------------------------------------------------------------+
```

**EC2** is a virtual machine running Ubuntu. Payara, Solr, and Apache all run here.

**RDS** is a managed PostgreSQL database hosted by AWS. Dataverse stores all its metadata here:
datasets, files, users, permissions, version histories. The database is the source of truth
for everything except the actual file content.

**S3** is object storage for the data files that users upload. Dataverse stores file metadata in RDS
and file content in S3. The two must stay in sync -- a file record in the database pointing to a
missing S3 object is a broken dataset.

**Payara** is a Jakarta EE application server. Dataverse runs as a WAR (Web Application Archive)
file deployed inside Payara, similar to how a web application runs inside Tomcat but targeting the
Jakarta EE ecosystem. Most Dataverse configuration is applied via Payara JVM options
or through the Dataverse API at first boot.

**Solr** is a search engine that powers Dataverse's dataset and file search. It maintains its own
index independently of the database. After any database restore, the Solr index must be explicitly
rebuilt -- it will not update itself. A running Dataverse with a stale or empty Solr index will
appear to have no datasets.

**Apache httpd** acts as a reverse proxy in front of Payara and handles SSL termination. Requests
from browsers hit Apache first, then are forwarded to Payara on a non-public port. Apache also
handles URL rewriting and HTTP to HTTPS redirects.

## The three repositories

This infrastructure is managed across three repositories:

| Repository | What it does |
|---|---|
| `terraform-dataverse` | Provisions AWS resources: EC2, RDS, S3, security groups, Elastic IP, IAM roles |
| `dataverse-ansible` | Configures the EC2 instance: installs and configures Payara, Solr, Apache, and Dataverse |
| `dataverse-infrastructure` | Makefile targets, tests, baseline scripts, and runbooks that tie the other two together |

These map onto two concerns:

- **Infrastructure** (Terraform): what AWS resources exist and how they are connected
- **Configuration** (Ansible): what software is installed on those resources and how it is configured

You run Terraform first to provision the resources, then Ansible to configure them. The
`dataverse-infrastructure` repo is where you do both -- its Makefile calls out to Terraform
and Ansible so you rarely interact with either tool directly.

## Why infrastructure as code

Before Terraform and Ansible, changes to the Dataverse environment meant clicking through the
AWS console or running commands directly on the server by hand. That works until something breaks:

- You cannot reproduce exactly what you did
- The next person does not know what state the server is in
- Rebuilding from scratch after a failure is slow and depends on memory

Infrastructure as code puts the desired state of the system in version-controlled files.
Terraform describes what AWS resources should exist. Ansible describes what should be installed
and how it should be configured. Running them again should produce the same result -- this property
is called **idempotency**, and it is one of the central ideas in Episode 4.

## The migration context

This lesson was developed during the migration of the UCLA Library Dataverse instance from
version 5.14 to 6.8. Several decisions in the codebase -- the Makefile targets, the baseline
scripts, the FAKE DOI provider configuration, the 7-phase migration plan -- exist because of
constraints the migration imposed.

Where those decisions appear in later episodes, we explain why they were made.
The migration itself is the subject of the final episode.

::::::::::::::::::::::::::::::::::::: challenge

### Take stock

Before moving on, open the three repositories in your browser:

- https://github.com/ucla-data-science-center/terraform-dataverse
- https://github.com/ucla-data-science-center/dataverse-ansible
- https://github.com/ucla-data-science-center/dataverse-infrastructure

In each one, find:

1. The main configuration file or entry point
2. At least one file that corresponds to something in the stack diagram above

:::::::::::::::::::::::::::::::::: solution

Some things to look for:

- `terraform-dataverse`: `environments/tim/main.tf` or `environments/jamie/main.tf` for the per-operator Terraform config
- `dataverse-ansible`: `tasks/main.yml` for the role entry point; `group_vars/` for environment-specific configuration
- `dataverse-infrastructure`: `Makefile` for the operations entry point; `scripts/baseline-capture.sh` for the baseline tooling

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Dataverse needs five components: Payara (app server), Solr (search), PostgreSQL via RDS (metadata), S3 (file content), Apache (proxy and SSL).
- Terraform provisions the AWS infrastructure; Ansible configures what runs on it.
- The three repos are `terraform-dataverse`, `dataverse-ansible`, and `dataverse-infrastructure`.
- Infrastructure as code makes the system reproducible, reviewable, and rebuildable.
- Many decisions in this codebase were shaped by the 5.14 to 6.8 migration -- that context appears throughout the lesson.

::::::::::::::::::::::::::::::::::::::::::::::::
