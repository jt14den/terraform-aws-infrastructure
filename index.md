---
site: sandpaper::sandpaper_site
---

This lesson traces the infrastructure that runs the UCLA Library Dataverse instance. It covers the AWS resources that provide compute, storage, and networking; the Ansible configuration that installs Dataverse and its dependencies; and the tooling that makes rebuilding and migrating the system repeatable.

It is written for people working in or being onboarded to UCLA Library data services: DSC staff, DataSquad students, and anyone who will be operating or handing off this infrastructure. The assumption is comfort with the command line and some exposure to cloud services or configuration management, but not infrastructure expertise.

By the end of this lesson you will be able to:

- Describe the components that make up a running Dataverse instance and what each does
- Navigate the three repositories that manage the UCLA Dataverse infrastructure
- Explain the division of responsibility between Terraform (infrastructure) and Ansible (configuration)
- Run the key Makefile targets for daily operations: `rebuild`, `baseline`, `reindex`
- Read test output and baseline comparisons to verify the system is in a known-good state
- Understand the 7-phase migration plan and what each phase accomplishes

## Who this is for

- **Data Science Center staff** coming up to speed on what the infrastructure team built and why
- **DataSquad students** supporting data services and infrastructure work
- **Incoming operators** taking over responsibility for the Dataverse instance
- **Tim and Jamie** using this as a structured way to document decisions made during the 5.14 to 6.8 migration

## Prerequisites

- Comfortable with the command line (navigating directories, running commands)
- Basic familiarity with Git (clone, commit, push)
- An AWS account with credentials for the `ucla-library-dsc` profile, or access to the team's shared environment

See the [Setup](learners/setup.md) page for installation instructions.
