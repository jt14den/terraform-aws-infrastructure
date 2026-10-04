---
title: 'Instructor Notes'
---

## Prepare the public core

Use [Setup](../learners/setup.md) before the workshop. Allocate 30–60 minutes for
downloads separately from teaching. Test on a learner-like machine, record the
companion commit and `pixi run ansible --version`, and confirm Podman, ports 8080
and 8443, `staging-up`, and `staging-verify` work. The lesson was statically aligned
with pointcloud-infra commit `71ed862` on 2026-10-04. Recheck if the companion moves.

No AWS credentials or private repository access are needed. Ensure learners have
an editor, terminal, Git, curl, Pixi, and Podman. Windows learners use WSL according
to the companion guide. Start from a learner branch and record existing edits.
Never use a blanket reset to clear a learner's prior work.

There is no verified offline point-cloud fixture workflow. Request tracing depends
on the public site; record actual outputs and dates if demonstrating live. If it
is unavailable, use the episode's recorded examples for diagnosis and label the
activity as analysis. The other core labs publish HTML locally and do not prove
3D rendering. Do not substitute a production deployment for a failed local lab.

## Facilitation and timing

The core totals 240 minutes, plus breaks. Installation and delayed retrieval
practice are outside that allocation. Allow extra time for the first pilot;
container downloads and machine performance can dominate the first converge.

| Episode | Minutes | Expected evidence | Check for understanding |
|---|---:|---|---|
| Request tracing | 35 | Page/code status, metadata response with supplied Referer | Why does 200 not prove rendering? Can a client forge Referer? |
| First converge | 50 | Recap, growing MOTD despite `changed_when: false`, repaired exact file | Are changed tasks included in recap `ok`? What prevents execution? |
| Variables and handlers | 35 | Description change, config check and reload, no second notification | Why does Apache reload for config but not plain HTML? |
| Roles and validation | 40 | New landing page and rejected TLS typo | What does argument validation leave untested? |
| Diagnose and recover | 35 | 200 with wrong heading, repaired source, fresh-container check | Why did the stock verifier miss the custom requirement? |
| Operate | 40 | Expiry warning interpretation, changed release, restored heading | What data would a static-content rollback not restore? |

The listed episodes sum to 235 minutes; reserve five minutes for transitions. For EX294 preparation, use the certification drills after the core and let learners skip Dataverse. Prioritize independent, requirement-driven repetitions over additional service-specific reading.
Ask learners to predict before revealing output or solutions. Pair on explanations,
not just command entry. Collect a short evidence log: prediction, command, observed
state, explanation, and assistance used (unaided, hint, or solution). Give one
hint at a time after an attempt; ask the learner to explain their repair before
showing a solution. Revisit the concept later with a small variation. Completing a reading exercise is not an executed lab.

## Common failures

- Podman connection errors: check the platform's machine/service startup procedure.
- Port conflicts: identify the local process using 8080/8443 before changing anything.
- TLS warnings: use the documented self-signed exception only on localhost; do not
  teach `curl -k` as a fix for public certificate failures.
- Missing Ansible commands: stay at the companion root and use `pixi run`; Molecule
  direct commands run inside the shown `(cd ansible && ...)` subshell.
- YAML parsing errors: inspect indentation and duplicate mappings. Add entries to
  the existing `folders` map, not a second copy of it.
- Nonzero second converge: identify the specific changed task and inspect its state.
  Never "fix" this by hiding change reports.
- A verifier fails after an intentional description change: restore that lab edit
  before running the baseline verifier; explain its hardcoded expectations.
- The invalid TLS input fails before configuration: this is the intended observation.
- Staging does not load S3 scans: this is a documented lab limitation, not evidence
  that learner HTML publication failed.

## Reset and continuity

After First converge, remove only the tasks learners added to `maintenance.yml`.
After the description lab, restore only that description. Keep the new Workshop
folder and its description through the next three labs. At the end, remove only
those learner-created entries, or retain them on a learner branch. Review diffs
before cleanup. `pixi run staging-down` destroys the local lab container;
`pixi run staging-up` recreates it from the current source.

Do not run cloud, deploy, migration, or production Ansible targets. The Dataverse
extension is a separate discussion using historical excerpts, not an operator
qualification or runnable migration procedure. Count comparisons must be paired
with integrity evidence; writes and external effects determine rollback limits.

## Pilot and completion

Pilot with someone who did not build the companion repository. Record setup time,
unclear directory transitions, predictions, actual errors, and whether recovery
works without instructor intervention. Student completion means independently
publishing, verifying, diagnosing, and restoring local content. Staff need separate
approved operational practice. Certification learners need requirement-driven
practice, SSH, real-host security, and reboot checks beyond this container core.
