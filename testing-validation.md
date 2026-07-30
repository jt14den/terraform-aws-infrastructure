---
title: "Testing and Validation"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- How do we know the migration worked?
- What does the test suite check?
- How do baseline comparisons verify data integrity?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Run the pytest suite and read its output.
- Explain what each test category checks.
- Compare two baseline snapshots and interpret the diff.
- Know what must pass before each migration phase can proceed.

::::::::::::::::::::::::::::::::::::::::::::::::

## Why testing matters here

For a database migration, "did it work?" is not obvious. Dataverse can appear to start
successfully while silently missing datasets, serving stale search results, or failing
on file downloads.

The test suite and baseline comparisons are the answer to that question. Together they check:

- The application is running and accepting requests
- The database has the expected number of datasets, files, and users
- S3 has the expected number of objects
- Solr is indexed and returning results
- File upload and download work end to end

No migration phase is complete until these pass.

## The pytest suite

Tests live in `dataverse-ansible/tests/integration/` -- the child repo, not the
orchestration repo. There's no `tests/` directory in `dataverse-infrastructure` itself.

Run the full suite:

```bash
make test ENV=tim
```

That target expands to, roughly:

```bash
cd dataverse-ansible && uv run pytest tests/integration -v --dataverse-url=https://<hostname>
```

Note `uv`, not a bare `pytest` -- and the `--dataverse-url` flag is required, since these
tests hit a live instance over the network rather than running against localhost.

### Test categories (real classes, from `test_smoke.py`)

**`TestAPIHealth`**: Check that the Dataverse API responds on expected endpoints.
These are the fastest tests and fail first if Payara is down or misconfigured.

**`TestS3Integration`**: Verify that Dataverse can read from and write to S3.
A test file is uploaded via the API and then downloaded. If the IAM instance profile or
bucket config is wrong, this fails.

**`TestSolrIndexing`**: Check that Solr is running and the index is not empty.
An empty index does not cause Dataverse to error -- it just silently returns no search results.
This test catches that.

**`TestSearch`, `TestDataverse`, `TestDatasets`, `TestWebInterface`, `TestAuthentication`**:
round out the smoke suite -- basic CRUD, page rendering, and login checks.

**Baseline count comparison** is a separate step (`make baseline-compare`), not a pytest
class -- it compares two JSON snapshots, not live API responses.

**PID/DOI checks exist, but are narrower than they sound.** `test_migration.py` has a
`TestPIDConfiguration` class, but it only runs with the `migration` pytest marker
(`make ansible-migration`, not the default `make test`), and it mostly checks that PID
*settings exist* in the database -- it doesn't distinguish FAKE from a real EZID
connection. A dedicated DOI/FAKE-provider validation test is planned (roadmap `03-03`)
but not yet built.

### Reading test output

```
PASSED tests/test_api.py::test_dataverse_api_version
PASSED tests/test_s3.py::test_s3_upload_download
FAILED tests/test_solr.py::test_solr_index_not_empty
  AssertionError: Solr returned 0 results -- did you run make reindex?
```

Any `FAILED` test is a blocker. Fix the underlying problem before proceeding.
The error message usually tells you what to do.

## Baseline comparisons

### Capturing baselines

Run `make baseline` before and after any significant operation:

```bash
# before migration
make baseline ENV=jamie
mv baseline-snapshots/baseline_<timestamp>.json baseline-snapshots/pre-migration.json

# ... do migration work ...

# after migration
make baseline ENV=jamie
mv baseline-snapshots/baseline_<timestamp>.json baseline-snapshots/post-migration.json
```

There's no `latest.json` -- every capture gets its own timestamped filename, so renaming
(or tracking the filename `make baseline` prints) is how you keep pre/post straight.

### Comparing

```bash
make baseline-compare BEFORE=baseline-snapshots/pre-migration.json AFTER=baseline-snapshots/post-migration.json
```

A clean comparison looks like:

```
datasets:    1,247  ->  1,247  OK
files:      18,903  -> 18,903  OK
users:         142  ->    142  OK
s3_objects: 18,903  -> 18,903  OK
s3_bytes:   84.2GB  ->  84.2GB OK
```

Any mismatch is a problem to investigate -- **except one field.** `downloads`/guestbook
history is deliberately treated as informational-only in `baseline-compare.sh`, not a
failure: download counts legitimately keep incrementing as long as the instance is live,
so a "drift" there doesn't mean data was lost. Datasets and files are the fields that
must match exactly. Common causes of a real mismatch:

- Solr not yet reindexed (run `make reindex`, then re-run the comparison)
- A dataset was published or retracted between captures (check the timing)
- Files missing from S3 (check S3 directly with `aws s3 ls`)
- Database restore captured different data than expected (verify backup timestamp)

::::::::::::::::::::::::::::::::::::: callout

### The Phase 7 comparison is the final gate

The baseline captured by Jamie before any migration work (run with `BASELINE_UPLOAD_BUCKET` set)
is compared against the post-cutover production environment. If these do not match,
the migration is not complete. Do not finalize the cutover until the comparison passes.

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### Interpret a failing test run

Imagine you run `make test ENV=tim` and see:

```
PASSED tests/test_api.py::test_dataverse_api_version
FAILED tests/test_s3.py::test_s3_upload_download
  ConnectionError: Could not connect to S3 endpoint
PASSED tests/test_solr.py::test_solr_index_not_empty
```

1. What does this tell you about the state of the environment?
2. What are the two most likely causes of the S3 failure?
3. What would you check first?

:::::::::::::::::::::::::::::::::: solution

The API and Solr are working, so Payara is up and Solr is indexed.
S3 connectivity is the problem -- Dataverse cannot reach S3.

Two most likely causes:

1. The S3 bucket name is wrong in the JVM options, or the IAM instance profile attached
   to the EC2 instance doesn't grant the right permissions on that bucket (check `group_vars`
   for the bucket name, and the Terraform IAM role/policy for permissions -- there are no
   AWS access keys to check, since this uses an instance profile, not vaulted credentials)
2. The security group does not allow outbound HTTPS to S3 (check the Terraform security group config)

Start with the bucket name and IAM policy -- they're the most common source of S3 config
problems. Check the Payara log for `AmazonS3Exception` or `AccessDenied` messages.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

### The one field that's allowed to drift

Without checking the episode: a `baseline-compare` run shows `datasets: 1,247 -> 1,247 OK`,
`files: 18,903 -> 18,903 OK`, but `downloads: 3,401 -> 3,512`. Is this a failure? Why or why not?

:::::::::::::::::::::::::::::::::::: solution

Not a failure. `downloads`/guestbook history is treated as informational-only by
`baseline-compare.sh` -- download counts naturally keep incrementing while an instance
is live and serving traffic, so a difference there reflects normal usage, not lost or
corrupted data. Datasets and files matching exactly is what actually gates a migration phase.

::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- The test suite (`dataverse-ansible/tests/integration/`, run via `make test`) covers API health, search, S3 round-trip, and Solr indexing as real pytest classes.
- PID/DOI-specific validation is thin today: a `TestPIDConfiguration` class exists but only runs under the `migration` marker and mostly checks settings exist, not FAKE-vs-EZID behavior.
- Baseline comparisons verify data integrity by comparing counts before and after migration, saved to `baseline-snapshots/` (no `latest.json`).
- Every baseline field must match exactly **except** `downloads`, which is deliberately informational-only.
- Jamie's pre-migration production baseline is the anchor for the final Phase 7 comparison.

::::::::::::::::::::::::::::::::::::::::::::::::
