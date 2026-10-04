# Lesson revision: evidence and validation

2026-10-04. Implemented the later six-episode core decision in the restructuring
plan, confirmed by the user after a scope check. The user clarified that passing
EX294 and testing knowledge independently of AI assistance are the priorities.
The core now leads to requirement-driven retries; Dataverse is optional for that goal.

## Sources inspected

- Local pointcloud-infra checkout, commit `71ed8620ba12c55918abb706e3a8c8b9035aaaa9`:
  `pixi.toml`, role defaults, argument specs, tasks, templates, handlers,
  Molecule scenario/verifier, content descriptions, packer, and `scripts/site-check.sh`.
  No companion files were changed. New lesson paths named Workshop are explicitly
  learner-created HTML, not existing fixtures or a claimed point-cloud dataset.
- Local Dataverse orchestration Makefile, commit
  `9e4d642b4c365f6ff4e6600cfc6d33a26ad04716`: rebuild starts with an unscoped
  destroy, then provisions, configures, restores a database, starts Payara and
  reindexes. No target was executed. This does not establish live resource state,
  effective deletion protection, current backup contents, or migration readiness.
- Installed ansible-core 2.21.4 `ansible-doc` for command and task keywords,
  and strategy source for recap accounting. Versioned online Ansible documentation
  URLs were unavailable through the browsing tool; installed matching documentation
  and executable dummy tests supplied the behavior checks instead.
- [Official EX294 objectives](https://www.redhat.com/en/services/training/ex294-red-hat-certified-engineer-rhce-exam-red-hat-enterprise-linux),
  checked 2026-10-04. Removed automatic equivalence between a companion environment
  pin and all exam versions. Did not add a new certification lab or claim complete coverage.

## Validation

- Existing R environment: sandpaper 0.17.3. Used `sandpaper::validate_lesson()`
  and `sandpaper::build_lesson(preview = FALSE)`, not CI deployment.
- Workbench fenced-div and internal-link/image validation passed. Local site build
  passed. Warnings: locale setting, unavailable CRAN package index during the first
  build, and Pandoc's deprecated `--mathml` flag. No new toolchain was installed.
- YAML/front matter, all 16 episode navigation targets, six-core ordering,
  required episode blocks, and rendered challenge/solution markup checked.
- `git diff --check` passed. Searched for stale episode numbers, invalid Vault
  view/edit examples, recap/guard confusion, and overclaims about count integrity.
- Executed dummy localhost Ansible checks under a fresh temporary directory:
  two hidden-change append runs grew the file; declarative copy repaired it;
  a second copy run reported no change; whole-file Vault encryption/view and an
  inline-vault assertion succeeded. Only invented lab secrets were used.
  The initial sandboxed run could not start Ansible's local RPC server; the same
  temporary-only checks succeeded with the reviewed sandbox escalation.
- Full Podman/Molecule labs and optional certification container drills were
  statically reviewed against source, not executed. The site build does not run them.
  No Terraform, AWS, deployment, migration, production Ansible, or live-service
  checks were run. Nothing was committed, pushed, or deployed.
- Existing untracked `dev/notes/HANDOFF.md` was left untouched.

## Remaining dependencies and limits

- No verified companion offline point-cloud fixture/workflow exists in the inspected
  checkout. Local labs publish HTML; live request tracing remains a service dependency.
- A learner pilot must establish actual setup/timing and full staging execution on
  supported machines. The nominal core is 235 minutes plus transitions and breaks.
- EX294 preparation still needs independent practice and the listed VM/SSH,
  privilege escalation, storage, SELinux/firewall, and reboot-persistence coverage.
- Historical Dataverse status/version/audit descriptions are case-study context,
  not revalidated production facts. Maintainers must verify target/resource scope,
  final dump selection, write freeze, file/object synchronization, upgrade sequence,
  integrity, routing, and reconciliation before using a real migration runbook.
