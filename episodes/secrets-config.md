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

- Database password for the RDS instance (`dataverse_postgresql_password`)
- Dataverse admin password (`dataverse_adminpass`)
- EZID credentials (production DOI registration)

Notably absent: **AWS credentials for S3 writes are not a vaulted secret at all.** The
EC2 instance authenticates to S3 through an IAM instance profile (`s3.use_iam_role: true`
in `group_vars`) -- there's no access key to leak in the first place, which is the safer
design and worth naming as deliberate, not an oversight.

There's no separate `group_vars/all/vault.yml` file -- secrets are inline, encrypted
in place inside the same flat `group_vars/<env>.yml` files as everything else, using
`!vault |` blocks:

```yaml
dataverse_adminpass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          663365396...
```

To decrypt and view one value, or a whole file:

```bash
ansible-vault view group_vars/dev.yml
```

To edit a vaulted value in place:

```bash
ansible-vault edit group_vars/dev.yml
```

Both commands prompt for the vault password (the `.vault-password` file from Episode 2).
That file itself is never committed to the repository.

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

Instead, test environments use a self-signed certificate. In `group_vars`, this is
controlled by a nested key under `letsencrypt.certbot`, not a flat variable:

```yaml
letsencrypt:
  certbot:
    test_cert: true   # dev.yml, test.yml -- staging certs during iterative rebuilds
    # test_cert: false  # TEMPLATE.yml, staging.yml default -- flip to false only at production cutover (Phase 7)
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

In `group_vars`, this isn't one variable but two nested blocks -- `pid:` (the identifier
format) and `doi:` (the registration service):

```yaml
pid:
  authority: "10.5072"
  protocol: doi
  shoulder: "FK2/"

doi:
  provider: FAKE     # dev.yml, test.yml
  # provider: EZID   # production, not configured yet -- no production group_vars file exists
```

The FAKE provider is used in all non-production environments throughout the migration.
The switch to real EZID happens only at Phase 7 (DNS cutover) -- and since there's no
`production.yml` yet, that switch requires writing production config, not just flipping
a value in an existing file.

## Environment configuration summary

| Setting | dev / test (tim, jamie) | staging / TEMPLATE default | Production |
|---|---|---|---|
| SSL certificate | Self-signed (`test_cert: true`) | Let's Encrypt (`test_cert: false`) | Not yet configured |
| DOI provider | FAKE (no external calls) | FAKE | EZID (planned, Phase 7) |
| Vaulted? | `dev.yml` partially vaulted | `staging.yml`/`test.yml` have unvaulted `CHANGE_ME_USE_VAULT` placeholders | -- |
| S3 access | IAM instance profile (no credentials to vault) | same | same |

::::::::::::::::::::::::::::::::::::: callout

### Vaulting is a work in progress, not a finished state

Don't assume every environment's secrets are actually encrypted right now. `test.yml`
currently has `dataverse_adminpass: "CHANGE_ME_USE_VAULT"` and
`dataverse_postgresql_password: "CHANGE_ME_USE_VAULT"` in plaintext -- literal placeholder
strings, not real secrets, but also not vaulted. `all.yml` (the shared defaults every
environment inherits unless it overrides them) has real plaintext defaults too, like
`adminpass: admin`. Treat "is this value vaulted in `dev.yml`" and "is this value vaulted
in `test.yml`" as two separate questions with two different answers today.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Spot the difference

Open `group_vars/dev.yml` and `group_vars/test.yml` in the `dataverse-ansible` repo.

1. Which top-level keys are shared structure (same key, different value) between the two files?
2. Find `dataverse_adminpass` in each file. Is it vaulted (`!vault |`) in both? If not, which one isn't, and what does that tell you about deploy-readiness?
3. Find the `pid:`/`doi:` blocks. What's the DOI provider in each?

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Secrets are vaulted inline inside `group_vars/<env>.yml` (`!vault |` blocks) -- there is no separate `group_vars/all/vault.yml` file.
- S3 access uses an IAM instance profile, not vaulted AWS credentials -- there's no access key to leak.
- Test environments use self-signed certificates (`letsencrypt.certbot.test_cert: true`); production would use Let's Encrypt.
- DOI config is two nested blocks, `pid:` and `doi:`, not one flat provider variable. FAKE is used everywhere non-production; EZID is planned for Phase 7 and has no group_vars file yet.
- Vaulting is incomplete today: `test.yml` has real unvaulted `CHANGE_ME_USE_VAULT` placeholders. Don't assume every environment is equally secret-safe.

::::::::::::::::::::::::::::::::::::::::::::::::
