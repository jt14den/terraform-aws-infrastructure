---
site: sandpaper::sandpaper_site
---

Learn infrastructure as code by configuring a small public Potree website with
Ansible on your laptop. Then use a separate Dataverse case study to reason about
Terraform, secrets, data migration, and operational risk.

This lesson is for library data-services staff, OSPO and DataSquad students, and
incoming service operators. You should be comfortable navigating a terminal,
editing text files, and using basic Git. No AWS account, cloud credentials, or
private repository access is required for the public core.

## Preparing for EX294

For learners aiming to pass EX294, the core is a first practice pass, not complete
exam preparation. After each worked example, close the solution and reproduce
the required state independently, then verify it. Use the
[certification track](learners/certification.md) for requirement-driven drills and
the explicit coverage gaps. Dataverse is optional for this goal; Terraform is not
an EX294 objective. Completing the website labs alone is not an exam-readiness test.

## Public core: six episodes

1. [Follow one point cloud request](episodes/pointcloud-request.md): trace HTTP requests and diagnose a failure boundary.
2. [First converge and second run](episodes/first-converge.md): write and repair a task, inspecting state as well as the recap.
3. [Variables, templates and handlers](episodes/variables-handlers.md): change a requirement and predict a reload.
4. [Organize a role, validate inputs](episodes/roles-validation.md): add collection content without duplicating infrastructure logic.
5. [Break, diagnose, recover](episodes/diagnose-recover.md): use evidence to repair a fault and rebuild locally.
6. [Operate the service](episodes/operate-service.md): interpret monitoring, release content, and roll back locally.

Allow about four hours plus breaks, with installation completed beforehand.
You will publish and verify a local collection landing page; rendering an actual
3D scan still depends on the live public service. The companion repository has
no verified offline point-cloud fixture workflow yet. Live exercises are labeled;
recorded outputs support discussion when that service is unavailable.

## Dataverse extension and optional practice

Start the [Dataverse case study](episodes/introduction.md) after the core. It
explains the larger stack and historical operational decisions. Its command
examples are for analysis, not a production runbook. Authorized maintainers need
their project's current repositories, access process, and reviewed runbook before
operating infrastructure. Public learners can reason from the included excerpts.

Staff can use the extension to explain deployment, monitoring, restore, and
cutover gates; reading it does not demonstrate production operating competence.
The [certification track](learners/certification.md) maps practice and gaps
separately. [Using AI](episodes/using-ai.md) adds an optional task-review exercise.

Start with [Setup](learners/setup.md).
