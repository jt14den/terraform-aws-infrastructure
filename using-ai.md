---
title: "Using AI Tools in Infrastructure Work"
teaching: 20
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- Where does AI actually help with infrastructure work, and where does it mislead?
- What should I never share with an AI tool?
- How do I stay technically grounded when AI is doing a lot of the work?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify tasks where AI assistance is genuinely useful in this context.
- Identify tasks where AI output requires careful verification or should not be trusted.
- Apply a consistent habit of sanitizing inputs before sharing anything with an AI tool.
- Name at least two strategies for staying grounded when AI accelerates your output.

::::::::::::::::::::::::::::::::::::::::::::::::

## What this episode is about

AI tools are useful in infrastructure work. They are also easy to misuse -- not through
dramatic failures, but through subtle ones: plausible-looking config that is slightly
wrong, outdated API syntax, or confident answers to questions the tool cannot actually know.

This episode covers two things. First, the practical mechanics: what to ask, what to
verify, and what to keep out of the conversation entirely. Second, something less often
discussed: what happens to your own understanding when AI does a lot of the heavy lifting,
and what you can do about it.

## What your institution has licensed

Before using any AI tool for work, check what your institution has approved.
Institutional licenses typically include data handling agreements that consumer-grade
tools do not. Using an unlicensed tool for work-related content -- especially anything
involving infrastructure, access credentials, or user data -- may violate your institution's
acceptable use policy.

At UCLA, the institution licenses **Gemini** for work use via Google Workspace.
This is the appropriate tool for UCLA Library staff working on infrastructure tasks.
When in doubt, check with your IT or information security team about what is approved.

Using a licensed tool does not mean anything goes -- it means the data handling terms
are known and accepted. The practices in this episode still apply.

## What not to share

Some content should never be pasted into an AI tool, regardless of which tool or
what license covers it:

- Secrets and credentials: passwords, API tokens, Ansible Vault contents, AWS access keys
- Private hostnames and IPs: internal RDS endpoints, EC2 instance addresses, VPN addresses
- Database connection strings: these combine a hostname, port, database name, username, and password
- SSH private keys
- Personally identifiable information: user account data, emails, names from the database

For infrastructure work, this means sanitizing before sharing.
Replace real values with placeholders:

```yaml
# what you have
db_host: dev-dataverse-db.cb4k4a6gqn27.us-west-2.rds.amazonaws.com
db_password: "actual-password-here"

# what you share with AI
db_host: <rds-endpoint>
db_password: "<redacted>"
```

The AI does not need the real values to help you. Hostnames and passwords do not change
what an Ansible task should look like or why a Terraform plan is failing.

## Where AI helps

**Config systems and infrastructure-as-code**

This is where AI tools are genuinely strong, and it is directly relevant to this lesson.
Terraform and Ansible are pattern-heavy, well-documented, and structurally consistent.
AI tools have seen enormous amounts of both. They are good at:

- Generating correct HCL resource blocks from a description
- Explaining what an Ansible module does and what its options mean
- Debugging Terraform plan output or Ansible task failures
- Translating "here is what I want to happen" into the right module or resource type

This is not a coincidence. Config systems are exactly the kind of domain where AI excels:
large training corpus, consistent syntax, well-defined semantics, errors with recognizable patterns.

**Understanding unfamiliar tools and concepts**

When you encounter something new, AI gives you a faster orientation than documentation alone.
"What is Payara and how does it relate to GlassFish?" or "explain what Ansible handlers do
and when they run" produces a usable explanation quickly, tailored to your context.

**Explaining error messages**

Paste a sanitized error message and ask what it means. For common errors from Terraform,
Ansible, Payara, or AWS, AI tools often recognize the pattern and explain the root cause
and likely fixes faster than searching.

**Drafting documentation and runbooks**

AI is useful for turning rough notes into structured documentation -- filling in a
runbook outline, writing a glossary entry, turning a list of steps into a coherent
procedure. This lesson was partly developed that way.

**Talking through a problem**

Describing a problem before asking a specific question often helps clarify your own
thinking. "I ran make rebuild and Payara is up but Dataverse search returns no results --
what are the possible causes?" returns a structured list you can work through.

## Where AI misleads

**Version-specific configuration**

AI training data has a cutoff. Dataverse, Payara, Terraform provider syntax, and Ansible
module arguments all change between versions. The AI may confidently give you configuration
syntax that was correct in an older version and is now wrong or deprecated.

Always check the version-specific documentation for anything configuration-related.
Do not assume AI-generated config is correct for the version you are running.

**Your specific environment**

The AI does not know your S3 bucket names, your RDS endpoint, your Terraform state,
or what is currently deployed. When you need environment-specific answers, the tools are:
AWS console, Terraform state, Ansible `--check` mode, and the Payara log.
AI is a supplement to these, not a replacement.

**Plausible but wrong**

AI tools produce fluent, confident output even when they are wrong. A hallucinated
Terraform resource argument or a non-existent Ansible module option looks exactly
like a correct one. The text is convincing; the output may not work.

This is the central risk. Never copy AI-generated configuration into a production
context without running it through the validation steps: `terraform plan`, `--check` mode,
the test suite.

**Security decisions**

Do not outsource security decisions to an AI tool. Questions like "is this IAM policy
safe?" benefit from AI explanation, but the final judgment belongs to a person who
understands the actual threat model and has access to the full context.

