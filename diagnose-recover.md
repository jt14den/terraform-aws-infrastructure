---
title: "Break, Diagnose, Recover"
teaching: 10
exercises: 25
---

:::::::::::::::::::::::::::::::::::::::::::::::: questions

- Which evidence distinguishes a content fault from a server failure?
- Can a rebuild reproduce the repaired state?

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: objectives

- Diagnose a planted content fault using HTTP and source evidence.
- Repair the source and verify the result.
- Recreate local staging to check reproducibility.

::::::::::::::::::::::::::::::::::::::::::::::::

## Plant a fault only in the local practice page

Continue at the companion repository root with the Workshop page from
[Organize a role, validate inputs](roles-validation.md). Preserve unrelated
changes. Before editing, request the page and record its heading.

A successful HTTP status can accompany incorrect content. This lab deliberately
keeps Apache working so that you must inspect more than availability.

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Find the broken requirement

Change the heading in `content/collections/Workshop/welcome.html` to
`Wrong collection`, leaving the title alone. Predict the HTTP status, then run:

```bash
pixi run staging-up
curl -k -s -o /dev/null -w '%{http_code}\n' https://localhost:8443/Workshop/welcome.html
curl -ksSf https://localhost:8443/Workshop/welcome.html
pixi run staging-verify
```

What does each observation establish? Can the existing verifier catch this
particular error? Repair the requirement in source, converge, and add a specific
check for the expected heading. Explain why a manual container edit is insufficient.

:::::::::::::::::::::::::::::::::::::::::::::::: solution

Expect 200 despite the wrong heading. The companion verifier checks built-in
collections and assets; it does not know the new Workshop requirement and may
pass. Restore `<h1>Workshop collection</h1>` in the source, then verify:

```bash
pixi run staging-up
curl -ksSf https://localhost:8443/Workshop/welcome.html | grep -F '<h1>Workshop collection</h1>'
pixi run staging-up
```

The final run should report no changes. If the grep fails, inspect the returned
body and the source before guessing at TLS or S3. Fixing only a container file
would leave incorrect source ready to overwrite it during the next converge.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Recover on a fresh target

Predict whether the fixed page survives destruction of the local container.
Recreate staging, then repeat the heading check and the existing verifier:

```bash
pixi run staging-down
pixi run staging-up
pixi run staging-verify
curl -ksSf https://localhost:8443/Workshop/welcome.html | grep -F '<h1>Workshop collection</h1>'
```

Record which source files are needed to reproduce the result.

:::::::::::::::::::::::::::::::::::::::::::::::: solution

The fixed HTML and description live in the local checkout and are republished
by the role. Destroying the container does not remove that source. The clean
converge should create the page with the correct heading. This tests recovery
of these files, not backup restoration of a database or S3 objects. If it fails,
retain the error and check whether the fix was made only inside the old container.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

Leave the page and staging running for [Operate the service](operate-service.md).

:::::::::::::::::::::::::::::::::::::::::::::::: keypoints

- A 200 response does not establish correct content.
- A verifier only checks requirements it actually asserts.
- Repair source, inspect the result, and rebuild to test reproducibility.

::::::::::::::::::::::::::::::::::::::::::::::::
