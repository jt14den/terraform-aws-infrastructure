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

The **Drill** notes below refer to the numbered drills further down this page. Objectives are grouped as on Red Hat's EX294 page (checked 2026-10-04). **Where** shows where you can see or practice each one in this lesson or in pointcloud-infra. **Gap** means you need a separate drill.

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

## Drills

Each drill is a requirement written the way the exam phrases tasks, with checks you run yourself and a time limit. Work from the requirement and `ansible-doc` only. Open the solution after you've attempted it, or when the time is up. Then rerun your own version and confirm the second run reports `changed=0`.

Drills 1 to 5 run on three practice nodes (containers) on your laptop, so they cover the Ansible mechanics. Storage, SELinux, firewalld and reboot persistence need virtual machines; see the end of this section.

### Drill 0: set up the practice nodes (10 minutes)

This is also an exam objective (installing collections), so do it by hand and don't skip ahead.

```bash
mkdir ~/ex294-drills && cd ~/ex294-drills
IMG=docker.io/geerlingguy/docker-rockylinux10-ansible:latest
for n in node1 node2 node3; do podman run -d --name $n --hostname $n --systemd=always $IMG /usr/sbin/init; done
```

Create `requirements.yml`:

```yaml
---
collections:
  - name: containers.podman
    version: "1.20.2"
  - name: ansible.posix
    version: "2.2.2"
  - name: community.general
    version: "11.4.9"   # last series supporting ansible-core 2.16
```

Then, from the exam environment (`pixi shell -e exam` in pointcloud-infra, then back to this directory):

```bash
ansible --version                      # should say core 2.16.x
ansible-galaxy collection install -r requirements.yml -p ./collections
```

::::::::::::::::::::::::::::::::::::: callout

### Why the pin matters

`community.general` 12.x needs ansible-core 2.17 or newer, and 13.x needs 2.18. Installing the newest version into the exam's 2.16 environment fails or misbehaves. When a module "doesn't work" and the code looks right, check the collection's `requires_ansible` in its `meta/runtime.yml`.

::::::::::::::::::::::::::::::::::::::::::::::::

Tear down when you're done studying: `podman rm -f node1 node2 node3`.

::::::::::::::::::::::::::::::::::::: challenge

### Drill 1: inventory and navigator (15 minutes)

Create `inventory.ini`, `ansible-navigator.yml` and `ping.yml` so that:

- `node1` and `node2` are in group `web`; `node3` is in group `db`; a group `prod` contains both groups.
- All hosts connect through Podman (`ansible_connection=containers.podman.podman`).
- `ansible-navigator.yml` sets stdout mode, disables the execution environment, and points at your inventory.
- `ping.yml` runs on `prod` and prints each host's groups.

**Checks**

1. `ansible-inventory --graph` shows `prod` containing `web` (node1, node2) and `db` (node3).
2. `ansible-playbook ping.yml` and `ansible-navigator run ping.yml` both succeed on all three nodes.
3. Node3's output says it's in `db, prod`.

:::::::::::::::::::::::::::::::::: solution

```ini
# inventory.ini
[web]
node1
node2

[db]
node3

[prod:children]
web
db

[all:vars]
ansible_connection=containers.podman.podman
```

```yaml
# ansible-navigator.yml
---
ansible-navigator:
  mode: stdout
  execution-environment:
    enabled: false
  ansible:
    inventory:
      entries:
        - inventory.ini
```

```yaml
# ping.yml
---
- name: Reach every node
  hosts: prod
  gather_facts: true
  tasks:
    - name: Show each host's groups
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} is in {{ group_names | join(', ') }}"
```

Plain `ansible-playbook` finds the inventory and your installed collections through an `ansible.cfg` in the same directory:

```ini
[defaults]
inventory = inventory.ini
collections_path = ./collections
```

The connection variable is only needed because these "servers" are containers; on real machines you'd rely on SSH.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Drill 2: failure handling (15 minutes)

