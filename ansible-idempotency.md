---
title: "Ansible: Configuration and Idempotency"
teaching: 25
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What does `dataverse-ansible` configure on the EC2 instance?
- What does idempotency mean in practice, and why does it matter for operations?
- How are environment-specific values managed without duplicating configuration?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain the structure of the `dataverse-ansible` role.
- Describe what `group_vars` does and how environment overrides work.
- Run the playbook and read the output.
- Explain the difference between module-level idempotency and playbook-level re-run safety, and why this role only has the first.

::::::::::::::::::::::::::::::::::::::::::::::::

## What Ansible does

Ansible is responsible for configuration -- the "what is installed and how is it set up" layer.
After Terraform creates the EC2 instance, Ansible connects to it over SSH and:

- Installs system packages (Java, Python, curl, and others)
- Installs and configures Payara
- Installs and configures Solr
- Installs and configures Apache with SSL
- Deploys the Dataverse WAR file into Payara
- Applies Dataverse configuration via the Dataverse API
- Sets JVM options in Payara for Dataverse to use

All of this is defined in the `dataverse-ansible` role -- a structured collection of tasks,
templates, handlers, and variable files.

## Role structure

The `dataverse-ansible` repo is an Ansible role. The key directories are:

```
dataverse-ansible/
  site.yml        <- the entry point playbook
  tasks/          <- what to do (~90 flat task files, one per concern)
  handlers/       <- actions triggered by other tasks (e.g., restart Payara)
  templates/      <- config file templates with variable substitution
  defaults/       <- default variable values
  group_vars/     <- environment-specific overrides
```

`site.yml` at the repo root is the entry point -- the repo's own comment describes
the whole thing as "this repository itself is the Dataverse ansible role." Instead of one
`tasks/main.yml` dispatching to sub-files, `tasks/` holds a flat collection of per-service
files (`payara.yml`, `solr.yml`, `dataverse-prereqs.yml`, and so on) that `site.yml` pulls
in directly. There's no top-level `vars/` directory in this role -- just `defaults/` for
role defaults and `group_vars/` for environment overrides.

## Idempotency (the concept) vs. this role (the reality)

An **idempotent** operation produces the same result whether you run it once or ten times.
For Ansible, this means: running the playbook on an already-configured server should make
no changes, because everything is already in the desired state. Individual Ansible
*modules* are generally built this way:

```yaml
- name: Install Java
  dnf:
    name: "java-{{ java.version }}-openjdk-devel"
    state: present   # "present" means "install if not already installed"
```

The `dnf` module checks whether the package is installed before trying to install it.
If it is already installed, Ansible reports `ok` and moves on. If not, it installs and
reports `changed`. That's real, and it's how most tasks in `dataverse-ansible` behave.

**But module-level idempotency is not the same as playbook-level re-run safety, and this
role does not have the second one.** The repo's own operating rule, stated plainly in
`CONTEXT.md`: *"Ansible is NOT idempotent. You must fully destroy the environment before
re-running. Do not re-run ansible-playbook against an existing instance."* In practice
that means: after any `make rebuild`, if something fails partway through, the fix is
`terraform destroy` and start over -- not "just run `make ansible` again and let idempotent
modules sort it out."

::::::::::::::::::::::::::::::::::::: callout

### Why re-running isn't safe here

`shell` and `command` tasks run every time unless guarded with `creates:` or `when:` --
and a role this size has a number of them: first-boot Dataverse API calls, database
schema bootstrapping, Let's Encrypt certificate issuance. Any one of those re-running
against an already-configured instance can fail outright or leave the system in a state
none of the individual `ok`/`changed` reports would have predicted. The safe mental model
for this specific role is **destroy-and-rebuild, not re-run-in-place** -- treat the
per-module idempotency as a nice property of individual steps, not a guarantee about the
playbook as a whole.

::::::::::::::::::::::::::::::::::::::::::::::::

## group_vars and environment overrides

The `group_vars/` directory holds variable files that override defaults for specific
environments. The real files are `all.yml` (non-secret defaults shared everywhere),
`dev.yml`, `test.yml`, `staging.yml`, and `TEMPLATE.yml` (a starting point for a new
environment) -- there is no `production.yml` yet.

