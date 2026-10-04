---
title: "The Migration Arc: 5.14 to 6.8"
teaching: 25
exercises: 5
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

- What changes between Dataverse 5.14 and 6.8?
- Why is the migration done in phases?
- What happens on cutover day and what does rollback look like?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Explain the 7-phase migration plan and why it is sequenced the way it is.
- Describe what changes between Dataverse 5.14 and 6.8.
- Identify the rollback decision points and what each one requires.
- Explain what the Elastic IP and DNS TTL reduction accomplish at cutover.

::::::::::::::::::::::::::::::::::::::::::::::::

## What changes in 6.x

Dataverse 6.x introduced several significant changes from the 5.x series:

- **Storage subsystem refactor**: S3 configuration moved to a new format; the storage driver ID must be set explicitly
- **DOI/PID provider changes**: The way persistent identifiers are configured changed; the FAKE provider configuration syntax differs
- **Solr schema updates**: The Solr index schema changed; the old index is incompatible and must be rebuilt from scratch
- **Java version requirement**: Dataverse 6.x requires Java 17 (up from Java 11 in 5.x)
- **API changes**: Some API endpoints changed or were deprecated; any integrations need to be verified

These changes mean a 5.x database cannot be imported directly into a 6.x instance without
migration steps. The migration process handles database schema updates automatically
(Dataverse runs schema migrations on startup), but the Solr index must be rebuilt manually.

::::::::::::::::::::::::::::::::::::: callout

### The target version moved after this lesson's title was set

This lesson (and its episode titles) were written targeting 6.8, which is what
`group_vars` is actually pinned to as of this writing. The version bump below hasn't
happened yet. It needs to: CVE-2026-1879 affects 6.0 through 6.8 and is patched starting
at 6.10, so the real target is now 6.10.1 or 6.11 (not the newest release, 6.12 as of
September 2026; letting a release season for a few weeks before adopting it is a
reasonable default for a two-person team). That version bump also carries a Payara
6-to-7 and Java 17-to-21 runtime upgrade, which upstream `gdcc/dataverse-ansible` already
made. Reusing their migration commit as a template beats re-deriving it from scratch.

Treat "6.8" throughout this lesson as "the version this was written against," not
"the current target." Once the group_vars pins actually move, the episode titles and
this callout should be the first things updated.

::::::::::::::::::::::::::::::::::::::::::::::::

## The historical 7-phase plan

The completion labels below reproduce the planning snapshot this case study was based on. They have not been revalidated as current operational status.

The migration is structured in seven phases. Each phase ends with a gate: the test suite
and (where applicable) baseline comparisons must pass before the next phase begins.

![The seven migration phases run in sequence, each gated by test/baseline passage before the next begins. Phase 2 is only one-third complete as of this writing.](fig/migration-phases.svg){alt="Pipeline diagram of the 7 migration phases in order: Phase 1 Production baseline (complete), Phase 2 Rebuild and hardening (1 of 3 done), Phase 3 Expanded test coverage, Phase 4 Jamie onboarding, Phase 5 Shibboleth/SSO, Phase 6 Maintenance window plan, Phase 7 DNS cutover."}

### Phase 1: Production Baseline Capture (complete)

Capture a timestamped snapshot of the production 5.14 instance before any migration work begins.
This snapshot is the anchor for the final comparison after cutover.

- Run by Jamie against production with `BASELINE_UPLOAD_BUCKET` set
- Stored in S3 for durability

### Phase 2: Rebuild + Infrastructure Hardening (partially complete in the snapshot)

Goal: Tim's dev environment rebuilds cleanly with an Elastic IP (no DNS wait on each
rebuild), FAKE DOI provider, `test_cert: true` enforced, and Solr reindex as an explicit
post-restore gate. Three plans, and they're not all done:

- `02-01` Elastic IP resource in Terraform: **not started**. This is the item covered
  in [Terraform](terraform-infrastructure.md): an `aws_eip` resource exists, but it's tied to the instance's lifecycle
  and doesn't survive `terraform destroy`, so rebuilds still require a manual DNS update.
- `02-02` FAKE DOI provider config + `test_cert: true` + Solr reindex make target: **done**.
- `02-03` Full `make rebuild ENV=tim` cycle validation (clean rebuild, reindex, smoke
  tests pass): **not started**.

So "Phase 2 complete" would mean all three; right now it's one of three. Don't treat this
phase as finished just because the rebuild command runs.

