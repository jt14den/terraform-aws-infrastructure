---
title: "Certification track: EX294"
---

This page is optional. It's for learners using this lesson to prepare for Red Hat exam **EX294, the Red Hat Certified Advanced System Administrator in Ansible exam**, which counts toward Red Hat Certified Engineer in Ansible. Nothing in the core episodes depends on it.

Check the exam details before you book: Red Hat updates objectives, and the exam page says objectives are based on the most recent product version. As of October 2026, the matching course (AU294) is based on **RHEL 10, ansible-core 2.16, and Ansible development tools aligned with Ansible Automation Platform 2.6**.

## How the exam works, and what that means for practice

- It's hands-on: you configure real systems with playbooks you write, and they're checked after a reboot.
- No internet. Product documentation is available, so learn to find answers with `ansible-doc`, not a browser.
- Your playbooks run against freshly installed systems, so practice from clean machines, not from something you've tweaked by hand.

So practice the same way: from a requirement, without the finished solution open, using only `ansible-doc`, then rerun and reboot to prove it holds.

## The study loop

Use this for every exercise in the lesson and every drill below:

1. **Specify.** Write down the end state and how you'll check it, before any YAML.
2. **Predict.** Which tasks will report `changed` on the first run? On the second?
3. **Attempt.** Write the smallest piece yourself. Look things up with `ansible-doc <module>`.
4. **Check.** Run it. Compare what happened with what you predicted, and check the system itself, not only Ansible's output.
5. **Explain.** One or two sentences: what surprised you, and why.
6. **Transfer.** A few days later, solve a variation without looking at your earlier solution.

AI tools are useful for review and hints (see the AI episode), but the exam has none. Do at least every other drill without one.

## Tools: the exam environment

[pointcloud-infra](https://github.com/ucla-data-science-center/pointcloud-infra) has a separate environment pinned to the exam's stack, so you practice against the same Ansible version the exam uses:

```bash
cd pointcloud-infra
pixi shell -e exam            # ansible-core 2.16 + ansible-navigator
ansible --version
ansible-doc -l | grep firewalld
ansible-navigator --help
```

The default environment (`pixi run ...`, ansible-core 2.21) is for running the real project. Newer ansible-core versions changed some templating behavior, so do exam drills in the `exam` environment.

`ansible-navigator` normally runs playbooks inside an execution environment (a container image). To practice the command line without one, add `--ee false`:

```bash
ansible-navigator run site.yml -i inventory --ee false -m stdout
```

## Objective map

Objectives are grouped as on Red Hat's EX294 page (checked 2026-10-04). **Where** shows where you can see or practice each one in this lesson or in pointcloud-infra. **Gap** means you need a separate drill.

| Objective | Where | Gap and drill |
|---|---|---|
| **Core components:** inventories, modules, variables, facts, loops, conditionals, plays, task failure, playbooks, config files, roles, documentation | Episode: First converge. pointcloud-infra `ansible/roles/potree/` uses `dnf`, `template`, `copy`, `file`, `systemd_service`, `firewalld`, `uri`, `unarchive`, `stat`, `find`; loops in `tasks/content.yml`; conditionals throughout | Failure handling: the role uses `block` and `failed_when` but not `rescue`/`always` or `ignore_errors`. Drill: a play that tries a package from an unavailable repo, reports clearly, and continues |
| **Configuration:** `ansible.cfg`, `ansible-navigator.yml`, static inventory files, host groups | `pointcloud-infra/ansible/ansible.cfg` | The project uses a dynamic AWS inventory and no `ansible-navigator.yml`. Drill: write a static inventory with two groups and a group of groups, plus an `ansible-navigator.yml` that sets the inventory and mode |
| **Managed nodes:** configure nodes, SSH keys, privilege escalation, deploy files | `become: true` in `playbooks/site.yml` | Drill (exam lab): create an automation user on each node, distribute its SSH key, give it passwordless sudo, then switch your inventory to use it |
| **Running playbooks:** `ansible-navigator` and `ansible-playbook`, find modules in collections, create inventories, configure the environment | `pixi run deploy-check`, `ansible/requirements.yml` | Drill: run the same playbook with both tools; use `ansible-navigator collections` and `ansible-doc` to find a module you haven't used |
| **Source control:** Git clone and add; VS Code to create playbooks and push | The whole lesson workflow (branch, commit, PR) | Drill: set up VS Code with the Ansible extension and write one drill in it |
| **Playbook creation:** common modules, registered results, conditionals, error handling, system configuration | `register` + `when` in `tasks/release.yml` and `tasks/apache.yml` | Covered by the drills above |
| **Roles and collections:** create and use roles, install roles, install collections, related content | The `potree` role; `meta/argument_specs.yml`; `requirements.yml` | Drill: install a role from Ansible Galaxy with `ansible-galaxy role install` and use it |
| **RHCSA tasks:** packages/repos, services, firewall, file systems, storage, file content, archiving, scheduling, security, users/groups | EPEL and packages (`tasks/packages.yml`); services and timers; firewalld; `ini_file`, `copy`, `template`; `unarchive`; SELinux checks (`tasks/selinux.yml`) | Not in the project: storage and file systems (`community.general.lvg`/`lvol`, `filesystem`, `mount`), `lineinfile`, `archive`, `cron`, `user`/`group`, `seboolean`/`sefcontext`. Drills on lab VMs (containers can't do storage or reboots) |
| **Content:** templates, Ansible Vault | `templates/pointcloud.conf.j2` | Vault: the project has no secrets. Drill: encrypt a dummy variable file, use it in a play, show the plaintext never appears in output |

## Practice environments

| What you're practicing | Where |
|---|---|
| Modules, templates, handlers, roles, idempotence | Local staging: `pixi run staging-up` (Rocky Linux 10 container) |
| Storage, file systems, users, SELinux, firewalld, reboot persistence | Virtual machines. A Terraform-built exam lab (one control node, three or four managed Rocky Linux nodes, destroyed after each session) is planned for this lesson |
| Timed, requirement-only practice | Clean VMs, a requirements list, `ansible-doc`, a timer, no AI |

## Outside resources

- [Red Hat's EX294 page](https://www.redhat.com/en/services/training/ex294-red-hat-certified-engineer-rhce-exam-red-hat-enterprise-linux): the authoritative objective list. Read it again the week you book.
- [Lisenet's EX294 sample exam](https://www.lisenet.com/2019/ansible-sample-exam-for-ex294/): free, 18 practical tasks on a control node plus four managed nodes. Written in 2019 for RHEL 8, so use it as a requirements bank, not a guide to current coverage.
- Sander van Vugt's EX294 video course (2nd edition): guided labs with explained solutions. Check which RHEL and tooling versions it covers before buying.