Dev and test both point at the FAKE DOI provider, but the real variables are nested
under `pid:` and `doi:` blocks, not a single flat `dataverse_doi_provider` key:

```yaml
# group_vars/dev.yml
pid:
  authority: "10.5072"
  protocol: doi
  shoulder: "FK2/"

doi:
  provider: FAKE
  baseurl: "https://mds.test.datacite.org/"
  username: "testaccount"
  password: "notmypassword"   # not vaulted in dev -- see Episode 6
```

The same playbook runs against every environment; the variables control which behavior
each environment gets. This is how we keep dev, test, and staging separate without
maintaining separate copies of the role.

## The inventory

Ansible needs to know which host to connect to. The inventory file lists hosts and their
connection details. For this project, the inventory is generated by Terraform output,
and the real EC2 user is `rocky` (Rocky Linux 9), not `ubuntu`:

```
[dataverse]
ec2-12-34-56-78.us-west-2.compute.amazonaws.com ansible_user=rocky ansible_ssh_private_key_file=~/.ssh/dataverse-key.pem
```

The Makefile generates and uses this inventory automatically when you run `make rebuild`.
`CONTEXT.md` flags this file specifically: it's gitignored and regenerated by Terraform
on every `apply` -- never edit it by hand, your edits will just be overwritten.

::::::::::::::::::::::::::::::::::::: challenge

### Run the real playbook and read the output

There is no `--check`-mode wrapper for this role today -- `make ansible ENV=tim` runs the
real thing:

```bash
cd dataverse-infrastructure
make ansible ENV=tim
```

(Under the hood: `cd dataverse-ansible && ansible-playbook -i ../<inventory> site.yml`.)

Read through the output and identify:

1. Which tasks report `changed` and which report `ok`?
2. Find one `shell` or `command` task in `tasks/`. Does it have a `creates:` or `when:` guard?
3. If this run failed halfway through, per `CONTEXT.md` what is the supported way to recover?

:::::::::::::::::::::::::::::::::: solution

`ok` means the module checked state and found nothing to do. `changed` means it modified
something. A freshly-provisioned instance should show mostly `changed` on first run;
re-running against the *same still-fresh* instance would show more `ok`s for the guarded
tasks -- but that's not a scenario this role is meant to be run in twice.

Some `shell`/`command` tasks are guarded, some aren't -- that inconsistency is exactly
why the repo-wide rule exists.

Per `CONTEXT.md`: destroy and rebuild (`make rebuild`), not re-run in place. There is no
supported partial-recovery path.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Module idempotency vs. playbook safety

In your own words: what's the difference between "this `dnf` task is idempotent" and
"this playbook is safe to re-run against a live instance"? Why can the first be true
while the second is false?

:::::::::::::::::::::::::::::::::::: solution

Module idempotency is a per-task property: a well-written module checks current state
before acting, so running it twice in a row causes no harm. Playbook-level re-run safety
depends on *every* task in the run having that property, including `shell`/`command`
tasks that don't check anything by default. One unguarded task -- a schema bootstrap, a
cert request, a first-boot API call -- is enough to make the whole playbook unsafe to
re-run, even though most of its individual tasks are perfectly idempotent on their own.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- `dataverse-ansible` installs and configures Payara, Solr, Apache, and Dataverse; `site.yml` is the entry point, and the whole repo is treated as one role.
- Individual modules (like `dnf`) are idempotent -- but this role, as a whole, is **not** safe to re-run against a live instance. The operating rule is destroy-and-rebuild, not re-run-in-place.
- `group_vars` (`all.yml`, `dev.yml`, `test.yml`, `staging.yml`) provides environment-specific values without duplicating the role; DOI config lives under nested `pid:`/`doi:` blocks, not flat keys.
- The Ansible inventory is generated from Terraform output (`ansible_user: rocky`) -- gitignored, regenerated on every `apply`, never hand-edited.
- Unguarded `shell`/`command` tasks are the reason re-running isn't safe -- prefer modules, and guard shell tasks with `creates:`/`when:` when you can't avoid them.

::::::::::::::::::::::::::::::::::::::::::::::::
