---
name: observability-fds-dataset-setup
description: >
  Register an ES|QL Data Federation (FDS) dataset backed by object storage. When a Fleet integration package policy
  exists, reads its input vars (bucket, region, prefix) to pre-fill registration bodies. When no policy exists, collects
  the same details from the user directly. Use when the user wants to register an FDS dataset, set up FDS, query
  object-storage data beyond Elasticsearch retention, or explore cloud logs without a full ingestion pipeline.
metadata:
  author: es-analytical-engine
  version: 0.5.0
  universal: true
compatibility: >
  Elasticsearch 9.5+ with ES|QL Data Federation enabled (esql.federation.enabled: true), Enterprise license,
  self-managed or Elastic Cloud Hosted (not Serverless). Kibana 9.5+ with xpack.dataFederation.enabled: true.
  Cluster-state encryption must be configured before storing credentials in a data source. Requires the `elastic` CLI
  with `es esql` and `kb` support.
---

# FDS Dataset Setup

Register an ES|QL Data Federation (FDS) dataset by discovering or collecting S3 configuration, building the data source
and dataset registration bodies, executing them, and verifying that data is queryable. This skill produces a registered
dataset. It does not build dashboards or charts, and it does not assess whether the Elasticsearch index already covers
the user's needs — use the **elasticsearch-esql** skill for index-backed queries.

For building dashboards on a registered FDS dataset, use the **kibana-fds-dashboards** skill after this skill confirms
that data is queryable.

<!-- begin-partial: preamble -->

## Environment Configuration

