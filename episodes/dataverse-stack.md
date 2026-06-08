---
title: "The Dataverse Stack: Payara, Solr, and the Data Layer"
teaching: 30
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What is Payara and why does Dataverse use it?
- How does Dataverse store and retrieve data across RDS, S3, and Solr?
- What breaks when Solr is out of sync, and how do you fix it?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Describe Payara's role as a Jakarta EE application server.
- Explain how Dataverse configuration is applied via JVM options and the API.
- Describe how RDS, S3, and Solr each serve different data needs.
- Know when and why a Solr reindex is required.

::::::::::::::::::::::::::::::::::::::::::::::::

## Payara: the application server

Payara is a Jakarta EE application server -- a runtime environment for Java web applications.
It is a community fork of GlassFish, maintained specifically for Jakarta EE compatibility.
Dataverse chose it because Dataverse is a Jakarta EE application and Payara has continued
to receive active maintenance after GlassFish development slowed.

Think of Payara the way you might think of a Python WSGI server or a Node.js process:
it is the process that loads the application, handles incoming requests, manages database
connections, and keeps the application running.

Dataverse is distributed as a WAR file -- a Web Application Archive. Ansible deploys this
WAR file into Payara during installation. Payara unpacks it and starts serving requests.

### Payara ports

Payara listens on several ports by default:

| Port | Purpose |
|---|---|
| 8080 | HTTP (application) |
| 8181 | HTTPS (application) |
| 4848 | Admin console |
| 9009 | Debug |

Apache sits in front of ports 80 and 443. Requests come in through Apache and are
proxied to Payara on port 8080 or 8181. Port 4848 is the Payara admin console --
it should not be publicly accessible and is locked down in the security group.

### Configuring Dataverse through Payara JVM options

Most Dataverse configuration is not stored in a config file. Instead, it is set as
JVM options in Payara's domain configuration (`domain.xml`). These are key-value pairs
that Dataverse reads at startup:

```
-Ddataverse.files.s3-bucket-name=ucla-dataverse-storage
-Ddataverse.files.storage-driver-id=s3
-Ddataverse.db.host=dev-dataverse-db.cb4k4a6gqn27.us-west-2.rds.amazonaws.com
```

Ansible sets these JVM options through the Payara admin API during the configure step.
You can also inspect or change them manually through the Payara admin console on port 4848,
but Ansible is the source of truth -- any manual changes will be overwritten on the next
playbook run.

### First boot and the Dataverse API

Some configuration can only be applied after Dataverse is running. During first boot,
Ansible calls the Dataverse API to:

- Set the root dataverse name and contact email
- Configure the DOI provider (FAKE for test environments, real EZID for production)
- Set storage credentials
- Apply any site-level settings

This is why `make rebuild` takes several minutes even after Payara is up -- the playbook
is waiting for Dataverse to finish its own initialization before it can call the API.

### Common Payara problems

- **Payara not responding after deploy**: Dataverse startup takes 2-5 minutes on first boot.
  Check the Payara log at `/usr/local/payara5/glassfish/domains/domain1/logs/server.log`.
  Look for `Dataverse started` or exception stack traces.
- **Out of memory**: Payara JVM heap settings are configured in Ansible group_vars.
  Default is 2GB -- increase if you see `OutOfMemoryError` in the Payara log.
- **WAR deploy failure**: The WAR file checksum or version may not match what Payara expects.
  Check the Ansible `payara.yml` task output for errors during the deploy step.

::::::::::::::::::::::::::::::::::::: callout

### Payara vs. GlassFish

Dataverse documentation and older issues sometimes reference GlassFish commands and
paths. Payara is API-compatible -- the `asadmin` command and most paths are the same.
If you find a GlassFish-specific workaround in an older issue, it will almost always
apply to Payara as well.

::::::::::::::::::::::::::::::::::::::::::::::::

## The data layer: RDS, S3, and Solr

Dataverse splits its data across three storage systems, each handling a different type:

### RDS (PostgreSQL)

RDS stores all structured metadata:

- Dataset records (titles, descriptions, authors, versions, publication dates)
- File records (names, checksums, file type, which dataset they belong to)
- User accounts and permissions
- Dataverse collection hierarchy
- Workflow state and notifications

RDS is the authoritative record for what datasets and files exist.
When you restore a database backup, you are restoring this metadata.

### S3

S3 stores the actual file content -- the bytes of every data file users have uploaded.
The connection between a file record in RDS and its content in S3 is a storage identifier
stored in the database.

After a database restore, Dataverse uses the storage identifiers to serve files from S3.
The files in S3 do not change when you restore the database -- only the metadata does.

This also means that if a file is deleted from S3 but its record remains in RDS, Dataverse
will show the file as available but fail when users try to download it.

::::::::::::::::::::::::::::::::::::: callout

### The baseline scripts check both

The `make baseline` command captures both database counts (via the Dataverse Metrics API)
and S3 object counts (via `aws s3 ls`). Comparing pre- and post-migration baselines
verifies that both the database and the storage bucket survived the migration intact.

::::::::::::::::::::::::::::::::::::::::::::::::

### Solr

Solr is a full-text search engine. Dataverse uses it to power all search and browse
functionality -- when you type a query in Dataverse or browse a collection, the results
come from Solr, not directly from the database.

Solr maintains its own index: a data structure optimized for fast search that is built
from the content in the database. This index is separate from and derived from the database.

**The index can go out of sync.** If you restore the database to an earlier state, the Solr
index may contain entries for datasets that no longer exist, or be missing entries for
datasets that were added. After any database restore, you must reindex:

```bash
make reindex ENV=tim
```

A Dataverse instance with a stale Solr index will appear to have no datasets -- or the
wrong datasets. This is one of the most common sources of confusion after a rebuild.

The reindex process reads all dataset metadata from the database and sends it to Solr.
Depending on how many datasets you have, this can take from seconds to hours.

::::::::::::::::::::::::::::::::::::: challenge

### Trace a file access

Given what you know about the data layer, answer these questions:

1. A user uploads a file to Dataverse. What gets written to RDS? What gets written to S3?
2. A user searches for "climate data." Which storage system answers that query?
3. You restore the database from a week-old backup. What is the state of S3? What is the state of Solr?
4. After the restore in question 3, what must you do before the system is usable again?

:::::::::::::::::::::::::::::::::: solution

1. RDS gets a new file record (name, checksum, storage identifier, dataset reference). S3 gets the file bytes at the storage identifier path.
2. Solr answers the search query. The database is not involved in search.
3. S3 is unchanged -- it still has all files ever uploaded, including those added in the past week. Solr still has the current index built from the current database before the restore.
4. Run `make reindex ENV=<env>` to rebuild the Solr index from the restored database state.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- Payara is a Jakarta EE application server; Dataverse runs as a WAR file inside it.
- Most Dataverse configuration is set as Payara JVM options, managed by Ansible.
- The Payara log at `glassfish/domains/domain1/logs/server.log` is the first place to look when things go wrong.
- RDS holds metadata; S3 holds file content; Solr holds the search index.
- Always run `make reindex` after a database restore -- Solr does not update itself.

::::::::::::::::::::::::::::::::::::::::::::::::
