# Restructure plan: two-part lesson + EX294 practice

2026-10-04. Follows `review-2026-10-04-pointcloud.md`. Inputs: Tim's answers, two ChatGPT consults on learning design and EX294 prep (adjudicated below), and Red Hat's EX294 objectives page (checked 2026-10-04).

## Decisions from Tim

- Restructure: yes, where it makes sense. Tim wants to learn this himself, not just document it.
- Learners: Tim, OSPO students, Jamie, Doug, Leigh.
- Goal for Tim: get RHCE certified (EX294) using our real infrastructure as practice.
- pointcloud-infra is public.

## ChatGPT advice: what we're adopting

| Claim | Verdict | Where it lands |
|---|---|---|
| Learner should write the automation, not run finished code | ADOPT | Exercises go worked example → partly finished example → requirement only. The finished pointcloud role is the worked example; exercises start from stripped-down branches/tags |
| Specify, predict, attempt, check, explain, transfer cycle | ADOPT | Recurring structure in every Part 1 episode |
| "Second run shows no changes" and "the app works" are separate checks | ADOPT | Already what Molecule does (idempotence step + verify step); make it explicit in P2 |
| Potree as the contrast to Dataverse's rerun restriction | ADOPT | P2 links to ep 4 |
| Optional EX294 mapping page | ADOPT | `learners/ex294-map.md` |
| Use a small prepared point cloud, keep conversion out of early exercises | ADOPT | Potree's release zip ships sample point clouds; use one offline |
| AI episode: attempt first, ask for a hint, rebuild later | ADOPT | Small edit to `using-ai.md` |
| Current EX294 objectives include ansible-navigator, Git, VS Code, development containers | VERIFIED | Red Hat page lists all four, plus Vault, static inventory and RHCSA tasks (firewall, storage, file systems, users, scheduling, archiving) |
| Lisenet free sample exam first, Sander van Vugt course for guided prep | ADOPT as external resources | Lisenet targets RHEL 8, so treat it as a skills check, not full coverage |
| Three episodes on a "disposable host" | MODIFY | Fine for the Ansible basics; RHCSA tasks need real VMs (below) |

## What both consults missed

**The exam runs on RHEL. pointcloud runs Ubuntu.** An Ubuntu/apt/Apache role never touches `dnf`, `httpd`, `firewalld`, SELinux file contexts, or the Red Hat file layout, and those are a big part of "RHCSA task automation". Dataverse already runs Rocky Linux 9.

**Containers can't practice the whole exam.** Molecule + Podman covers packages, services, templates, files, handlers and roles. It can't credibly cover firewalld, SELinux, storage/LVM, file systems, or "persists after reboot". Those need VMs.

## Proposed shape

**Part 1: Learn the tools on pointcloud (hands-on, no AWS needed for most)**

| # | Episode | You build | EX294 objectives touched |
|---|---|---|---|
| P1 | From hand-built server to desired state | Write down what the server needs (packages, files, services, cert) as requirements before any YAML | plays, documentation lookup |
| P2 | Your first converge | Fill in a partly finished playbook; predict first and second runs; run Molecule; separate "no changes" from "site works" | modules, playbooks, facts, config files |
| P3 | Make it reusable | Variables, templates, handlers, then refactor into a role; deploy a second config without copying | variables, templates, handlers, roles, loops, conditionals |
| P4 | Two operating systems | Port the role to Rocky Linux: dnf, httpd, firewalld, SELinux contexts on the web root, OS-specific vars | packages/repos, services, firewall, security, conditionals |
| P5 | Break it, fix it, rebuild it | Instructor-planted faults; diagnose from evidence, fix in code, rebuild from scratch | handling task failure, troubleshooting |
| P6 | Terraform for a server that already exists | `import`, `prevent_destroy`, the Elastic IP switch, reading a plan | (Terraform; not EX294) |
| P7 | Watching for boring failures | cert expiry check, scheduled jobs, monitoring outside the box | task scheduling |

**Part 2: Dataverse** (the existing episodes, light edits): introduction, tooling, Terraform, Ansible idempotency (now links back to P2), the Dataverse stack, secrets (Vault), Makefile, testing, migration arc. Then the AI episode closes the lesson.

**Learner extras**
- `learners/ex294-map.md`: every exercise mapped to EX294 objectives, plus the gaps and where to practice them.
- `learners/exam-lab.md`: a Terraform-built practice lab of one control node and three to four managed Rocky VMs, destroyed after each session. Use it for storage, LVM, users, firewalld, SELinux, reboot persistence, `ansible-navigator`, and the Lisenet tasks.

**Upstream contribution:** belongs in OSPO education material, not here. This lesson links to `pointcloud-infra/docs/hacking-on-potree.md`.

## Open decision (blocks drafting P3 to P6)

What OS does production run after the rebuild? See the conversation on 2026-10-04.

## Revision after external review (2026-10-04, later)

A second external review (adjudicated in `pointcloud-infra/docs/research/2026-10-04-validation-adjudication-potree-ecosystem.md`) changed the plan. Production OS is decided: **Rocky Linux 10** (Red Hat shop; AU294 is RHEL 10 based). The Ubuntu version of the role is tag `ubuntu-baseline` in pointcloud-infra.

**Smaller shared core (6 episodes), each with an observable outcome:**

| # | Episode | Learner can... |
|---|---|---|
| 1 | Follow one point cloud request | trace browser to Apache to S3 and say where a failure happens |
| 2 | First converge and second run | explain what changed, and check the site separately from "0 changed" |
| 3 | Variables, templates and handlers | change one requirement and predict the reload |
| 4 | Organize a role, validate inputs | add a collection without copying infrastructure logic |
| 5 | Break, diagnose, recover | fix a planted fault from evidence and confirm recovery |
| 6 | Operate the service | read monitoring, release, and roll back |

**Extensions (separate episodes, outside the main flow):**
- Transfer: port the role from `ubuntu-baseline` to Rocky (after ep 5, once the model is stable).
- Cloud: Terraform import, `prevent_destroy`, the Elastic IP cutover (needs AWS; offline `plan` reading is labeled as analysis).
- Certification track: EX294 objective map, exam lab (Rocky VMs via Terraform), `pixi run -e exam` (ansible-core 2.16 + ansible-navigator, matching AU294).

**Other changes**
- Use the current exam name: "Red Hat Certified Advanced System Administrator in Ansible" (EX294); it counts toward Red Hat Certified Engineer in Ansible.
- Teach the difference between controller tooling (ansible-navigator, execution environments, VS Code dev containers) and managed targets (Molecule containers, lab VMs).
- Part 2 (Dataverse) must work without private access: sanitized fixtures and sample outputs, or labeled clearly as a case study.
- Completion differs by audience: students (publish, verify, diagnose a collection), staff operators (deploy, monitor, restore), certification track (core plus requirement-driven practice without the finished role).
- Pilot the core with someone who didn't build the repo.
