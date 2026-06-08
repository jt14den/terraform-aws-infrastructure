---
title: "Using AI Tools in Infrastructure Work"
teaching: 15
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions

- Where does AI actually help with infrastructure work, and where does it mislead?
- What should I never share with an AI tool?
- How do I use AI productively without skipping validation?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify tasks where AI assistance is genuinely useful in this context.
- Identify tasks where AI output requires careful verification or should not be trusted.
- Apply a consistent habit of sanitizing inputs before pasting anything into an AI tool.
- Know which AI tools your institution has licensed for work use.

::::::::::::::::::::::::::::::::::::::::::::::::

## What this episode is about

AI tools are useful in infrastructure work. They are also easy to misuse -- not through
dramatic failures, but through subtle ones: plausible-looking config that is slightly
wrong, outdated API syntax, or confident answers to questions the tool cannot actually know.

This episode is about developing a clear-eyed working relationship with these tools:
knowing what to ask, what to verify, and what to keep out of the conversation entirely.

## What your institution has licensed

Before using any AI tool for work, check what your institution has approved.
Institutional licenses typically include data handling agreements that consumer-grade
tools do not. Using an unlicensed tool for work-related content -- especially anything
involving infrastructure, access credentials, or user data -- may violate your institution's
acceptable use policy.

At UCLA, the institution licenses **Gemini** for work use (via Google Workspace).
This is the appropriate tool for UCLA Library staff working on infrastructure tasks.
When in doubt, check with your IT or information security team about what is approved.

Using a licensed tool does not mean anything goes -- it means the data handling terms
are known and accepted. You still need to follow the practices in this episode.

## What not to share

Some content should never be pasted into an AI tool, regardless of which tool or
what it is licensed for:

- **Secrets and credentials**: passwords, API tokens, Ansible Vault contents, AWS access keys
- **Private hostnames and IPs**: internal RDS endpoints, EC2 instance addresses, VPN addresses
- **Database connection strings**: these combine a hostname, port, database name, username, and password
- **SSH private keys**: never, ever
- **Personally identifiable information**: user account data from the database, emails, names

For infrastructure work specifically, this means you should sanitize before sharing.
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

**Understanding unfamiliar tools and concepts**

When you encounter a tool, service, or concept you have not worked with before,
AI can give you a faster orientation than documentation alone. Asking "what is Payara
and how does it relate to GlassFish?" or "explain what Ansible handlers do and when
they run" gets you a usable explanation quickly.

This is one of the highest-value uses. The tool is good at synthesizing documentation
into plain explanations tailored to your context.

**Explaining error messages**

Paste an error message (sanitized) and ask what it means. For common errors from
Terraform, Ansible, Payara, or AWS, AI tools often recognize the error pattern and
can explain the root cause and likely fixes faster than searching.

**Generating boilerplate**

Writing a new Ansible task, a Terraform resource block, or a Makefile target follows
patterns. AI is good at generating correct structure for common cases. Use it to get
a starting point, then read and adjust rather than writing from scratch.

**Drafting documentation and runbooks**

AI is useful for turning rough notes into structured documentation -- filling in a
runbook outline, writing a glossary entry, or turning a list of steps into a coherent
procedure. This lesson was partly developed this way.

**Talking through a problem**

Describing a problem to an AI tool, even before asking a specific question, often helps
clarify your own thinking. "I ran make rebuild and Payara is up but Dataverse search
returns no results -- what are the possible causes?" returns a structured list you can
work through.

## Where AI misleads

**Version-specific configuration**

AI training data has a cutoff. Dataverse, Payara, Terraform provider syntax, and Ansible
module arguments all change between versions. The AI may confidently give you configuration
syntax that was correct in an older version and is now wrong or deprecated.

Always check the version-specific documentation for anything configuration-related.
Do not assume AI-generated config is correct for the version you are running.

**Your specific environment**

The AI does not know your S3 bucket names, your RDS endpoint, your Terraform state,
or what is currently deployed. It cannot tell you "why is my EC2 instance not coming up"
based on your environment -- it can only suggest things to check.

When you need environment-specific answers, the tools are: AWS console, Terraform state,
Ansible `--check` mode, and the Payara log. AI is a supplement to these, not a replacement.

**Plausible but wrong**

AI tools produce fluent, confident output even when they are wrong. A hallucinated
Terraform resource argument or a non-existent Ansible module option will look
exactly like a correct one. The text is convincing; the output may not work.

This is the central risk. Never copy AI-generated configuration into a production
context without running it through the validation steps: `terraform plan`, `--check` mode,
the test suite.

**Security decisions**

Do not outsource security decisions to an AI tool. Questions like "is this IAM policy
safe?" or "does this security group expose anything it shouldn't?" benefit from AI
explanation, but the final judgment belongs to a person who understands the actual
threat model and has access to the full context.

## A structured approach

These habits reduce the risk of AI misleading you without giving up the benefit:

**Sanitize before sharing.** Replace real hostnames, passwords, and identifiers with
placeholders before pasting anything into an AI tool. Make this automatic.

**State your versions.** When asking about configuration, include the versions you are
running: "Terraform 1.7, AWS provider 5.x, Ansible 2.16, Dataverse 6.3." This narrows
the search space and reduces the chance of getting outdated syntax.

**Ask for explanation, not just output.** Instead of "write me an Ansible task to configure
Payara JVM options," try "explain how Ansible configures Payara JVM options, then show me
an example task." The explanation helps you verify the output.

**Verify before applying.** Every piece of AI-generated configuration should go through
the same validation path as anything else: `terraform plan`, `--check` mode, test suite.
AI output is a draft, not a finished product.

**Use it to understand, not to skip understanding.** The goal of this lesson is to build
a real mental model of the infrastructure. Using AI to generate config you do not understand
defeats that goal. Use it to accelerate learning, not to bypass it.

::::::::::::::::::::::::::::::::::::: challenge

### Sanitize a config snippet

Take the following snippet from a real group_vars file:

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

The DOI authority (`10.25346`) is a public value and safe to share.
The RDS endpoint, password, and bucket name should be replaced.
The bucket name is not a credential, but it identifies your institution's storage --
use a placeholder when context does not require the real name.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Use your institution's licensed AI tools for work; at UCLA, that is Gemini via Google Workspace.
- Never share credentials, private hostnames, database connection strings, or SSH keys with any AI tool.
- Sanitize config snippets by replacing real values with placeholders before sharing.
- AI is strong at explanation, error interpretation, boilerplate generation, and documentation drafting.
- AI is weak on version-specific config, your specific environment, and security decisions.
- Every piece of AI-generated configuration needs the same validation as anything else: `terraform plan`, `--check`, test suite.

::::::::::::::::::::::::::::::::::::::::::::::::
