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

Tests live in `dataverse-infrastructure/tests/`. They use pytest and the Dataverse API.

Run the full suite:

```bash
make test ENV=tim
```

Or run it directly with more output:

```bash
cd tests
pytest -v
```

### Test categories

**API smoke tests**: Check that the Dataverse API responds on expected endpoints.
These are the fastest tests and fail first if Payara is down or misconfigured.

**S3 connectivity**: Verify that Dataverse can read from and write to S3.
A test file is uploaded via the API and then downloaded. If S3 credentials or bucket
config is wrong, this fails.

**Solr health**: Check that Solr is running and the index is not empty.
An empty index does not cause Dataverse to error -- it just silently returns no search results.
This test catches that.

**Baseline count comparison**: If `BEFORE` and `AFTER` baseline files are set,
verify that dataset and file counts match within an acceptable threshold.

**DOI/FAKE validation**: Verify that the configured PID provider works.
In test environments, this checks that the FAKE provider mints identifiers correctly.
In production, this would check EZID connectivity.

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
mv baselines/latest.json baselines/pre-migration.json

# ... do migration work ...

# after migration
make baseline ENV=jamie
mv baselines/latest.json baselines/post-migration.json
```

### Comparing

```bash
make baseline-compare BEFORE=baselines/pre-migration.json AFTER=baselines/post-migration.json
```

A clean comparison looks like:

```
datasets:    1,247  ->  1,247  OK
files:      18,903  -> 18,903  OK
users:         142  ->    142  OK
s3_objects: 18,903  -> 18,903  OK
s3_bytes:   84.2GB  ->  84.2GB OK
```

Any mismatch is a problem to investigate. Common causes:

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

1. The S3 IAM credentials or bucket name are wrong in the JVM options (check group_vars and verify with `ansible-vault view`)
2. The security group or IAM role does not allow outbound HTTPS to S3 (check the Terraform security group config)

Start with the JVM options -- they are the most common source of S3 config problems.
Check the Payara log for `AmazonS3Exception` or credential error messages.

::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

- The test suite covers API smoke, S3 round-trip, Solr health, and baseline count comparisons.
- All tests must pass before a migration phase is complete.
- Baseline comparisons verify data integrity by comparing counts before and after migration.
- A mismatch in baseline comparison is a blocker -- investigate before proceeding to the next phase.
- Jamie's pre-migration production baseline is the anchor for the final Phase 7 comparison.

::::::::::::::::::::::::::::::::::::::::::::::::