On the `web` group, with privilege escalation, try to install the package `pointcloud-viewer-extras` (it doesn't exist). The play must not fail:

- If the install fails, write `pointcloud-viewer-extras: not available on <hostname>` to `/root/install-report.txt` (mode 0600).
- Whether or not it fails, print "Optional package step finished on <hostname>".

**Checks**

1. The recap shows `rescued=1` and `failed=0` for node1 and node2, and the play doesn't touch node3.
2. `podman exec node1 cat /root/install-report.txt` shows the message.
3. The "finished" message appears for both web nodes.

:::::::::::::::::::::::::::::::::: solution

```yaml
---
- name: Install an optional package, survive if it's missing
  hosts: web
  become: true
  tasks:
    - name: Try the optional package, report either way
      block:
        - name: Install the optional package
          ansible.builtin.dnf:
            name: pointcloud-viewer-extras
            state: present
      rescue:
        - name: Record the failure
          ansible.builtin.copy:
            dest: /root/install-report.txt
            content: "pointcloud-viewer-extras: not available on {{ inventory_hostname }}\n"
            mode: "0600"
      always:
        - name: Summarize
          ansible.builtin.debug:
            msg: "Optional package step finished on {{ inventory_hostname }}"
```

You'll see a red `fatal:` line for each node, which is expected: the failure happens, then `rescue` handles it. `rescued=1` in the recap is the proof. The `copy` task is idempotent, so the report file doesn't change on a second run, but the failing `dnf` task and the rescue still run each time.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Drill 3: Vault (20 minutes)

Create a variables file `secrets.yml` containing `app_db_password: "S3cret-drill-only"`, encrypted with Ansible Vault using a password file. Write a play for the `db` group that creates `/etc/drill-app.conf` (root, mode 0600) containing `db_password=<the value>`.

**Checks**

1. `head -1 secrets.yml` shows `$ANSIBLE_VAULT;...`, not your password.
2. Running with `-v` never prints the password: `ansible-playbook vault.yml --vault-password-file .vault-pass -v | grep -c S3cret` prints `0`.
3. `podman exec node3 stat -c '%a %U' /etc/drill-app.conf` prints `600 root`.
4. A second run reports `changed=0`.

:::::::::::::::::::::::::::::::::: solution

```bash
echo 'drill-vault-pass' > .vault-pass && chmod 600 .vault-pass
printf 'app_db_password: "S3cret-drill-only"\n' > secrets.yml
ansible-vault encrypt secrets.yml --vault-password-file .vault-pass
```

```yaml
# vault.yml
---
- name: Configure an app with a secret
  hosts: db
  become: true
  vars_files:
    - secrets.yml
  tasks:
    - name: Write the app config
      ansible.builtin.copy:
        dest: /etc/drill-app.conf
        content: "db_password={{ app_db_password }}\n"
        owner: root
        group: root
        mode: "0600"
      no_log: true
```

`no_log: true` keeps the password out of task output, including with `-v`. Never commit `.vault-pass`. In a real repository it's gitignored and kept somewhere else.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Drill 4: users, SSH keys, privilege escalation (25 minutes)

On every node in `prod`:

- A group `operators` with GID 2000.
- Users `automation` (UID 2001) and `auditor` (UID 2002), both in `operators` (as a supplementary group), shell `/bin/bash`.
- `automation` accepts an SSH key you generate (`ssh-keygen -t ed25519 -N '' -f drill_key`).
- `automation` has passwordless sudo, and the sudoers file is validated before it's installed.

**Checks**

1. `podman exec node2 id automation` shows UID 2001 and groups `automation` and `operators`.
2. `podman exec node2 sudo -l -U automation` ends with `(ALL) NOPASSWD: ALL`.
3. `podman exec node2 stat -c '%a %U' /home/automation/.ssh/authorized_keys` prints `600 automation`.
4. A second run reports `changed=0` on all three nodes.

:::::::::::::::::::::::::::::::::: solution

```yaml
---
- name: Operator accounts
  hosts: prod
  become: true
  vars:
    operators:
      - name: automation
        uid: 2001
      - name: auditor
        uid: 2002
  tasks:
    - name: Operators group exists
      ansible.builtin.group:
        name: operators
        gid: 2000

    - name: Operator users exist
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: operators
        append: true
        shell: /bin/bash
      loop: "{{ operators }}"
      loop_control:
        label: "{{ item.name }}"

    - name: Automation user accepts the control node's key
      ansible.posix.authorized_key:
        user: automation
        key: "{{ lookup('ansible.builtin.file', 'drill_key.pub') }}"

    - name: Sudo is installed
      ansible.builtin.dnf:
        name: sudo
        state: present

    - name: Automation user has passwordless sudo
      ansible.builtin.copy:
        dest: /etc/sudoers.d/automation
        content: "automation ALL=(ALL) NOPASSWD: ALL\n"
        owner: root
        group: root
        mode: "0440"
        validate: /usr/sbin/visudo -cf %s
```

`validate:` matters: a syntax error in a sudoers file can lock everyone out, so Ansible checks the new file before moving it into place. First run `changed=4` on each node (group, users, key, sudoers; the package was already there), second run `changed=0`.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Drill 5: scheduling, file content, archiving (20 minutes)

On the `web` group:

- `cronie` installed and `crond` running and enabled.
- A root cron job named `dnf makecache` that runs `/usr/bin/dnf -q makecache` at 02:30 every day.
- `/etc/dnf/dnf.conf` contains exactly one line `max_parallel_downloads=10`, even if the setting is already there with another value.
- A gzip archive of `/etc/dnf` at `/root/dnf-config.tar.gz`, mode 0600.

**Checks**

1. `podman exec node1 crontab -l` shows `30 2 * * * /usr/bin/dnf -q makecache`.
2. `podman exec node1 grep -c max_parallel /etc/dnf/dnf.conf` prints `1`.
3. `podman exec node1 tar -tzf /root/dnf-config.tar.gz` lists files under `dnf/`.
4. A second run reports `changed=0`.

:::::::::::::::::::::::::::::::::: solution

```yaml
---
- name: Housekeeping on web servers
  hosts: web
  become: true
  tasks:
    - name: Cron is installed and running
      ansible.builtin.dnf:
        name: cronie
        state: present

    - name: Cron service is enabled
      ansible.builtin.service:
        name: crond
        state: started
        enabled: true

    - name: Nightly metadata refresh
      ansible.builtin.cron:
        name: dnf makecache
        user: root
        minute: "30"
        hour: "2"
        job: /usr/bin/dnf -q makecache

    - name: Faster downloads
      ansible.builtin.lineinfile:
        path: /etc/dnf/dnf.conf
        regexp: '^max_parallel_downloads='
        line: max_parallel_downloads=10

    - name: Back up /etc/dnf
      community.general.archive:
        path: /etc/dnf
        dest: /root/dnf-config.tar.gz
        format: gz
        mode: "0600"
```

`lineinfile` with a `regexp` replaces the existing line instead of adding a duplicate, which is what keeps it idempotent. The archive module only rewrites the archive when the contents change.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

### Drills that need virtual machines

Containers share the host's kernel and have no real disks, boot process, or SELinux. These exam topics need VMs, and they're waiting on the exam lab (Rocky Linux machines built with Terraform):

- Storage: partitions, LVM, file systems, persistent mounts (`community.general.lvg`, `lvol`, `filesystem`; `ansible.posix.mount`)
- SELinux: file contexts and booleans (`community.general.sefcontext`, `ansible.posix.seboolean`, `ansible.builtin.command: restorecon`)
- Firewall rules with `ansible.posix.firewalld`, on a machine whose firewall is actually running
- "Persists after reboot": run, reboot, check again

Until the lab exists, read the `ansible-doc` pages for those modules and write the playbooks, but don't treat a playbook as done until it has been run on a real machine and survived a reboot.

## Practice environments

| What you're practicing | Where |
|---|---|
| Modules, templates, handlers, roles, idempotence | Local staging: `pixi run staging-up` (Rocky Linux 10 container) |
| Inventories, users, Vault, cron, failure handling, collections, navigator | The three practice nodes from Drill 0 |
| Storage, file systems, SELinux, firewalld, reboot persistence | Virtual machines. A Terraform-built exam lab (one control node, three or four managed Rocky Linux nodes, destroyed after each session) is planned for this lesson |
| Timed, requirement-only practice | Clean VMs, a requirements list, `ansible-doc`, a timer, no AI |

## Outside resources

- [Red Hat's EX294 page](https://www.redhat.com/en/services/training/ex294-red-hat-certified-engineer-rhce-exam-red-hat-enterprise-linux): the authoritative objective list. Read it again the week you book.
- [Lisenet's EX294 sample exam](https://www.lisenet.com/2019/ansible-sample-exam-for-ex294/): free, 18 practical tasks on a control node plus four managed nodes. Written in 2019 for RHEL 8, so use it as a requirements bank, not a guide to current coverage.
- Sander van Vugt's EX294 video course (2nd edition): guided labs with explained solutions. Check which RHEL and tooling versions it covers before buying.
