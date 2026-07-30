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

Destroys and recreates **the entire environment** -- EC2, RDS, and S3 together -- then
configures it and restores data. It requires `DB_PASS` and prints a real warning before
it runs, because of exactly the misconception this section used to encode: this is not
an EC2-only operation.

The real 7 steps, from the Makefile itself:

1. **Destroy** -- `terraform destroy` tears down the whole per-environment module: EC2,
   RDS, and the S3 bucket all go together. Nothing about this step is EC2-scoped.
2. **Provision** -- `terraform init -upgrade` then `terraform apply` builds all of it
   back from scratch: new EC2 instance, new empty RDS database, new empty S3 bucket.
3. **DNS propagation** -- prints the new EC2 IP and pauses for you to manually update
   the DNS A record (this is the Elastic IP gap from Episode 3 in practice -- the IP
   really did change, and nothing updates DNS automatically yet).
4. **Ansible** -- runs `site.yml` to install and configure Payara, Solr, Apache, and Dataverse.
5. **Restore the database from S3** -- `scripts/restore-db.sh` pulls the latest dump and
   loads it into the just-created, currently-empty RDS instance. This step exists
   *because* step 1 wiped the database -- without it, rebuild would hand you an empty Dataverse.
6. **Start Payara** -- over SSH (`ssh rocky@$IP 'sudo systemctl start payara'`).
7. **Wait, then reindex** -- polls the app until it responds, then runs `make reindex`
   to rebuild Solr from the just-restored database.

There is no test-suite step at the end -- rebuild ends at reindex and tells you to
`make logs` to monitor. Running `make test` afterward is a separate, manual step.

Use `make rebuild` when:

- Starting fresh after a failed or degraded environment
- Testing infrastructure changes that require a clean stack
- Validating a new Ansible configuration from scratch

```bash
make rebuild ENV=tim DB_PASS=<dataverse_postgresql_password>
```

::::::::::::::::::::::::::::::::::::: callout

### Nothing survives a rebuild by default -- the restore step is what saves you

The single most consequential fact about `make rebuild`: it destroys RDS and S3 along
with EC2, and the reason the environment isn't empty afterward is step 5, restoring from
a database dump in S3 (`ucla-dataverse-migration-assets`) -- a *separate* bucket from the
one `terraform destroy` just deleted. The security/reliability audit of this repo calls
this restore-every-rebuild pattern "the single most valuable reliability practice here" --
it means every rebuild is implicitly a disaster-recovery drill, proving the backup
actually works. But it also means a stale or missing dump turns rebuild into "spin up an
empty Dataverse," not "restore my environment." There's no confirmation step that checks
dump freshness before restoring (a known gap, audit F4).

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

The snapshot is saved to `baseline-snapshots/baseline_<timestamp>.json` and optionally
uploaded to S3 if `BASELINE_UPLOAD_BUCKET` is set:

```bash
BASELINE_UPLOAD_BUCKET=ucla-dataverse-migration-assets make baseline ENV=jamie
```

Jamie's production baseline (run before migration begins) is the anchor for the
Phase 7 post-cutover comparison. Upload it to S3 so it is durable and shared.

### `make baseline-compare`

Compares two baseline snapshots and reports differences:

```bash
make baseline-compare BEFORE=baseline-snapshots/before.json AFTER=baseline-snapshots/after.json
```

A passing comparison shows matching counts across all fields -- with one deliberate
exception: `downloads`/guestbook-history drift is treated as informational only, not a
failure, since download counts can legitimately keep changing between the two snapshots.
Any discrepancy in dataset or file counts, though, is a problem to investigate before
declaring the migration complete.

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

Under the hood this is a `curl -X DELETE` to Dataverse's admin API over public HTTPS --
which only works today because that API is currently open to the internet, a Critical
security finding covered in depth in Episode 5. `make baseline` and `make test` share the
same dependency.

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

### Trace a rebuild -- for real this time

Open `dataverse-infrastructure/Makefile` and find the `rebuild` target. Without relying
on memory of this episode, answer from the actual Makefile:

1. What happens to the RDS database during step 1? What restores it, and from where?
2. Does the target end with a test run? What does it actually end with?
3. What manual action does step 3 require from the operator, and what does that tell you about the Elastic IP's current state (Episode 3)?

:::::::::::::::::::::::::::::::::::: solution

1. `terraform destroy` in step 1 deletes the RDS instance along with EC2 and S3 -- the
   whole module goes together. Step 5, `scripts/restore-db.sh`, restores it from a dump
   pulled from the `ucla-dataverse-migration-assets` S3 bucket (a separate bucket from
   the one that just got destroyed).
2. No. It ends with step 7 (wait for the app, then `make reindex`) and a message pointing
   you to `make logs` to monitor. `make test` is a separate command you run yourself afterward.
3. Step 3 pauses and prints the new EC2 IP, asking you to update the DNS A record by hand
   before continuing. That's only necessary because the Elastic IP doesn't yet survive a
   rebuild (Episode 3) -- if it did, this manual step wouldn't exist.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- The Makefile is the daily operations interface -- rarely run Terraform or Ansible directly.
- `make rebuild ENV=<env> DB_PASS=<pass>` destroys and recreates EC2, RDS, **and** S3 together, then restores the database from an S3 dump. It does not preserve data by default -- the restore step is what puts data back.
- `make baseline ENV=<env>` captures a timestamped snapshot to `baseline-snapshots/`, with dataset, file, and S3 counts.
- `make reindex ENV=<env>` rebuilds the Solr index after any database restore -- and depends on the admin API being open over public HTTPS (a known security gap, Episode 5).
- Always specify `ENV=` -- the Makefile will error without it.

::::::::::::::::::::::::::::::::::::::::::::::::
