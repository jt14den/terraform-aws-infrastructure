---
title: "Secrets and Environment Configuration"
teaching: 20
exercises: 8
---

:::::::::::::::::::::::::::::::::::::: questions

- How are secrets kept out of the repository?
- What configuration is different between staging and production?
- What is a FAKE PID provider and why does it exist?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain what Ansible Vault does and how to use it.
- Describe the differences between test and production configuration.
- Identify which config values are environment-specific vs. shared.
- View a vaulted file's contents using the vault password.

::::::::::::::::::::::::::::::::::::::::::::::::

## Ansible Vault

Ansible Vault is a tool for encrypting sensitive values so they can be stored in the
repository without exposing secrets. The vault password decrypts them at playbook runtime.

Common secrets stored in the vault for this project:

- Database password for the RDS instance
- Dataverse admin API token
- EZID credentials (production DOI registration)
- AWS credentials for the Dataverse application to write to S3

To view a vaulted file:

```bash
ansible-vault view group_vars/all/vault.yml
```

To edit it:

```bash
ansible-vault edit group_vars/all/vault.yml
```

Both commands prompt for the vault password. The vault password itself is stored separately
and shared with authorized operators -- it is never committed to the repository.

::::::::::::::::::::::::::::::::::::: callout

### Never commit unencrypted secrets

If you accidentally add a plaintext secret to the repo, treat it as compromised and rotate it.
Remove it from git history using `git filter-branch` or `git filter-repo`, then notify the team.
The vault exists to prevent this -- if in doubt, vault it.

::::::::::::::::::::::::::::::::::::::::::::::::

## Test cert vs. real SSL

In production, Dataverse serves HTTPS using a Let's Encrypt certificate obtained via Certbot.
Certbot contacts Let's Encrypt's servers to verify domain ownership and issue the certificate.

In test environments, running Certbot would:

- Fail if the test EC2 instance is not reachable on a public domain
- Hit Let's Encrypt rate limits during rapid rebuilds
- Register a real certificate for a temporary hostname

Instead, test environments use a self-signed certificate. In `group_vars`, this is controlled by:

```yaml
dataverse_use_test_cert: true   # test environments
dataverse_use_test_cert: false  # production
```

When `test_cert: true`, Ansible generates a self-signed certificate locally and skips
Certbot entirely. Browsers will show a security warning for self-signed certs -- that is expected.

## FAKE PID provider

DOI registration is the process of minting a persistent identifier for a dataset and
registering it with a DOI service (UCLA uses EZID, which registers with DataCite).

In production, every published dataset gets a real, resolving DOI. In test environments,
you do not want to:

- Mint real DOIs that point to a test server
- Hit EZID's API during rapid rebuilds and testing
- Risk cluttering the production DOI namespace with test records

Dataverse has a built-in FAKE PID provider for exactly this purpose. It generates DOI-like
identifiers (they look like DOIs but do not resolve) without contacting any external service.

In `group_vars`:

```yaml
dataverse_pid_provider: "FAKE"    # test environments
dataverse_pid_provider: "EZID"    # production
```

The FAKE provider is used in all non-production environments throughout the migration.
The switch to real EZID happens only at Phase 7 (DNS cutover).

## Environment configuration summary

| Setting | Test (tim, jamie) | Production |
|---|---|---|
| SSL certificate | Self-signed (`test_cert: true`) | Let's Encrypt via Certbot |
| DOI provider | FAKE (no external calls) | EZID (DataCite registration) |
| Database | RDS dev instance | RDS production instance |
| S3 bucket | `tim-dataverse-dev-storage-...` | Production bucket |
| Payara heap | 2GB | 4GB |

::::::::::::::::::::::::::::::::::::: challenge

### Spot the difference

Open `group_vars/all/` in the `dataverse-ansible` repo.

1. Which variables are shared across all environments?
2. Which variables are overridden per environment?
3. Find the variable that controls the DOI provider. What is its default value?

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Secrets live in Ansible Vault; the vault password is shared with authorized operators only.
- Test environments use self-signed certificates; production uses Let's Encrypt.
- The FAKE PID provider generates DOI-like identifiers without contacting EZID.
- The switch from FAKE to real EZID happens only at Phase 7 production cutover.
- `group_vars` is where environment-specific overrides live -- check here first when behavior differs between environments.

::::::::::::::::::::::::::::::::::::::::::::::::
