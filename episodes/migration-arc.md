---
title: "The Migration Arc: 5.14 to 6.8"
teaching: 25
exercises: 5
---

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

These changes mean a 5.x database cannot simply be imported into a 6.x instance without
migration steps. The migration process handles database schema updates automatically --
Dataverse runs schema migrations on startup -- but the Solr index must be rebuilt manually.

## The 7-phase plan

The migration is structured in seven phases. Each phase ends with a gate: the test suite
and (where applicable) baseline comparisons must pass before the next phase begins.

### Phase 1: Production Baseline Capture (complete)

Capture a timestamped snapshot of the production 5.14 instance before any migration work begins.
This snapshot is the anchor for the final comparison after cutover.

- Run by Jamie against production with `BASELINE_UPLOAD_BUCKET` set
- Stored in S3 for durability

### Phase 2: Rebuild + Infrastructure Hardening (current, partially complete)

Goal: Tim's dev environment rebuilds cleanly with an Elastic IP (no DNS wait on each
rebuild), FAKE DOI provider, `test_cert: true` enforced, and Solr reindex as an explicit
post-restore gate. Three plans, and they're not all done:

- `02-01` Elastic IP resource in Terraform -- **not started**. This is the item covered
  in Episode 3: an `aws_eip` resource exists, but it's tied to the instance's lifecycle
  and doesn't survive `terraform destroy`, so rebuilds still require a manual DNS update.
- `02-02` FAKE DOI provider config + `test_cert: true` + Solr reindex make target -- **done**.
- `02-03` Full `make rebuild ENV=tim` cycle validation (clean rebuild, reindex, smoke
  tests pass) -- **not started**.

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

This phase has a long lead time -- campus IT needs to register the SP metadata.
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

The production migration itself:

1. Reduce DNS TTL to 60 seconds (done days in advance)
2. Open maintenance window; notify users
3. Run final production DB dump
4. Restore DB to 6.x environment
5. Run `make rebuild ENV=jamie` against 6.x
6. Run Solr reindex
7. Run full test suite; baseline comparison must pass
8. Switch DNS A record to the Elastic IP on the 6.x instance
9. Switch DOI/EZID configuration from FAKE to real
10. Monitor for 24 hours
11. Terminate old 5.14 instance

Rollback is possible until step 9 (EZID switch). If anything fails before that point,
switch DNS back to the old instance and the 5.14 instance is back up within the TTL window.

::::::::::::::::::::::::::::::::::::: callout

### A gap this sequence doesn't cover: the actual file bytes

Steps 3-6 move the *database* -- dump, restore, rebuild, reindex. None of them move the
files themselves from the old instance's storage to the new one's S3 bucket. The
infrastructure security/reliability audit flags this as a High-severity gap (F6): today,
file-bytes migration exists only as manual prose in the ansible repo's migration guide
(an `aws s3 sync` command a human is expected to run and remember), and the integrity
checks in step 7 (baseline comparison) count database rows and S3 object counts -- they
don't verify that a given file is actually downloadable. A cutover that follows only the
11 steps above could produce a production Dataverse where datasets exist, search works,
and downloads 404. A round-trip file-retrievability test is planned (roadmap `03-02`)
but isn't itself a migration step -- something still has to actually move the bytes, and
that's not automated yet. Resolve this before Phase 7, not during it.

::::::::::::::::::::::::::::::::::::::::::::::::

## Rollback decision points

| Point | Rollback action |
|---|---|
| Before DNS switch | Switch DNS back; old instance still running |
| After DNS switch, before EZID | Switch DNS back; new DOIs minted as FAKE can be re-minted |
| After EZID switch | Rollback is complex; requires DOI management coordination |

The goal is to not reach a point where rollback is complex. The test suite gate
after the DB restore (step 7) is the last clean opportunity to stop.

## DNS TTL and the Elastic IP

DNS TTL (Time To Live) controls how long resolvers cache the DNS record.
If the TTL is 3600 seconds (one hour) and you switch the DNS A record,
some users will still be routed to the old IP for up to an hour.

The procedure reduces TTL to 60 seconds several days before cutover.
At that TTL, the propagation delay after the DNS switch is at most 60 seconds.

The plan is for an Elastic IP to give the new 6.x instance a known, stable address before
cutover day, so the DNS change is a single A record update. That depends on `02-01`
landing first, though (see Phase 2 above) -- as of today the EIP doesn't survive a
rebuild, so "known, stable IP before cutover" isn't true yet. If cutover happened this
week, step 8 below would still need the same manual "read the new IP off the Terraform
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
2. Besides the 7 roadmap phases, what other body of work now also has to be resolved before Phase 7 can safely run -- and why wasn't it part of the original roadmap?

:::::::::::::::::::::::::::::::::::: solution

1. No. Only `02-02` (FAKE DOI + `test_cert` + reindex target) is done. The Elastic IP
   (`02-01`) and full rebuild validation (`02-03`) are both still unstarted.
2. The infrastructure security/reliability audit, run after this roadmap was written,
   found real gaps the 7 phases don't cover: an open admin API on an internet-facing
   instance (Critical), unencrypted RDS storage, a stale/manual backup pipeline, and no
   automated file-bytes migration step (the gap covered above). None of these were
   anticipated when the roadmap's phases were scoped -- they surfaced from auditing what
   was actually built, not from the migration plan itself. They now gate Phase 7 alongside
   the original 6 phases.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Dataverse 6.x requires Java 17, a Solr schema rebuild, and updated DOI/S3 configuration.
- The 7-phase plan gates each phase with tests before proceeding to the next -- but "current phase" doesn't mean "current phase complete." Phase 2 is one-third done as of this writing.
- Phases 1-3 establish the foundation; phases 4-6 harden for production; phase 7 is cutover.
- Rollback is straightforward until the DNS switch and EZID activation -- that window is the target.
- DNS TTL reduction and a *persistent* Elastic IP would together make the cutover switchover fast and predictable -- but the EIP isn't persistent yet, and the automated cutover sequence has no step that moves file bytes, a gap found after the roadmap was written.

::::::::::::::::::::::::::::::::::::::::::::::::
