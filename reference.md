---
title: "Reference: Glossary"
---

This glossary covers the tools, concepts, and services that appear throughout this lesson.
Terms are grouped by layer. Cross-references point to related terms in the same glossary.

---

## Application layer

**Dataverse**
An open-source research data repository platform developed by Harvard's IQSS. It manages
dataset publishing, versioning, access control, and persistent identifier registration.
UCLA's instance runs at dataverse.library.ucla.edu. See: [Payara], [Solr], [WAR file].

**Payara**
A Jakarta EE application server and the runtime container for Dataverse. A community fork
of GlassFish maintained for modern Java compatibility. Dataverse runs as a WAR file deployed
inside Payara. Most Dataverse configuration is applied via Payara JVM options. See: [GlassFish], [WAR file], [JVM options].

**GlassFish**
The original Jakarta EE application server from which Payara was forked. Some Dataverse
documentation and older issues reference GlassFish commands. They are mostly
interchangeable with Payara -- the `asadmin` CLI and domain structure are the same.

**WAR file**
Web Application Archive. A packaged Java web application -- a JAR file with a specific
structure that application servers like Payara know how to deploy. Dataverse is distributed
as a WAR file. Ansible downloads and deploys it during installation.

**Solr**
An open-source full-text search engine (from Apache). Dataverse uses it to power all search
and browse operations. Solr maintains its own index built from the database. The index must
be explicitly rebuilt after any database restore. See: [Solr reindex].

**Solr reindex**
The process of rebuilding Solr's search index from the database. Triggered by `make reindex ENV=<env>`.
Required after any database restore. A Dataverse instance with a stale or empty Solr index
will appear to have no datasets. See: [Solr].

**Payara admin console**
A web UI for managing Payara, accessible on port 4848. Used to inspect JVM options,
view deployment status, and monitor thread pools. Should not be publicly accessible.

**JVM options**
Java Virtual Machine startup parameters passed to Payara. Dataverse uses JVM options
(prefixed with `-D`) to configure database connection, S3 bucket, DOI provider, and more.
These are set by Ansible and visible in Payara's `domain.xml`. Example:
`-Ddataverse.files.s3-bucket-name=ucla-dataverse-storage`.

**Jakarta EE**
A set of Java enterprise specifications (formerly Java EE). Dataverse targets the Jakarta EE
platform. Payara implements the Jakarta EE spec as its runtime. See: [Payara], [GlassFish].

**Apache httpd**
The HTTP server used as a reverse proxy in front of Payara. Handles SSL termination,
URL rewriting, and HTTP-to-HTTPS redirects. Configured by Ansible via templates.

