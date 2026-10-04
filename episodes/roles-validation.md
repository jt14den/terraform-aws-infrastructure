---
title: "Organize a Role, Validate Inputs"
teaching: 15
exercises: 25
---

:::::::::::::::::::::::::::::::::::::::::::::::: questions

- Where does collection content belong?
- What can role input validation prove?

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: objectives

- Locate the public interface and implementation of a role.
- Add a collection landing page without copying tasks.
- Diagnose an invalid input and distinguish type checks from functional tests.

::::::::::::::::::::::::::::::::::::::::::::::::

## Use the existing role

Work at the pointcloud-infra root with local staging running. Read these files
under `ansible/roles/potree/`:

| File | Responsibility |
|---|---|
| `defaults/main.yml` | Default inputs, including release and TLS settings |
| `meta/argument_specs.yml` | Input types, required values, allowed choices |
| `tasks/main.yml` | Order of configuration work |
| `tasks/content.yml` | Pack, verify, and activate collection content |
| `tasks/release.yml` | Verify installed files and repair an incomplete release |

The role is already organized. Your task is to use its interface, not duplicate
its package, Apache, or release tasks for each collection. In `playbooks/group_vars/all.yml`,
`potree_content_src` points to `content/collections/` relative to the playbook.

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Publish a small collection landing page

Create a new folder `content/collections/Workshop/` (choose another name and
adapt the commands if it already exists). Add `welcome.html` with a heading
`Workshop collection` and a sentence explaining that this is a local practice
page, not a 3D viewer. Add `Workshop: "Local practice collection"` under the
existing `folders` mapping in `content/descriptions.yml`.

Predict whether this needs any new infrastructure task. Run and verify:

```bash
pixi run staging-up
curl -ksSf https://localhost:8443/Workshop/welcome.html
curl -ksSf https://localhost:8443/ | grep 'Local practice collection'
```

Inspect `git diff` and explain how the new content reached Apache.

:::::::::::::::::::::::::::::::::::::::::::::::: solution

One suitable `welcome.html` is:

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>Workshop collection</title></head>
<body><h1>Workshop collection</h1><p>Local practice page, not a 3D viewer.</p></body>
</html>
```

The existing content task packs the collection directory, verifies a release
against its manifest, and points the web-root symlink at it. Apache serves the
new page. The description template handles the listing text. No copied role or
new cloud resource is needed. This verifies HTML publication, not cloud rendering.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Reject a bad input before configuration

Read the choices for `potree_tls_mode` in the argument specification. Predict
what happens with `selfsigned_typo`, then run this **local Molecule** command:

```bash
(cd ansible && pixi run molecule converge -- -e potree_tls_mode=selfsigned_typo)
```

Find the validation error. Correct the input by removing the command-line override:

```bash
pixi run staging-up
pixi run staging-verify
```

Does validating the input prove the site works?

:::::::::::::::::::::::::::::::::::::::::::::::: solution

The permitted modes are `selfsigned` and `letsencrypt`. Role argument validation
rejects the typo before the configuration tasks run. The command-line override
is not saved to a file, so the normal staging task restores the scenario's
`selfsigned` input. A valid choice does not prove a certificate, network path,
or viewer works; the HTTP and functional checks remain necessary.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

Keep the Workshop page for [Break, diagnose, recover](diagnose-recover.md).
A real offline scan requires metadata, binary data, a viewer page pointing to
local URLs, and a tested workflow. Those fixtures are not provided here.

:::::::::::::::::::::::::::::::::::::::::::::::: keypoints

- Reuse a role by changing inputs and content, not by copying infrastructure tasks.
- Input validation catches some errors early; it does not replace service checks.
- HTML publication and successful point-cloud rendering are different outcomes.

::::::::::::::::::::::::::::::::::::::::::::::::
