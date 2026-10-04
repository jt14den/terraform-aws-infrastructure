---
title: "Variables, Templates and Handlers"
teaching: 15
exercises: 20
---

:::::::::::::::::::::::::::::::::::::::::::::::: questions

- How does one requirement become a configuration file?
- When should a handler run?

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: objectives

- Trace a variable through a template to a managed file.
- Change a listing description and predict handler notification.
- Verify the rendered response and a second converge.

::::::::::::::::::::::::::::::::::::::::::::::::

## Start from working staging

From the pointcloud-infra repository root, run `pixi run staging-up` if staging
is stopped. Preserve your existing edits. In this episode change only
`content/descriptions.yml`, and record the original Iceland folder description
so you can restore it afterwards.

The role separates inputs (`defaults/main.yml` and playbook variables), templates
(`templates/`), tasks (`tasks/`), and handlers (`handlers/main.yml`). Read
`ansible/playbooks/group_vars/all.yml`: it loads `content/descriptions.yml` into
`potree_descriptions`. Then read the role's `templates/pointcloud-descriptions.inc.j2`
and the "Install listing descriptions" task in `tasks/apache.yml`.

A template renders variables into text. The `template` module compares that text
with the destination file; if the file changes, this task notifies `Reload httpd`.
Two handlers listen to that name: a configuration syntax check and a reload.
Handlers normally run after the play's tasks and repeated notifications coalesce.
The syntax-check handler uses `changed_when: false`; it still executes.

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Change one requirement

Requirement: the root listing must describe Iceland as `Workshop collection`.
Change the **existing** `Iceland` entry under `folders` in `content/descriptions.yml`.
Do not add a second `folders` mapping or edit a generated file in the container.

Before running, predict which task will notify a handler and whether the handler
should run on a second converge. Then run:

```bash
pixi run staging-up
curl -ksSf https://localhost:8443/ | grep 'Workshop collection'
podman exec pointcloud-staging apachectl configtest
pixi run staging-up
```

Record the changed task, handlers, HTTP response, and second recap. Explain why
changing this description should reload Apache but editing an ordinary HTML page
need not do so. Use `-k` only for this local self-signed lab.

:::::::::::::::::::::::::::::::::::::::::::::::: solution

Change the entry to `Iceland: "Workshop collection"`. The description template
writes `/etc/httpd/conf.d/pointcloud-descriptions.inc`. Apache must reload to use
that configuration. Expect the template to report a change and both listening
handlers to run. On the second run, unchanged template output should not notify
them. The response check confirms the new description is actually served; a
recap alone cannot do that. Ordinary HTML is read as content without a config reload.

If the description is absent, inspect indentation and the exact folder name,
then inspect the generated include file. A successful syntax check only proves
Apache accepts the configuration, not that the description is correct.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

Restore only your description edit and converge again. Run `pixi run staging-verify`
after restoring the original content; the existing verifier checks some specific
descriptions. Continue to [Organize a role, validate inputs](roles-validation.md).

:::::::::::::::::::::::::::::::::::::::::::::::: keypoints

- Variables carry requirements; templates render them into managed files.
- Changed template output can notify handlers; reporting no change suppresses notification.
- Check the served result as well as configuration syntax.

::::::::::::::::::::::::::::::::::::::::::::::::