### Phase 3: Expanded Test Coverage

Extend the test suite with:

- Baseline count comparison tests
- S3 file upload and download round-trip
- DOI/FAKE validation
- Explicit Solr reindex gate

These tests gate every subsequent phase.

### Phase 4: Jamie's Environment Onboarding

Jamie can independently:

- Destroy and rebuild her environment from scratch
- Run the full test suite
- Do this using only the runbook (not by asking Tim)

This proves the process is operator-independent. Required before production cutover.

### Phase 5: Shibboleth / SSO

Configure Shibboleth authentication via Ansible, register the SP metadata with the UCLA campus IdP,
deploy to Jamie's environment, and validate the full login flow.

This phase has a long lead time: campus IT needs to register the SP metadata.
Coordination starts in Phase 2 to avoid blocking the cutover schedule.

### Phase 6: Maintenance Window Planning

Document the complete cutover procedure before executing it:

- Step-by-step cutover sequence with timing estimates
- Rollback decision points and what each one requires
- DOI/EZID switch procedure
- User communication templates
- DNS TTL reduction checklist

Nothing on cutover day should be improvised. This phase produces the document that
the team follows on cutover day.

### Phase 7: DNS Cutover

The following is an **illustrative sequence for review**, not a runnable
production procedure. The local orchestration Makefile inspected on 2026-10-04
(commit `9e4d642`) starts `rebuild` with an unscoped `terraform destroy`, provisions
resources, runs Ansible, and invokes `scripts/restore-db.sh`. This establishes
that `rebuild` is not an in-place application restart. Actual deletion behavior
also depends on Terraform protections and resource configuration. Never place
this destructive target after a final restore.

1. Rehearse creation and configuration of the destination **before final restoration**.
   Verify target versions, database upgrade order, storage identifiers, ownership,
   deletion protections, backup independence, and the exact scope of every target.
2. Prepare DNS/routing and lower TTL ahead of the maintenance window. Notify users
   and record who can authorize cutover or rollback.
3. Freeze writes on the source, including uploads, APIs, background jobs, and
   administrative changes. Keep the destination closed to user writes as well.
4. Capture the final consistent database dump/snapshot and a baseline at that
   boundary. Record timestamps, checksums, versions, and restore identifiers.
   Keep the recovery copy outside any resources slated for destruction.
5. Synchronize file/object bytes from the source storage to the destination,
   including the final delta after the freeze. Verify the mapping from database
   storage identifiers to objects; an earlier bulk copy alone is insufficient.
6. Restore that exact final database to the prepared destination. Apply the
   documented version-specific schema upgrade sequence, then rebuild Solr.
   Verify the restore helper selects the intended final dump rather than an older
   object labeled "latest". Do not run `make rebuild` afterwards.
7. Check counts, metadata relationships, object sizes/checksums, and representative
   downloads against known source bytes. Check authentication, permissions, search,
   and application behavior. Matching counts alone do not prove data integrity.
8. Switch the reviewed DNS or routing target while writes remain frozen. Verify
   TLS and access through the public name; retain the old system and backups.
9. Configure the approved production identifier service and reopen writes only
   after all gates pass. Record this boundary and monitor new writes and failures.
10. Retire the old system only after the agreed recovery/retention period and
    explicit approval. A fixed 24-hour timer is not proof that recovery is safe.

Maintainers must resolve file synchronization, freeze enforcement, restore
selection, upgrade compatibility, admin API access, and post-cutover data
reconciliation in the real runbook. The lesson does not validate these details.

::::::::::::::::::::::::::::::::::::: callout

### Cutover pressure and the pull toward hand-fixing

The validation gate is the moment automation trust gets tested hardest: something in the test suite or
baseline comparison fails, users are waiting, and SSHing into the new instance to patch
whatever's wrong feels like the responsible thing to do. It's the same instinct
[Ansible idempotency](ansible-idempotency.md) (Ansible: Configuration and Idempotency) warns about, just under more pressure.

Resist it here more than anywhere else. A hand-patch made during cutover is exactly the
kind of undocumented change that breaks the "read the repo, know the server" property this
stack depends on, and it happens at the one moment when there's the least time to notice
it happened. The rollback table below and the phase gates before it exist so the answer to
"something broke during cutover" is "roll back, or re-run the documented sequence," not
"fix it live and write it down later."

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: callout