---

## The deeper risk: losing contact

Because AI is particularly good at config systems -- Terraform, Ansible, CI pipelines --
it is possible to move very fast without fully understanding what was built.
The acceleration is real. So is the risk.

This is not unique to AI. It is a well-studied pattern in automation research.
When a system handles the hard parts reliably, people stop building the mental model
that lets them catch when it is wrong. In aviation and medicine, this is called
**automation bias**: over-trusting automated output because it is usually right,
even when evidence of an error is present. The cognitive load of staying engaged is high,
and the tool usually gets it right -- until it doesn't.

For infrastructure work, the specific version looks like this:

- AI writes your Ansible task. It works. You move on.
- Six months later, the task behaves unexpectedly in a new context.
- You do not have the mental model to know why, because you never built it.

The output was correct. The understanding was not transferred.

A useful framing here comes from chess: after computers became stronger than humans,
the best human-computer teams were not grandmasters who let the engine decide everything,
but players who kept their own judgment engaged and used the engine to check it.
The technical term in HCI research is **cognitive offloading** -- and the question is not
whether to do it, but which parts you can safely offload and which you must own.

For this work, the parts you need to own:

- Why idempotency matters and how to verify it
- What each layer (Terraform, Ansible, Dataverse) is responsible for
- What the test suite is actually asserting
- What the migration phases gate and why they are sequenced the way they are

These are not things AI can hold for you. If you cannot explain them without opening a
chat window, you do not have them yet.

## Strategies for staying grounded

**Write it down in your own words (this lesson)**

Writing this lesson is itself a strategy. Turning what was built into structured
explanation -- not AI-generated, but your own reconstruction -- forces contact with
the material. If you cannot explain the Solr reindex requirement in a callout block,
you do not understand it yet.

This is the oldest study technique there is, and it works: retrieval practice, or
the "protege effect." Teaching something consolidates the understanding in a way
that reading or watching does not.

**Ask for explanation, not just output**

Instead of "write me an Ansible task to configure Payara JVM options," try:
"explain how Ansible configures Payara JVM options, then show me an example task."
The explanation gives you something to evaluate the output against.

If the explanation does not make sense, do not use the output.

**State your versions and read the diff**

When asking for configuration, include the versions you are running: "Terraform 1.7,
AWS provider 5.x, Ansible 2.16, Dataverse 6.3." This reduces outdated syntax.

Then read the generated output line by line, the same way you would read a code review.
If you cannot explain what a line does, look it up before using it.

**Validate before applying -- always**

Every piece of AI-generated configuration should go through the same path as everything
else: `terraform plan`, `--check` mode, the test suite.

Make this non-negotiable. AI output is a draft, not a finished product.

**Practice without the tool periodically**

Work through a piece of configuration or a debugging problem without AI assistance,
even if it takes longer. This is not about refusing help -- it is about checking that
the skill is still there when you need it. Pilots practice manual landings even though
automation is usually better. The same logic applies.

**Pair with another person**

When another person is present -- Jamie, a student, a colleague -- explain what you
are doing and why. This is not about the other person checking your work; it is about
the act of explaining. If you cannot explain it, you do not understand it.
Onboarding Leigh or a new DataSquad student is actually useful for this: teaching
someone who does not have the context forces you to articulate things you have internalized.

::::::::::::::::::::::::::::::::::::: callout

### A question worth asking periodically

Could you rebuild this environment -- Terraform, Ansible, Dataverse configuration --
from scratch without AI assistance if you had to?

Not immediately, and not perfectly. But could you trace the steps, know what to look for,
and understand why each piece is there?

If the answer is uncertain, that is a signal. Not to stop using AI, but to spend
some time with the material on your own terms.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Sanitize a config snippet

Take the following snippet from a group_vars file:

```yaml
dataverse_db_host: prod-dataverse-db.abc123xyz.us-west-2.rds.amazonaws.com
dataverse_db_password: "s3cur3P@ssw0rd!"
dataverse_s3_bucket: ucla-dataverse-production
dataverse_doi_authority: "10.25346"
```

Rewrite it as you would share it with an AI tool when asking for help with a
configuration problem. What is safe to share? What should be replaced?

:::::::::::::::::::::::::::::::::: solution

```yaml
dataverse_db_host: <rds-endpoint>
dataverse_db_password: "<redacted>"
dataverse_s3_bucket: <s3-bucket-name>
dataverse_doi_authority: "10.25346"
```

The DOI authority (`10.25346`) is a public value -- safe to share.
The RDS endpoint, password, and bucket name should be replaced.
The bucket name is not a credential, but it identifies your institution's storage;
use a placeholder when the real name is not needed to answer the question.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Use your institution's licensed AI tools for work; at UCLA, that is Gemini via Google Workspace.
- Never share credentials, private hostnames, database connection strings, or SSH keys with any AI tool.
- AI is particularly strong with config systems like Terraform and Ansible -- and that is exactly where the risk of losing contact is highest.
- Automation bias is real: over-trusting AI output because it is usually right, until it isn't.
- Strategies for staying grounded: write it down in your own words, ask for explanation before output, validate everything, practice without the tool periodically, teach it to someone else.
- Writing this lesson is itself one of those strategies.

::::::::::::::::::::::::::::::::::::::::::::::::
