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

### Phase 2: Rebuild + Infrastructure Hardening (current)

Tim's dev environment rebuilds cleanly with:

- Elastic IP in Terraform (stable hostname across rebuilds)
- FAKE DOI provider configured in Ansible
- `test_cert: true` enforced
- Solr reindex as an explicit post-restore gate

Goal: a clean, repeatable rebuild cycle before touching the actual migration.

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

The Elastic IP means the new 6.x instance has a known, stable IP address before cutover day.
The DNS change is a single A record update from the old production IP to the Elastic IP.

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

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Dataverse 6.x requires Java 17, a Solr schema rebuild, and updated DOI/S3 configuration.
- The 7-phase plan gates each phase with tests before proceeding to the next.
- Phases 1-3 establish the foundation; phases 4-6 harden for production; phase 7 is cutover.
- Rollback is straightforward until the DNS switch and EZID activation -- that window is the target.
- DNS TTL reduction and the Elastic IP together make the cutover switchover fast and predictable.

::::::::::::::::::::::::::::::::::::::::::::::::
