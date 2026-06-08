---
title: "The Makefile: Daily Operations"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What can I do from the Makefile?
- What does `make rebuild` actually do, step by step?
- When do I run `make baseline` vs. `make reindex`?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- List the main Makefile targets and what each does.
- Trace the steps of `make rebuild` in order.
- Run `make baseline` and inspect the snapshot it produces.
- Know when each operational target should be used.

::::::::::::::::::::::::::::::::::::::::::::::::

## Why a Makefile

The `dataverse-infrastructure` Makefile is the daily operations interface for this project.
It wraps Terraform, Ansible, and the baseline scripts behind named targets so you rarely
need to invoke those tools directly.

A Makefile target is a named command. Running `make <target>` executes the commands
associated with that target. For example:

```bash
make rebuild ENV=tim
```

...runs a sequence of Terraform and Ansible commands in the right order, with the right
arguments, for Tim's environment.

The Makefile lives in `dataverse-infrastructure/`. Most targets require an `ENV` argument
that specifies which operator environment to work with.

## Main targets

### `make rebuild`

Destroys and recreates the EC2 instance, then runs Ansible to configure it.

Steps in order:

1. `terraform destroy` -- terminates the existing EC2 instance (RDS and S3 are preserved)
2. `terraform apply` -- provisions a new EC2 instance with the Elastic IP attached
3. Generates a new Ansible inventory from Terraform output
4. Waits for the instance to become reachable via SSH
5. Runs the Ansible playbook to install and configure all services
6. Runs the Dataverse first-boot API configuration
7. Runs the test suite to verify the instance is healthy

Use `make rebuild` when:

- Starting fresh after a failed or degraded environment
- Testing infrastructure changes that require a clean EC2 instance
- Validating a new Ansible configuration from scratch

```bash
make rebuild ENV=tim
```

::::::::::::::::::::::::::::::::::::: callout

### RDS and S3 survive a rebuild

`make rebuild` destroys only the EC2 instance. The RDS database and S3 bucket are
not touched. This is intentional -- data persists across rebuilds. After rebuild,
Dataverse reconnects to the same database and storage it had before.

If you want to restore the database to a specific backup as part of the rebuild,
you need to do that as a separate step before running Ansible.

::::::::::::::::::::::::::::::::::::::::::::::::

### `make baseline`

Captures a timestamped JSON snapshot of the current state of the Dataverse instance.
The snapshot includes:

- Dataset count (published, draft, total)
- File count
- User count
- Dataverse collection count
- S3 object count and total bytes

```bash
make baseline ENV=tim
```

The snapshot is saved to `baselines/` and optionally uploaded to S3 if
`BASELINE_UPLOAD_BUCKET` is set:

```bash
BASELINE_UPLOAD_BUCKET=ucla-dataverse-migration-assets make baseline ENV=jamie
```

Jamie's production baseline (run before migration begins) is the anchor for the
Phase 7 post-cutover comparison. Upload it to S3 so it is durable and shared.

### `make baseline-compare`

Compares two baseline snapshots and reports differences:

```bash
make baseline-compare BEFORE=baselines/before.json AFTER=baselines/after.json
```

A passing comparison shows matching counts across all fields.
Any discrepancy in dataset or file counts after migration is a problem to investigate
before declaring the migration complete.

### `make reindex`

Triggers a full Solr reindex of all datasets from the database.

```bash
make reindex ENV=tim
```

Run this after:

- Any database restore
- A `make rebuild` that restored from a backup
- Noticing that Dataverse search returns no results or stale results

Reindexing triggers Dataverse to read all dataset metadata from RDS and send it to Solr.
The time it takes depends on how many datasets exist.

### `make test`

Runs the pytest test suite against the target environment:

```bash
make test ENV=tim
```

The tests cover API smoke tests, S3 connectivity, Solr health, and basic CRUD operations.
See Episode 8 for detail on what the tests check.

## The `ENV` argument

Most targets require `ENV=<operator>` to specify which environment to use.
Valid values are `tim` and `jamie`. This controls:

- Which Terraform environment directory is used (`environments/tim` or `environments/jamie`)
- Which Ansible inventory file is used
- Which group_vars overrides are applied

Running `make rebuild` without `ENV` will error. Always specify it.

::::::::::::::::::::::::::::::::::::: challenge

### Trace a rebuild

Without running it, trace through what `make rebuild ENV=tim` does by reading the Makefile.

1. What is the first command it runs?
2. At what point does the old EC2 instance stop existing?
3. At what point does Ansible start running?
4. What is the last thing the target does?

:::::::::::::::::::::::::::::::::: solution

Open `dataverse-infrastructure/Makefile` and find the `rebuild` target.
The steps should follow the sequence described above: terraform destroy, terraform apply,
inventory generation, Ansible run, and test suite. The exact commands will be visible
in the Makefile.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- The Makefile is the daily operations interface -- rarely run Terraform or Ansible directly.
- `make rebuild ENV=<env>` destroys and recreates the EC2 instance and reconfigures it; RDS and S3 are preserved.
- `make baseline ENV=<env>` captures a timestamped snapshot of dataset, file, and S3 counts.
- `make reindex ENV=<env>` rebuilds the Solr index after any database restore.
- Always specify `ENV=` -- the Makefile will error without it.

::::::::::::::::::::::::::::::::::::::::::::::::
