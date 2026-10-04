# Lesson review: adding pointcloud.ucla.edu as a second worked example

2026-10-04. Context: the `pointcloud-infra` repo (Terraform + Ansible + Molecule/Podman staging) was scaffolded today, and Tim wants OSPO students working on it. This reviews the current lesson with that audience in mind.

## What works

- It's honest. "Idempotency (the concept) vs. this role (the reality)" and the gotcha callouts from real sessions are the lesson's best material, and rare in IaC teaching.
- Real incidents (the `gh pr create` fork trap, swap/disk sizing) make the case for the practices better than any toy example could.
- The AI episode and the migration arc are distinctive and worth keeping as they are.

## Problems, most important first

1. **Learners can't do the exercises.** Everything assumes access to private repos and the `ucla-library-dsc` AWS account. The first challenge says "if you don't have access yet, read through the solution instead." For students and anyone outside the two operators, the lesson is read-only.
2. **The only example is the hardest one.** Payara, Solr, RDS and S3 sync is a lot to hold while also learning what Terraform state or an Ansible handler is. There's no small case to learn the tools on first.
3. **Setup contradicts the episodes.** `learners/setup.md` reads like a generic template ("You will need an AWS account you can log into... AWS will be used to create a virtual machine"), while the episodes use a shared institutional profile. It also has a broken code fence (four backticks after the Homebrew block).
4. **Tool versions have drifted.** Episodes say Terraform >= 1.5 and Ansible >= 2.14 and use uv + Make. Current practice (and pointcloud-infra) is Terraform 1.10+ for S3-native locking, ansible-core 2.18+, and a single pinned environment.
5. **The title limits reuse.** "Dataverse Infrastructure" tells a non-Dataverse reader it isn't for them.

## Proposal

Make it a two-case lesson: learn the tools hands-on with pointcloud, which runs entirely on a laptop with no AWS account, then apply them to Dataverse, which is the real-world complexity.

### New episodes (Part 1: pointcloud)

| # | Working title | Hands-on? | Core idea |
|---|---|---|---|
| P1 | A small service: pointcloud.ucla.edu | read + discuss | A hand-built server, the August 2026 outage (full disk, silently expired certs), and what "describe it as code" would have changed |
| P2 | Your first converge: local staging with Molecule and Podman | yes, laptop only | `pixi run staging-up`, see the site, change CSS, re-run; `pixi run test` and what the idempotence step actually checks |
| P3 | Reading a role | yes | defaults vs. group_vars, argument specs, templates, handlers, and the case where a handler is the wrong choice (the content tarball) |
| P4 | Terraform for a server that already exists | plan only | `import` blocks, `prevent_destroy`, the `eip_target` switch for cutover/rollback, reading a plan; contrast with the Dataverse Elastic IP episode |
| P5 | Watching for boring failures | yes | Let's Encrypt stopped sending expiry emails; the site-check script and scheduled GitHub Action; why monitoring lives outside the server |

Optional, likely better in the OSPO education material than here: **Working on upstream software** (building Potree, testing a change in staging, contributing to a backlogged single-maintainer project).

### Changes to existing material

- **Introduction**: present both cases; keep the Dataverse stack diagram for Part 2.
- **Tooling setup / setup.md**: rewrite around pixi + Podman for Part 1 (no AWS needed); move AWS profile setup to the start of Part 2 and mark it operator-only.
- **Ansible idempotency (ep 4)**: link back to P2. The contrast between "pointcloud passes Molecule's idempotence test" and "this role only has module-level idempotency" is a strong teaching moment.
- **Terraform (ep 3)**: note S3-native locking (`use_lockfile`) as the current option vs. the DynamoDB table Dataverse uses.
- **Title**: something like "Infrastructure as Code for Library Services: Terraform and Ansible", with Dataverse and pointcloud as the worked examples.
- Fix the setup.md code fence regardless.

### Open questions for Tim

1. Two-part restructure, or keep the lesson Dataverse-only and write pointcloud as its own short lesson?
2. Who is the primary learner: OSPO students (needs Part 1 first), or DSC operators (current framing)?
3. Is the upstream-contribution episode in scope here or in OSPO education?