**Certbot**
A tool that automates obtaining and renewing Let's Encrypt SSL certificates. Used in
production. Skipped in test environments in favor of a self-signed certificate. See: [Let's Encrypt], [test cert].

**Let's Encrypt**
A free, automated certificate authority. Certbot obtains SSL certificates from Let's Encrypt
for the production Dataverse instance. See: [Certbot].

---

## Persistent identifiers

**DOI**
Digital Object Identifier. A persistent identifier assigned to published datasets.
UCLA Dataverse registers DOIs through EZID using the DataCite DOI service.
Example: `doi:10.25346/S6/XYZ`. See: [EZID], [FAKE PID provider], [DataCite].

**EZID**
A DOI registration service provided by the California Digital Library. UCLA Dataverse uses
EZID to mint and manage DOIs for published datasets. EZID integrates with DataCite.
Only active in the production environment. See: [DOI], [DataCite], [FAKE PID provider].

**DataCite**
The global DOI registration agency for research data. EZID registers DOIs with DataCite on
behalf of UCLA. DOIs registered through DataCite resolve globally. See: [DOI], [EZID].

**FAKE PID provider**
A built-in Dataverse provider that generates DOI-like identifiers without contacting any
external service. Used in all non-production environments to avoid minting real DOIs during
testing and development. The identifiers look like DOIs but do not resolve. See: [DOI], [EZID].

**PID**
Persistent Identifier. The general term for identifiers like DOIs that remain stable over time.
Dataverse's configuration uses the term "PID provider" to refer to the DOI registration service.

**test cert**
A self-signed SSL certificate used in test environments instead of a Let's Encrypt certificate.
Triggers a browser security warning but is functionally equivalent for testing purposes.
Controlled by `dataverse_use_test_cert: true` in Ansible group_vars.

---

## Infrastructure (AWS)

**EC2**
Elastic Compute Cloud. AWS's virtual machine service. The Dataverse application, Payara, Solr,
and Apache all run on a single EC2 instance. Instance type and size are set in Terraform variables.

**RDS**
Relational Database Service. AWS's managed database service. Used to run the PostgreSQL
database for Dataverse. AWS handles backups, patching, and availability. The database
hostname looks like `dev-dataverse-db.cb4k4a6gqn27.us-west-2.rds.amazonaws.com`.

**S3**
Simple Storage Service. AWS object storage. Dataverse stores all uploaded file content in S3.
Separate S3 buckets are used for the production instance, each operator's dev environment,
and migration assets. See: [S3 object], [S3 bucket].

**S3 bucket**
A named container for S3 objects. Each environment has its own bucket. The production bucket
name is different from the dev buckets. The bucket name is a Payara JVM option.

**S3 object**
A single file stored in S3. Dataverse creates one S3 object per uploaded file. The object
key (path) is derived from the storage identifier stored in the database.

**Elastic IP (EIP)**
A static IP address reserved in AWS and attached to an EC2 instance. Unlike a regular EC2
public IP, an Elastic IP does not change when the instance is stopped, started, or terminated
and replaced. This makes it possible to rebuild the EC2 instance without updating DNS or
Ansible inventory. See: [make rebuild].

**IAM**
Identity and Access Management. AWS's permission system. The EC2 instance uses an IAM role
with a policy that allows it to read and write the Dataverse S3 bucket.

**IAM role**
A set of permissions assigned to an AWS resource (like an EC2 instance). The EC2 instance
assumes its IAM role to interact with S3 without needing hardcoded credentials.

**Security group**
An AWS firewall that controls inbound and outbound traffic for an EC2 instance. Defined in
Terraform. The Dataverse security group opens ports 22 (SSH), 80 (HTTP), and 443 (HTTPS)
and restricts port 4848 (Payara admin) to trusted IPs only.

**AWS CLI**
The command-line interface for AWS. Used to inspect S3 buckets, check IAM identity, and
run baseline scripts. Configured via `~/.aws/credentials` and `~/.aws/config`.

**AWS profile**
A named set of credentials and configuration in `~/.aws/credentials`. This project uses
`ucla-library-dsc`. Set `export AWS_PROFILE=ucla-library-dsc` before running Makefile targets.

**VPC**
Virtual Private Cloud. An isolated network within AWS. All project resources (EC2, RDS, S3 endpoints)
are within the same VPC. Network-level access between resources is controlled by security groups
and subnet routing.

**AZ**
Availability Zone. A physically separate data center within an AWS region. RDS can be configured
across multiple AZs for high availability. The dev environment uses a single AZ to reduce cost.

---

## Terraform

**Terraform**
An infrastructure-as-code tool from HashiCorp. Describes AWS resources in HCL (HashiCorp
Configuration Language) files and manages their lifecycle (create, update, destroy).
See: [Terraform state], [remote state], [terraform plan].

**HCL**
HashiCorp Configuration Language. The syntax Terraform uses for configuration files.
`.tf` files are written in HCL.

**Terraform state**
A JSON file (`terraform.tfstate`) that records what resources Terraform has created and
their current configuration. Terraform compares state against the configuration to determine
what to create, change, or destroy. See: [remote state].

**Remote state**
Storing the Terraform state file in a shared location (S3) rather than locally. Allows
multiple operators to share the same state. Prevents two people from making conflicting
changes. Used in this project; state is in `s3://ucla-dataverse-migration-assets/terraform/state/`.

**terraform plan**
A Terraform command that shows what would change if you ran `terraform apply` -- without
making any changes. Always run `terraform plan` before `terraform apply` to review the diff.

**terraform apply**
A Terraform command that creates, updates, or destroys resources to match the configuration.
Always review `terraform plan` first.

**terraform destroy**
A Terraform command that destroys all resources managed by the current configuration.
`make rebuild` calls `terraform destroy` for the EC2 instance as its first step.

**module**
A reusable Terraform configuration component. The `terraform-dataverse` repo uses modules
to share resource definitions between Tim's and Jamie's environments.

**tfvars**
A Terraform variable values file (`terraform.tfvars`). Contains environment-specific values
(instance size, region, bucket names) without secrets.

---

## Ansible

**Ansible**
An agentless configuration management tool. Uses SSH to connect to servers and apply
configuration defined in YAML playbooks and roles. Used in this project to install and
configure Dataverse and all its dependencies. See: [playbook], [role], [task], [idempotency].

**Playbook**
An Ansible YAML file that defines what to do on which hosts. Calls roles and tasks.
The main playbook for this project runs the `dataverse-ansible` role against the target host.

**Role**
A structured, reusable unit of Ansible configuration. The `dataverse-ansible` role installs
and configures Payara, Solr, Apache, and Dataverse. Roles have a specific directory structure
(`tasks/`, `handlers/`, `templates/`, `defaults/`, `vars/`).

**Task**
A single action in an Ansible playbook or role. Examples: install a package, copy a file,
restart a service. Tasks are defined in YAML and use Ansible modules.

**Module**
An Ansible built-in operation. Examples: `apt` (install packages), `template` (render and copy
a config file), `service` (start/stop/restart a service). Prefer modules over `shell` or `command`
for idempotency.

**Handler**
An Ansible task that only runs when notified by another task. Used for restarts: a task that
modifies a config file notifies the handler, which restarts the service at the end of the play.
Handlers only run once per play even if notified multiple times.

**Idempotency**
The property of an operation that produces the same result whether run once or many times.
An idempotent Ansible playbook makes no changes on a system that is already in the desired state.
Ansible reports `ok` for tasks where no change was needed. See: [Ansible].

**group_vars**
A directory of YAML variable files that apply to specific host groups. Used to provide
environment-specific configuration overrides without modifying the role itself.
Files in `group_vars/all/` apply to all hosts; files named after a group apply only to that group.

**Ansible Vault**
A tool for encrypting sensitive values in Ansible files. Vaulted files can be stored in the
repository -- they are decrypted at runtime using the vault password. Used for database
passwords, API tokens, and EZID credentials.

**Inventory**
A file or directory that tells Ansible which hosts to connect to and how. For this project,
the inventory is generated from Terraform output and lists the EC2 instance's IP and SSH key.

**--check mode**
An Ansible dry-run flag. With `--check`, Ansible reports what it would change without
making any changes. Useful for auditing configuration drift.

---

## Operations

**make rebuild**
The primary Makefile target for recreating an environment from scratch. Runs `terraform destroy`,
`terraform apply`, generates a new inventory, and runs the Ansible playbook. See: [Makefile].

**make baseline**
Captures a timestamped JSON snapshot of dataset, file, user, and S3 counts from a running
Dataverse instance. Used to verify data integrity before and after migration.

**make reindex**
Triggers a full Solr reindex of all datasets from the database. Required after any database
restore. See: [Solr reindex].

**Makefile**
A file containing named commands (`targets`) that can be run with `make <target>`.
The `dataverse-infrastructure` Makefile wraps Terraform, Ansible, and the baseline scripts
behind named targets. See: [make rebuild], [make baseline], [make reindex].

**Baseline**
A timestamped JSON snapshot of system state captured by `make baseline`. Fields include
dataset count, file count, user count, and S3 object count/bytes. Pre- and post-migration
baselines are compared to verify data integrity.

**DNS TTL**
Time To Live for a DNS record. Controls how long DNS resolvers cache the record before
re-querying. Reduced to 60 seconds before cutover so DNS changes propagate quickly after
the A record is switched.

**Maintenance window**
A scheduled period of downtime for the migration cutover. Users are notified in advance.
During the window, the database is frozen, migrated, and the DNS record is switched.

**Rollback**
Reverting to the previous state after a failed migration. Rollback is possible by switching
the DNS A record back to the old production IP. The old instance remains running until
rollback is no longer needed.

**Shibboleth**
A federated identity protocol used for campus single sign-on (SSO). UCLA uses Shibboleth
for library authentication. The Dataverse migration includes configuring Shibboleth via
Ansible and registering the SP metadata with the UCLA campus Identity Provider (IdP).

**SP (Service Provider)**
The application side of a Shibboleth SSO connection. Dataverse acts as the SP. The SP
registers metadata with the IdP and receives user attributes after authentication.

**IdP (Identity Provider)**
The authentication service that verifies user identity in a Shibboleth SSO setup.
UCLA's campus IdP handles authentication for library services including Dataverse.

**mod_shib**
The Apache module that handles Shibboleth SP authentication. Installed and configured
by Ansible in Phase 5 of the migration.

---

## Migration-specific

**5.14**
The current production Dataverse version at UCLA Library at the start of this migration.
Released 2023. Uses Java 11, an older Solr schema, and a different PID provider configuration.

**6.8**
The target Dataverse version for the migration. Requires Java 17, a new Solr schema,
and updated S3/DOI configuration.

**Migration arc**
The 7-phase plan for migrating the UCLA Dataverse instance from 5.14 to 6.8.
Each phase gates the next with test suite and baseline validation. See Episode 9.

**Phase gate**
The requirement that tests and baselines pass before proceeding to the next phase.
If the gate fails, the current phase is not complete regardless of what was deployed.

**Production baseline**
The baseline snapshot captured by Jamie against the live 5.14 production instance before
any migration work begins. Stored durably in S3. Used as the comparison target after
Phase 7 cutover.
