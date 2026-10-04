---
title: "Setup"
---

## Public core: your laptop only

Install Git, curl, [Pixi](https://pixi.sh), and [Podman](https://podman.io).
Use the companion repository's [getting started guide](https://github.com/ucla-data-science-center/pointcloud-infra/blob/main/docs/getting-started.md)
for your platform (macOS, Linux, or Windows through WSL). Allow 30–60 minutes
before the workshop for downloads and container setup. Internet access is needed
for dependencies; this is not an offline installation.

Clone into a new directory, or use your existing checkout without overwriting work:

```bash
git clone https://github.com/ucla-data-science-center/pointcloud-infra.git
cd pointcloud-infra
pixi run setup
pixi run ansible --version
pixi run staging-up
pixi run staging-verify
```

Open <https://localhost:8443/>. Accept the self-signed certificate only for this
local lab. Expect the collection listing and styling. Public S3-backed scans do
not normally load from localhost. The role skips firewalld and SELinux enforcement
in the container; this is not proof of production readiness.

All core commands start at the **pointcloud-infra repository root**, unless a
subshell explicitly changes directory. `pixi run` supplies the controller tools;
you do not need a Pixi shell. Check `git status --short` before each exercise,
use a learner branch, and preserve any previous work. Do not run `deploy`,
`deploy-check`, or the `tf-*` tasks during the core.

If Podman cannot connect, follow the platform guide to start its machine/service.
If ports 8080 or 8443 are busy, stop the conflicting local service before retrying.
If downloads fail, retain the error and check network access; do not substitute
production deployment. The instructor can demonstrate while you inspect source.

Leave staging running for the next exercise, or stop this lab container with
`pixi run staging-down`. Recreate it with `pixi run staging-up`.

The [request-tracing episode](../episodes/pointcloud-request.md) uses only curl
but depends on the live public site. All other core changes target local staging.
No verified local point-cloud fixture is provided yet; the exercises publish
HTML and test service behavior without claiming successful 3D rendering.

## Dataverse extension: read-only case study

No additional installation or credentials are needed to analyze the extension.
Private-repository exploration is optional and limited to authorized maintainers.
Its historical examples must not be copied into a live terminal as workshop tasks.

For actual operator onboarding, obtain repository access, the intended AWS account
and role/profile, SSH authorization, tool versions, and the existing Vault password
through the project's established process. This lesson does not specify that
process or supply credentials. A newly generated password cannot decrypt existing
project ciphertext. See [Tooling setup](../episodes/tooling-setup.md).

The [optional certification drills](certification.md) have a separate controller
environment and disposable nodes; they are not prerequisites for the core.