### Database and file bytes are one recovery boundary

The historical rebuild target restores a database, but that does not establish
that uploaded files were copied or restored. The sequence above makes the final
file/object synchronization explicit. A round-trip upload test exercises a new
file; it does not prove historical files survived. Compare stored identifiers,
checksums and retrievability with the frozen source.

::::::::::::::::::::::::::::::::::::::::::::::::

## Rollback decision points

| Point | What rollback requires |
|---|---|
| Before routing changes, writes frozen | Keep the destination closed; verify the source is authoritative before reopening it |
| After routing changes, both systems still frozen | Route back, verify clients reach the old service, then reopen writes under the runbook |
| After new writes on the destination | Freeze again and reconcile or replay new metadata and file writes before reverting; DNS alone risks data loss |
| After external identifier registration | Also coordinate persistent identifier records and their targets; do not assume they can be undone |

Accepting new writes changes rollback even if no DOI has been registered.
Do not allow both systems to accept independent writes during routing changes.

## DNS TTL and the Elastic IP

DNS TTL (Time To Live) controls how long resolvers cache the DNS record.
If the TTL is 3600 seconds (one hour) and you switch the DNS A record,
some users will still be routed to the old IP for up to an hour.

The procedure reduces TTL to 60 seconds several days before cutover.
A lower TTL reduces expected cache duration, but does not guarantee every client switches within 60 seconds. Test routing and allow for resolver and client caching.

The plan is for an Elastic IP to give the new 6.x instance a known, stable address before
cutover day, so the DNS change is a single A record update. That depends on `02-01`
landing first, though (see Phase 2 above). As of today the EIP doesn't survive a
rebuild, so "known, stable IP before cutover" isn't true yet. If cutover happened this
week, the routing step would still need the same manual "read the new IP off the Terraform
output" step every other rebuild requires.

::::::::::::::::::::::::::::::::::::: challenge

### Phase check

For each phase below, identify what must be true before that phase can start:

1. Phase 3 (Expanded Test Coverage)
2. Phase 4 (Jamie's Environment Onboarding)
3. Phase 7 (DNS Cutover)

:::::::::::::::::::::::::::::::::: solution

1. Phase 3 requires Phase 2 complete: Tim's env must rebuild cleanly with Elastic IP, FAKE DOI, and test_cert enforced.
2. Phase 4 requires Phase 3 complete: the expanded test suite must pass on Tim's env before onboarding Jamie.
3. Phase 7 requires Phases 1-6 complete: production baseline captured, Jamie can rebuild independently, Shibboleth validated, and the cutover runbook is written and reviewed.

:::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Beyond the 7 phases

1. Is Phase 2 done? Name the one item of three that's actually complete.
2. Besides the 7 roadmap phases, what other body of work now also has to be resolved before Phase 7 can safely run, and why wasn't it part of the original roadmap?

:::::::::::::::::::::::::::::::::::: solution

1. No. Only `02-02` (FAKE DOI + `test_cert` + reindex target) is done. The Elastic IP
   (`02-01`) and full rebuild validation (`02-03`) are both still unstarted.
2. The infrastructure security/reliability audit, run after this roadmap was written,
   found real gaps the 7 phases don't cover: an open admin API on an internet-facing
   instance (Critical), unencrypted RDS storage, a stale/manual backup pipeline, and no
   automated file-bytes migration step (the gap covered above). None of these were
   anticipated when the roadmap's phases were scoped: they surfaced from auditing what
   was actually built, not from the migration plan itself. They now gate Phase 7 alongside
   the original 6 phases.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Dataverse 6.x requires Java 17, a Solr schema rebuild, and updated DOI/S3 configuration.
- The 7-phase plan gates each phase with tests before proceeding to the next, but "current phase" doesn't mean "current phase complete." Phase 2 is one-third done as of this writing.
- Phases 1-3 establish the foundation; phases 4-6 harden for production; phase 7 is cutover.
- Keep both sides write-frozen through validation and routing changes. New writes require reconciliation on rollback, independently of DOI registration.
- DNS TTL reduction and a *persistent* Elastic IP would together make the cutover switchover fast and predictable, but the EIP isn't persistent yet, and file-byte synchronization must be implemented and rehearsed before the illustrative cutover sequence becomes a runbook.

::::::::::::::::::::::::::::::::::::::::::::::::
