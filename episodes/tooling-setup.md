---
title: "Tooling Setup"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::::::::::::: callout

### Dataverse extension: case study

This is outside the six-episode public core. Operational commands and historical
status descriptions are examples for analysis, not a current production runbook.
No AWS or private access is required to discuss the included material. Only
authorized maintainers using a reviewed, current runbook should operate the
actual service. Do not run these commands as workshop exercises.

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::: questions

- What tools do I need installed to work with this infrastructure?
- How do the three repositories fit together on my local machine?
- How do I configure AWS credentials for the correct profile?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify the inputs an authorized maintainer needs for onboarding.
- Explain the nested repository layout.
- Explain why existing Vault ciphertext needs its existing password.
- Distinguish configuration, authentication, connectivity, and privilege escalation.

::::::::::::::::::::::::::::::::::::::::::::::::

## Operator onboarding, separate from learner setup

The [public setup](../learners/setup.md) uses Pixi and Podman. The Dataverse
orchestration checkout historically used Terraform, Ansible, AWS CLI, Make, and
uv. Maintainers must read its current lockfiles and runbook for compatible
versions; the Potree environment is not automatically the Dataverse environment.

The orchestration repository nests `terraform-dataverse/` and `dataverse-ansible/`.
A local Makefile inspected on 2026-10-04 has a `bootstrap` target that clones those
children. This explains the layout; public learners need not clone private code.
The paths below are relative to the **orchestration repository root**:

```text
terraform-dataverse/environments/<operator>/  infrastructure configuration
dataverse-ansible/site.yml                   configuration entry point
dataverse-ansible/group_vars/                environment inputs
Makefile                                    orchestration
```

Before any real operation, maintainers must establish the intended AWS account,
role and profile, resource ownership, SSH authorization, inventory location,
and current operating procedure. A profile name alone does not prove the account
or permission scope. Do not run initialization, planning, or connectivity commands
against cloud systems as part of this workshop.

Terraform initialization installs providers and configures a backend; validation
and plan review are separate checks. Plans normally read remote state and APIs
and can execute configured data sources, so calling them universally safe would
be misleading. Use the supplied analysis exercise in
[Terraform](terraform-infrastructure.md) without cloud access.

## Setting up Ansible Vault

Existing project secrets require the **existing password that encrypted them**.
Authorized maintainers must obtain it through the project's established process.
This lesson does not specify that process. Generating a random replacement will
not decrypt existing ciphertext, and can destroy access if it overwrites the
only correct local password file.

If the project's current configuration expects `dataverse-ansible/.vault-password`,
store the supplied password there with restrictive permissions (0600), outside
version control. Verify the ignore rule without displaying the password. Do not
replace an existing file. A password manager or other established recovery process
must provide continuity beyond one laptop.

For a **new disposable practice vault**, use the dummy lab in
[Secrets and environment configuration](secrets-config.md). It creates a fresh
private temporary directory and never uses project secrets.

## What connectivity checks establish

A valid AWS identity does not demonstrate SSH reachability. An inventory that
names a host does not demonstrate authentication. An Ansible `ping` result over
SSH demonstrates connection and Python module execution for that user; it does
not by itself prove privilege escalation. A separate approved `become` test must
verify the effective user. None of these checks is run against live targets here.

::::::::::::::::::::::::::::::::::::: challenge

### Explain two onboarding failures

1. The Makefile expects nested repositories, but they were cloned as siblings.
   What should you inspect before moving files?
2. A playbook cannot decrypt an inline Vault value. Would a new random password
   fix it? What should an authorized maintainer establish?

:::::::::::::::::::::::::::::::::: solution

1. Read the current Makefile's paths and bootstrap target, and check for existing
   work before rearranging anything. Relative paths depend on the checkout layout.
2. No. Establish which vault identity/password encrypted the value and obtain the
   existing password through the project process. Check the configured password
   source and permissions without printing secrets. A new password is only for
   new ciphertext or a deliberate rekey using the old password.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Public learners need only the core setup; operator onboarding is separate.
- Repository paths, credentials, inventory, and permissions must match the current project.
- Existing Vault ciphertext needs its existing password, not a newly generated one.
- Configuration, connectivity, and privilege escalation are distinct checks.
::::::::::::::::::::::::::::::::::::::::::::::::
