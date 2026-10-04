---
title: "First Converge and Second Run"
teaching: 20
exercises: 30
---

:::::::::::::::::::::::::::::::::::::: questions

- What does it mean to run a playbook against a server, and then run it again?
- What should the second run report, and why?
- How is "the playbook made no changes" different from "the site works"?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Build a local copy of pointcloud.ucla.edu with Molecule and Podman, and predict which tasks report `changed` on a first and a second run.
- Write a task that breaks idempotence, observe the failure, and fix it.
- Write tasks from a written requirement with no starter code, and check the result on the host.
- Distinguish Molecule's idempotence step from its verify step.

::::::::::::::::::::::::::::::::::::::::::::::::

## Before you start

Commands below start at the companion repository root. Subshells return there automatically. Module behavior was checked against the installed ansible-core 2.21.4 documentation (`pixi run ansible-doc`); use your installed version when looking up options.

You need the pointcloud-infra repository set up on your laptop. If you completed Setup, use that checkout; do not clone it again. Follow its [getting started guide](https://github.com/ucla-data-science-center/pointcloud-infra/blob/main/docs/getting-started.md) (about 30 minutes the first time, mostly downloads). In short:

```bash
git clone https://github.com/ucla-data-science-center/pointcloud-infra.git
cd pointcloud-infra
pixi run setup
```

No AWS account or server access is needed. Everything in this episode runs in a container on your machine.

## Staging: the same playbook, on your laptop

pointcloud-infra has one playbook, `ansible/playbooks/site.yml`, and one role, `ansible/roles/potree/`. Production and your laptop run the same code. On your laptop, [Molecule](https://ansible.readthedocs.io/projects/molecule/) creates a Rocky Linux 10 container with Podman and points the playbook at it. We call that container **staging**.

Running a playbook against a host to bring it to the described state is called a **converge**.

```bash
pixi run staging-up
```

The first run takes a few minutes. Ansible prints each task, and at the end a recap:

```output
PLAY RECAP *********************************************************************
pointcloud-staging         : ok=35   changed=27   unreachable=0    failed=0  ...
```

Your exact numbers may differ. Read them as:

- **ok** in PLAY RECAP: 35 tasks succeeded, including the 27 that reported changes. It does not mean 35 tasks did nothing.
- **changed**: 27 of those successful tasks reported changes. This is a reported status, not independent proof of the resulting state.
- **failed**: something went wrong. Ansible stops for that host.

Open <https://localhost:8443/> and click through the browser's certificate warning (staging uses a self-signed certificate). You should see the same UCLA Library listing as the live site.

::::::::::::::::::::::::::::::::::::: callout

### What staging doesn't do

The public S3-backed point clouds do not normally load in staging: its browser requests do not carry the allowed `www.pointcloud.ucla.edu` Referer. This is a hotlinking deterrent, not authentication; clients can supply that header (see the previous episode). Staging also skips the parts a container can't do (firewalld, SELinux, rebooting, the AWS agent). The repository lists these differences in a real-server [acceptance checklist](https://github.com/ucla-data-science-center/pointcloud-infra/blob/main/docs/acceptance.md).

::::::::::::::::::::::::::::::::::::::::::::::::

## The second run

::::::::::::::::::::::::::::::::::::: challenge

### Predict, then run

Before running anything, write down how many tasks you expect to report `changed` if you run `pixi run staging-up` again, right now, with no edits. Then run it.

:::::::::::::::::::::::::::::::::: solution

You should see `changed=0`. The role aims to leave packages, files, and services in their declared state. Some commands still execute to pack or verify content without reporting a change. This property is called **idempotence**: running the playbook once or ten times leaves the host in the same state.

If you see a non-zero `changed`, look at which tasks reported it. That's a real finding worth reporting as an issue.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

Compare this with the Dataverse role in the [Ansible episode](ansible-idempotency.md): its individual modules are idempotent, but the playbook as a whole isn't safe to rerun. The pointcloud role is held to the stronger standard, and the tests check it on every change.

## Breaking idempotence on purpose

Here's a reasonable-looking request: *"When an operator logs in, show a message saying the server is managed by Ansible."* The text goes in `/etc/motd`.

::::::::::::::::::::::::::::::::::::: challenge

### Attempt 1: the shell way

Make a branch. In `ansible/roles/potree/tasks/maintenance.yml`, add a task named "Tell operators this host is managed" that uses `ansible.builtin.shell` to append the line `Managed by Ansible. Changes made by hand will be overwritten.` to `/etc/motd`. Write it yourself; `pixi run ansible-doc ansible.builtin.shell` shows the options.

Before you run it, write down what the recap will say on the next two runs. Then run `pixi run staging-up` twice and look at the file:

```bash
podman exec pointcloud-staging cat /etc/motd
```

Finally, write one or two sentences: was your prediction right, and what does the file tell you that the recap didn't?

:::::::::::::::::::::::::::::::::: solution

A natural way to write it:

```yaml
- name: Tell operators this host is managed
  ansible.builtin.shell: echo "Managed by Ansible. Changes made by hand will be overwritten." >> /etc/motd
```

The append task reports a change on both runs (one additional changed task in the recap), and the file grows by one line each run:

```output
Managed by Ansible. Changes made by hand will be overwritten.
Managed by Ansible. Changes made by hand will be overwritten.
```

Without execution guards or custom reporting, `shell` and `command` run on each normal playbook run and report `changed`. Here `>>` appends, so the task doesn't describe a state at all; it describes an action.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

Molecule has a step that catches exactly this. On a host that's already converged, it runs the playbook once more and fails if any task reports `changed`:

```bash
(cd ansible && pixi run molecule idempotence)
```

```output
CRITICAL Idempotence test failed because of the following tasks:
*  => potree : Tell operators this host is managed
```

::::::::::::::::::::::::::::::::::::: challenge

### Hide the change, then inspect it

Keep the append command and add `changed_when: false` at the same indentation
as `ansible.builtin.shell`. Predict the task status and the number of new lines
after two runs. Record `podman exec pointcloud-staging wc -l /etc/motd`, run
`pixi run staging-up` twice, then inspect the count and file contents again.
Run `(cd ansible && pixi run molecule idempotence)`. Is passing enough evidence?

:::::::::::::::::::::::::::::::::: solution

```yaml
- name: Tell operators this host is managed
  ansible.builtin.shell: echo "Managed by Ansible. Changes made by hand will be overwritten." >> /etc/motd
  changed_when: false
```

The task reports `ok` but appends a line each time, including during Molecule's
idempotence run. The file grows even if that check passes. `changed_when` changes
reporting and handler notification, not execution. Inspecting the desired state
is stronger evidence than a recap that says `changed=0`. Remove this reporting
override when replacing the command with `copy` below.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Attempt 2: describe the state

Rewrite the task so it describes the end state of `/etc/motd` instead of an action. Use `pixi run ansible-doc ansible.builtin.copy` to find the option that sets a file's contents directly from a string. Set the owner, group and mode too.

Run `pixi run staging-up`, check `/etc/motd`, then run `(cd ansible && pixi run molecule idempotence)` again.

:::::::::::::::::::::::::::::::::: solution

```yaml
- name: Tell operators this host is managed
  ansible.builtin.copy:
    dest: /etc/motd
    content: "Managed by Ansible. Changes made by hand will be overwritten.\n"
    owner: root
    group: root
    mode: "0644"
```

The first run reports `changed=1`, and the duplicated lines from attempt 1 are gone: `copy` makes the file contain exactly this text, whatever was there before. The idempotence step then passes:

```output
INFO     default ➜ idempotence: Executed: Successful
```

Prefer a module that describes the desired state. For commands, `creates` prevents execution when a matching path exists; `removes` prevents execution when no matching path exists. A false `when` condition skips a task. In contrast, `changed_when` controls reported change status and whether a task notifies handlers; it does not prevent execution. Choose a guard only when it actually represents the required state.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

Write down, in your own words, why attempt 2 passes the idempotence step and attempt 1 doesn't. If you can't say it without looking back, reread the two recaps side by side.

::::::::::::::::::::::::::::::::::::: challenge

### Follow-up practice outside the timed workshop

Come back to this after a break, a few hours at least, with the earlier solutions closed. Here's a new requirement:

> Every server must have the `tree` package installed. Every server must also have a file `/etc/pointcloud-release` containing the line `potree X`, where `X` is the Potree version the role installs (look in `defaults/main.yml` for the variable). The file is owned by root, mode 0644.

Write it as tasks in `maintenance.yml`. Before running anything, write down your prediction for the first and second runs. Then run `pixi run staging-up` twice and `(cd ansible && pixi run molecule idempotence)`. Check the file on the container yourself; don't rely on the recap.

:::::::::::::::::::::::::::::::::: solution

```yaml
- name: Troubleshooting tools are installed
  ansible.builtin.dnf:
    name: tree
    state: present

- name: Record which Potree release this host serves
  ansible.builtin.copy:
    dest: /etc/pointcloud-release
    content: "potree {{ potree_version }}\n"
    owner: root
    group: root
    mode: "0644"
```

First run: `changed=2`. Second run: `changed=0`, and `(cd ansible && pixi run molecule idempotence)` passes. The variable (`potree_version`) means the file follows the version if someone upgrades Potree, with no second edit. `cat /etc/pointcloud-release` on the container shows `potree 1.8.2`.

If you used `shell: echo ... > /etc/pointcloud-release`, the task reports `changed` every run even though the file never changes; that's the attempt 1 mistake again.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

## "No changes" is not "it works"

A playbook can be perfectly idempotent and still configure a broken site: if the Apache configuration had a typo, every run would happily put the same typo back. So Molecule has a separate **verify** step that checks the result, not the playbook. In pointcloud-infra, `ansible/molecule/default/verify.yml` checks things a visitor would notice: the listing has the UCLA skin and real titles, the viewer's JavaScript is served, plain HTTP redirects to HTTPS, descriptions appear.

```bash
pixi run staging-verify
```

The full test runs every step on a fresh container: create, converge, idempotence, verify, destroy.

```bash
pixi run test
```

This is what CI runs on every pull request.

::::::::::::::::::::::::::::::::::::: challenge

### Which step catches it?

For each problem, say whether the **idempotence** step, the **verify** step, or neither would catch it.

1. A task appends a line to a config file on every run.
2. A template typo makes Apache serve the default test page instead of the listing.
3. The S3 bucket policy changes and point clouds stop loading on the live site.

:::::::::::::::::::::::::::::::::: solution

1. Idempotence: the task reports `changed` on the second run.
2. Verify: the playbook converges cleanly (and idempotently), but the listing check fails.
3. Neither. Staging doesn't load S3 data, and the bucket isn't managed by this playbook. The outside-in health check can catch a failed metadata request, but not a binary-only or browser-rendering failure. Each kind of check covers different failures; none covers everything.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

When done, inspect `git diff` and remove only the tasks you added, preserving earlier work. Keep staging running for [Variables, templates and handlers](variables-handlers.md), or use `pixi run staging-down` and recreate it next time.

::::::::::::::::::::::::::::::::::::: keypoints

- A converge applies the described state; recap `ok` includes successful tasks that reported `changed`.
- A second run should report `changed=0`, but verify the resulting state as well.
- `creates`, `removes`, and `when` can prevent execution; `changed_when` only changes reporting and handler notification.
- Molecule's idempotence step checks the playbook; its verify step checks the result. You need both, and an outside-in check for what staging can't see.

::::::::::::::::::::::::::::::::::::::::::::::::
