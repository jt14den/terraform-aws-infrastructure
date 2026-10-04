---
title: "Operate the Service"
teaching: 15
exercises: 25
---

:::::::::::::::::::::::::::::::::::::::::::::::: questions

- What does a green monitor actually tell us?
- How do we release and roll back content locally?

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::: objectives

- Interpret the scope of the existing health monitor.
- Release a local content change and verify it.
- Roll back through source and explain the limits of that rollback.

::::::::::::::::::::::::::::::::::::::::::::::::

## Read the monitor before trusting it

At the pointcloud-infra root, read `scripts/site-check.sh`. It checks the canonical
homepage's status, an exact bare-name redirect, a metadata URL extracted from a
collection page (with a supplied Referer), and certificate expiry on both names.
It does not execute the viewer or fetch point-cloud binaries.

Running `pixi run site-check` contacts the **live public service**. It is optional
here; use the following illustrative output for the exercise without contacting
that service:

```output
ok   https://www.pointcloud.ucla.edu/ -> 200
ok   https://pointcloud.ucla.edu/Iceland/Torfljar.html -> 301 https://www.pointcloud.ucla.edu/Iceland/Torfljar.html
ok   Iceland/Torfljar.html: point cloud metadata loads (200)
FAIL www.pointcloud.ucla.edu: certificate expires in 5 days
```

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Choose an operational response

Does this output show a current site outage? What needs attention, and what
further test would establish that a visitor can view a scan?

:::::::::::::::::::::::::::::::::::::::::::::::: solution

The HTTP checks passed at that moment; the expiry threshold failed before the
certificate expired. An operator should investigate renewal and monitor it,
not wait for an outage. A browser test must also check JavaScript, CORS, binary
requests, and rendering. The output does not prove those work.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

## Release and roll back in staging

The role's `tasks/content.yml` and `tasks/release.yml` install content in verified
release directories, then switch `/var/www/pointcloud` to the active release.
Older releases are pruned according to `potree_keep_releases`. A source rollback
can recreate a release even if the old target directory was pruned, provided its
inputs and dependencies remain available.

:::::::::::::::::::::::::::::::::::::::::::::::: challenge

### Release a visible change, then restore the previous requirement

Use the Workshop page from the previous episodes. Record its full contents and
the current symlink target:

```bash
podman exec pointcloud-staging readlink /var/www/pointcloud
```

Change the heading to `Workshop collection, release 2`. Predict whether Apache
needs to reload for this HTML-only change. Converge, inspect the symlink again,
and fetch the page. Then restore only your HTML edit in source, converge again,
and verify the original heading:

```bash
pixi run staging-up
podman exec pointcloud-staging readlink /var/www/pointcloud
curl -ksSf https://localhost:8443/Workshop/welcome.html
```

Finally run `pixi run staging-verify` and a second converge. Explain what was
rolled back, and what was not.

:::::::::::::::::::::::::::::::::::::::::::::::: solution

The changed content produces a different archive checksum and release directory.
An HTML-only edit does not require an Apache configuration reload. Restoring the
previous file and converging republishes the old content through the same role;
verify the original heading instead of relying only on the link name or recap.

This rolls back static content. It does not reverse database writes, DOI
registration, dependencies, or S3 data changes. Production release and rollback
need their own reviewed procedure and acceptance checks.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

## Finish the public core

Inspect your diff. Remove only the Workshop folder and description you created
(or keep them on your learner branch), then converge and verify if you want a
baseline staging site. Stop the local container with `pixi run staging-down`.

You have practiced publication, verification, diagnosis, and a local content
rollback. Offline 3D rendering and real-host firewall/SELinux acceptance remain
outside this lab. Continue to the separate [Dataverse case study](introduction.md),
optional [AI review exercise](using-ai.md), or [certification practice](../learners/certification.md).

:::::::::::::::::::::::::::::::::::::::::::::::: keypoints

- Monitors provide bounded evidence, including early warnings.
- Verify a release with a requirement-specific check.
- Static-content rollback does not establish data-migration rollback safety.

::::::::::::::::::::::::::::::::::::::::::::::::