This skill executes Elasticsearch operations through the `elastic` CLI. If the
[`elastic` CLI](https://github.com/elastic/cli#configuration) is not installed, tell the user what it is needed for. Do
not guess credentials, call the HTTP API directly, or attempt other workarounds.

This skill references operations in HTTP-shorthand form (e.g., `GET /`, `GET /_cat/indices`, `GET /{index}/_mapping`,
`GET /{index}/_settings/index.mode`, `POST /_query`). The [Operations](#operations) table at the end of this document
maps each shorthand to the equivalent `elastic` CLI command — always use the CLI rather than calling the HTTP API
directly.

<!-- end-partial: preamble -->

## Prerequisites

- **ES|QL Data Federation enabled.** `esql.federation.enabled: true` on every Elasticsearch node (static setting;
  restart required). `xpack.dataFederation.enabled: true` in Kibana for the UI. Call `GET /` and check the version is
  9.5+. If the feature is disabled, `GET /_query/data_source` returns a feature-not-enabled error — report it and stop.
- **Cluster-state encryption configured.** A data source with credentials (static or federated identity) requires
  encryption at rest. If missing, the `PUT /_query/data_source` call fails with a clear error. Report it and stop.
- **Enterprise license.** Data Federation requires an Enterprise license or a trial. If the license check fails, report
  it and stop.
- **S3 data source only.** This skill registers `type: "s3"` data sources. GCS and Azure Blob Storage require different
  data source types not covered here.
- **Fleet integration (optional).** A Fleet package policy with an `aws-s3` input stream provides auto-discovery of
  bucket, region, and prefix. Without one, the user must supply these details manually (the skill handles both paths).

## Process

### 0. Check what already exists

1. **Check for existing FDS datasets.** Call `GET /_query/dataset` and `GET /_query/data_source`. If a dataset already
   covers the same bucket and prefix, report the existing dataset name. Ask if the user wants to probe it (jump to
   Step 7) or register a new one with different settings.

2. **Check for a Fleet integration policy.** Call `GET kbn:/api/fleet/package_policies`. Filter for policies with an
   input stream whose `data_stream.dataset` matches the user's intent (e.g. `aws.vpcflow`, `aws.elb_logs`). If a
   policy exists, proceed to Step 1. If no policy is found, collect S3 bucket name, region, object prefix, file format,
   and `@timestamp` column details from the user directly and skip to Step 3.

### 1. Verify connectivity and feature status

Call `GET /` to confirm the cluster is reachable and version is 9.5+. Call `GET kbn:/api/status` to confirm Kibana is
reachable. If either fails, stop and surface the error.

### 2. Extract S3 details from the policy

Integration policy vars live at **two different levels** in the Fleet API response. Extract from the correct level:

**Stream-level vars** — inside `inputs[].streams[].vars` where the stream matches:

| Stream var           | What it gives us          | Often missing?                                    |
| -------------------- | ------------------------- | ------------------------------------------------- |
| `collect_s3_logs`    | Whether S3 mode is active | Always present (default `false`)                  |
| `bucket_arn`         | S3 bucket ARN             | Yes — SQS default has no bucket                   |
| `access_point_arn`   | S3 Access Point ARN       | Yes — alternative to `bucket_arn`                 |
| `bucket_list_prefix` | Object prefix             | Yes — often empty                                 |
| `region`             | AWS region                | Yes — defaults to empty (SDK default `us-east-1`) |
| `start_timestamp`    | Agent ingest lower bound  | Yes — only set when the user configured it        |
| `ignore_older`       | Agent ingest age cutoff   | Yes — duration like `48h`; alternative to above   |

**Policy-level vars** — in the top-level `vars` of the policy object:

| Policy var       | What it gives us        | Often missing?          |
| ---------------- | ----------------------- | ----------------------- |
| `role_arn`       | IAM role for FDS access | Yes — not always set    |
| `default_region` | Fallback AWS region     | Yes — defaults to empty |

`role_arn` and credentials are **not** in the stream vars — they are package-level.

**If `collect_s3_logs` is `false` or both `bucket_arn` and `access_point_arn` are empty**, the policy is SQS-only or
CloudWatch-only. The default for all AWS integration data streams is `collect_s3_logs: false`. Tell the user and ask
them to supply:

- The S3 bucket name (e.g. `my-vpc-flow-logs`)
- The AWS region (e.g. `us-east-1`)
- The object prefix (e.g. `AWSLogs/123456789012/vpcflowlogs/us-east-1/`)
- A `role_arn` the FDS data source can assume, or confirm anonymous/static credentials

**Parse the bucket name from `bucket_arn`.** The ARN format is `arn:aws:s3:::<bucket-name>`. Extract the bucket name
after the last `:::` — e.g. `arn:aws:s3:::my-vpc-flow-logs` → `my-vpc-flow-logs`.

**If the policy uses `access_point_arn` instead of `bucket_arn`**, the access point ARN format is
`arn:aws:s3:<region>:<account>:accesspoint/<name>`. An access point does not expose the underlying bucket name. Ask the
user for the actual S3 bucket name that the access point routes to.

**If `region` is empty**, check `default_region` at the policy level. If both are empty, try to infer the region from
`bucket_list_prefix` — hive-style prefixes often contain `aws-region=<region>` (e.g.
`AWSLogs/aws-account-id=.../aws-service=.../aws-region=us-east-1/`). If the prefix does not contain a region, the SDK
default is `us-east-1` but FDS needs an explicit region — ask the user to confirm.

**Access keys are Fleet secret references** (returned as `::es_redacted::`). Do not ask for them. Ask for a `role_arn`
or direct the user to use anonymous access for public buckets.

**Standalone packages** (`aws_logs`, `aws_billing`, `amazon_security_lake`) do not use the `collect_s3_logs` toggle.
They have `bucket_arn` directly as a stream var. If it is present and non-empty, use it.

### 3. Determine layout and format

The policy does not record the object layout or file format. Check the `references/` directory for a file matching the
integration (e.g. `aws-vpcflow.md` for VPC Flow Logs, `aws-elb.md` for ELB access logs). If a matching reference file
exists, read it for the known layout, format, `@timestamp` mapping, and partition spec. If no reference file exists for
this integration, ask the user for the details below.

**Always verify the prefix from the Fleet policy against the confirmed layout.** The `bucket_list_prefix` from the
policy is ground truth — use it as the base for the resource glob. Append the suffix that matches that layout. Do not
substitute a reference path if the policy prefix differs.

- **Hive** — `partition_detection: "hive"` and a `partition_spec` aligned with `@timestamp`. A VPC Flow path appends
  `year=*/month=*/day=*/hour=*/*.parquet`.
- **Date-prefix or flat** — no `partition_detection` override and no `partition_spec`. The glob must cover the real
  directories. An ELB access-log path is `YYYY/MM/DD/` and appends something like `*/*/*/*.log.gz`. Queries scan every
  file the glob matches.

Ask the user to confirm or override:

- **File format.** Parquet, NDJSON, CSV, or TSV. On Elasticsearch 9.6+, `settings.format` set to `"csv"` or `"tsv"`
  forces that reader on files whose extension is not recognised, including `.log` and `.log.gz`. Verify with a test
  query. On Elasticsearch before 9.6, plain text (`.log`, `.txt`, `.log.gz`) is not supported — stop. Gzip-compressed
  structured files (`.json.gz`, `.csv.gz`) may work when the underlying format is supported — verify with a test query
  before committing.
- **CSV and TSV settings.** Confirm `delimiter` (default `,` for CSV, tab for TSV), `mode` (`quoted` when a field can
  contain the delimiter inside quotes, otherwise `plain`), `header_row` (`true` when the first row names the columns,
  `false` for headerless files), and `null_value` when a token should read as null. Space-delimited logs use
  `"delimiter": " "` together with `"format": "csv"`.
- **Object layout.** Hive, date-prefix, or flat, as above. Confirm which one the files use before writing the glob.
- **`@timestamp` mapping.** The source column name, type, and format vary by integration (e.g. VPC Flow uses `start` as
  epoch_second; CloudTrail uses `eventTime` as ISO 8601). Without a reference file, the user must supply these.
- **`partition_spec`.** For a hive layout, it must align with the `@timestamp` mapping
  (`year(@timestamp), month(@timestamp), day(@timestamp), hour(@timestamp)`). For a plain date-prefix or flat layout, do
  not set `partition_spec`.

### 4. Build and confirm the data source body

Pre-fill the `PUT /_query/data_source/<name>` body:

```json
{
  "type": "s3",
  "description": "<integration> logs via FDS",
  "settings": {
    "region": "<region>",
    "auth": "federated_identity",
    "role_arn": "<role_arn>"
  }
}
```

For anonymous access (public buckets), use `"auth": "anonymous"` and omit `role_arn`. For static credentials, use
`"auth": "static_credentials"` with `access_key` and `secret_key` — but prefer `federated_identity` when available.

Propose a data source name derived from the integration (e.g. `aws_vpcflow_s3`). Show the body. Wait for the user to
confirm or edit.

### 5. Build and confirm the dataset body

Pre-fill the `PUT /_query/dataset/<name>` body using the integration reference. Build the `resource` glob from the
policy's `bucket_list_prefix` (not the reference layout) plus the suffix from that reference. The template below
is the hive Parquet shape. Use the CSV template after it for delimited text, including a plain date-prefix layout.

```json
{
  "data_source": "<data_source_name>",
  "resource": "s3://<bucket>/<bucket_list_prefix>/year=*/month=*/day=*/hour=*/*.parquet",
  "settings": {
    "schema_resolution": "first_file_wins",
    "partition_detection": "hive",
    "partition_spec": "year(@timestamp), month(@timestamp), day(@timestamp), hour(@timestamp)"
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date",
        "path": "start",
        "format": "epoch_second"
      }
    }
  }
}
```

The `@timestamp` mapping (`path`, `format`) and `partition_spec` come from the integration reference file. They are
**not** in the Fleet policy. Without this mapping, `@timestamp` is not bound, the time picker does not work, and
`partition_spec` cannot prune files. Omit `partition_detection` and `partition_spec` for a plain date-prefix or flat
layout — pruning is not available there.

**CSV or TSV.** Use this shape for delimited text. Set `delimiter`, `mode`, `header_row`, and the glob from what the
user confirmed. `","` and `"\t"` are the usual separators. `" "` is for space-delimited text. Use `"quoted"` only when a
field can contain the delimiter inside quotes, otherwise `"plain"`. On Elasticsearch 9.6+, `format` also selects the
reader for an unrecognised extension such as `.log.gz`. That is one case, not the default template.

```json
{
  "data_source": "<data_source_name>",
  "resource": "s3://<bucket>/<prefix>/<layout-glob>",
  "settings": {
    "format": "<csv_or_tsv>",
    "delimiter": "<delimiter>",
    "header_row": true,
    "mode": "<quoted_or_plain>",
    "schema_resolution": "first_file_wins"
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date",
        "path": "<timestamp_column>"
      }
    }
  }
}
```

**Headerless files.** When `header_row` is `false`, columns are positional (`col0`, `col1`, ...). If the user has
already confirmed a known log type and a matching reference file exists in `references/`, use the full `properties`
mapping from that reference directly — no fingerprinting or validation needed.

Otherwise, read [references/csv-fingerprints.md](references/csv-fingerprints.md) to attempt structural fingerprinting.
If fingerprinting does not match, it chains to
[references/column-identification.md](references/column-identification.md) which presents the user with three paths
(name the log type, provide mapping directly, or identify from sample data) and validates all non-reference mappings
via a temp dataset before final registration. Do not invent field names.

Propose a dataset name derived from the integration (e.g. `vpc_flow`). Show the body. Wait for the user to confirm or
edit.

### 6. Execute registration

Register the data source and dataset using the native ES|QL CLI commands.

**Create the data source.** Write the data source body from Step 4 to a JSON file and execute:

`PUT /_query/data_source/<data_source_name>` with `--input-file <path>`.

The call may succeed even with an invalid bucket or wrong credentials — registration is lazy. Cluster-state encryption
errors do fail immediately. S3 access errors surface at query time in Step 7. Show any error and ask the user to fix.

**Create the dataset.** Write the dataset body from Step 5 to a JSON file and execute:

`PUT /_query/dataset/<dataset_name>` with `--input-file <path>`.

Common failures: data source does not exist (register it first), resource glob matches no files, `partition_spec` syntax
error. Show the error and ask the user to fix.

**Verify both registrations.** Call `GET /_query/data_source/<data_source_name>` and
`GET /_query/dataset/<dataset_name>` to confirm the definitions were stored correctly.

### 7. Probe for data and hand off

Run `POST /_query` with a bounded time window. Choose the window size based on the dataset's layout:

- **Hive with `partition_spec`:** use the finest partition grain (e.g. one hour for `hour(@timestamp)`, one day for
  `day(@timestamp)`).
- **Date-prefix:** use one day (matches the typical `YYYY/MM/DD/` directory structure).
- **Flat (no partitions):** use one day as a reasonable default.

Start from the current time (UTC) and step backward:

```esql
FROM <dataset>
| WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>")
| LIMIT 1
```

If 0 rows, step backward by one window. Try at most 3 windows. If all return 0 rows, ask the user for a time range
that has data. Do not widen the window beyond the chosen grain — that scans more files.

Once `POST /_query` returns rows, report the dataset name and a confirmed time window with data. Hand off to
**kibana-fds-dashboards** for dashboards or **elasticsearch-esql** for queries.

## Edge Cases

- **Unrecognised extension:** On Elasticsearch before 9.6, plain text (`.log`, `.txt`, `.log.gz`) is not supported.
  Report and stop. On 9.6+, set `settings.format` to `"csv"` or `"tsv"` to force that reader, then verify with a test
  query. A failed test means stop.
- **Compressed structured files:** `.json.gz` or `.csv.gz` may work. Verify with a test query before committing.
- **GCS or Azure data:** This skill covers S3 only. GCS and Azure Blob require different `type` values in the data
  source body. Report the limitation and stop.
- **Dataset name conflict:** `PUT /_query/dataset` fails if the name collides with an index, alias, or view. Suggest a
  different name.
- **Prefix layout mismatch:** The policy prefix may differ from the reference (e.g. hive-style `aws-account-id=...`
  vs classic `AWSLogs/<account>/...`). Always use the policy prefix.

## Examples

**"Set up an FDS dataset from my aws.vpcflow integration."** — Fleet policy exists.

1. `GET /_query/dataset` → no VPC flow dataset.
2. `GET kbn:/api/fleet/package_policies` → policy found with `aws-s3` input and `aws.vpcflow` stream.
3. Extract from policy: `bucket_arn: arn:aws:s3:::my-vpc-flow-logs`, `region: us-east-1`,
   `bucket_list_prefix: AWSLogs/...`. From policy-level vars: `role_arn: arn:aws:iam::123456789012:role/fds-reader`.
4. Read `references/aws-vpcflow.md` for `@timestamp` mapping and hive layout.
5. Confirm data source and dataset bodies with the user.
6. Register via `PUT /_query/data_source/<name>` and `PUT /_query/dataset/<name>`.
7. Probe with `FROM vpc_flow | WHERE @timestamp >= ... | LIMIT 1`.
8. Hand off to **kibana-fds-dashboards**.

**"Register an FDS dataset for my VPC flow logs."** — Dataset already exists.

1. `GET /_query/dataset` → `vpc_flow_ui` already registered with a resource glob covering `year=*/month=*`.
2. Report: "An FDS dataset `vpc_flow_ui` already exists for this data." Ask the user: probe it, or register a new one?
3. User says probe. Run `FROM vpc_flow_ui | WHERE @timestamp >= TO_DATETIME("2026-10-21T09:00:00.000Z") AND @timestamp < TO_DATETIME("2026-10-21T10:00:00.000Z") | LIMIT 1` → returns rows.
4. Report the dataset name and confirmed time window.

**"I want to explore my CloudTrail logs in S3 but there's no integration set up yet."** — No Fleet policy.

1. `GET /_query/dataset` → no CloudTrail dataset.
2. `GET kbn:/api/fleet/package_policies` → no matching CloudTrail policy found.
3. Collect from user: S3 bucket name, AWS region, object prefix (e.g. `AWSLogs/<account>/CloudTrail/<region>/`), file
   format (`.json.gz` NDJSON), and `@timestamp` column (CloudTrail uses `eventTime`, ISO 8601).
4. Confirm layout is date-prefix (`<YYYY>/<MM>/<DD>/`), not hive.
5. Build data source and dataset bodies. No `partition_spec` for date-prefix layout.
6. Register, probe, and hand off.

## Guidelines

1. **Do not invent S3 details.** Every bucket, region, prefix, and credential must come from the Fleet policy or be
   confirmed by the user.
2. **Do not skip the `@timestamp` mapping.** Without it, the time picker does not bind, `partition_spec` does not prune,
   and queries scan all files.
3. **Do not register a dataset on an unsupported format.** Data Federation supports Parquet, NDJSON, CSV, and TSV. On
   Elasticsearch before 9.6, plain text (`.log`, `.txt`, `.log.gz`) is not supported. On 9.6+, `.log` and `.log.gz` work
   only when `settings.format` is `"csv"` or `"tsv"` and a test query succeeds. Gzip-compressed structured files
   (`.json.gz`, `.csv.gz`) may work — verify with a test query before committing the dataset.
4. **Do not guess column names or types.** The integration reference provides known columns. Anything else must come
   from the probe sample in Step 7.
5. **Show bodies and wait for confirmation.** Both the data source and dataset bodies contain credentials and access
   details. The user must review them before execution.
6. **One data source per credential set, one dataset per file collection.** Multiple datasets can share a data source if
   they use the same credentials and region.
7. **Distinguish agent ingest settings from FDS scope.** `start_timestamp` and `ignore_older` are agent filters — they
   do not affect FDS queries. Do not tell the user to change the integration when FDS already covers their time range.
8. **Use the Fleet policy prefix, not the reference template.** The `bucket_list_prefix` from the policy is the actual
   S3 path. The reference file documents common layouts but cannot predict every account's prefix structure.

## Operations

| HTTP API (shorthand)                  | `elastic` CLI command                                                 |
| ------------------------------------- | --------------------------------------------------------------------- |
| `GET /`                               | `elastic es info`                                                     |
| `GET kbn:/api/status`                 | `elastic kb system get-status`                                        |
| `GET kbn:/api/fleet/package_policies` | `elastic kb fleet-package-policies get-fleet-package-policies`        |
| `GET /_query/data_source`             | `elastic es esql get-data-source`                                     |
| `GET /_query/dataset`                 | `elastic es esql get-dataset`                                         |
| `PUT /_query/data_source/<name>`      | `elastic es esql put-data-source --name '<name>' --input-file <path>` |
| `PUT /_query/dataset/<name>`          | `elastic es esql put-dataset --name '<name>' --input-file <path>`     |
| `GET /_query/data_source/<name>`      | `elastic es esql get-data-source --name '<name>'`                     |
| `GET /_query/dataset/<name>`          | `elastic es esql get-dataset --name '<name>'`                         |
| `POST /_query`                        | `elastic es esql query --format tsv --query "<esql>"`                 |
