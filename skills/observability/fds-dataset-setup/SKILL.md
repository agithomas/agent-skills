---
name: observability-fds-dataset-setup
description: >
  Register an ES|QL Data Federation (FDS) dataset backed by S3 object storage. When a Fleet integration package policy
  exists, reads its aws-s3 input vars (bucket, region, prefix) to pre-fill registration bodies. When no policy exists,
  collects the same details from the user directly. Executes data source and dataset registration via the Elasticsearch
  ES|QL dataset API. Use when the user wants to query S3 data beyond Elasticsearch retention, explore cloud logs
  without a full ingestion pipeline, or set up an FDS dataset. Not for index-backed data (use elasticsearch-esql) or
  GCS/Azure storage (different data source types).
metadata:
  author: es-analytical-engine
  version: 0.4.1
  universal: true
compatibility: >
  Elasticsearch 9.5+ with ES|QL Data Federation enabled (esql.federation.enabled: true), Enterprise license,
  self-managed or Elastic Cloud Hosted (not Serverless). Kibana 9.5+ with xpack.dataFederation.enabled: true.
  Cluster-state encryption must be configured before storing credentials in a data source. Requires the `elastic` CLI
  with `es esql` and `kb` support.
---

# FDS Dataset Setup from Fleet Integration

Register an ES|QL Data Federation dataset by reading an existing Fleet integration package policy for its S3
configuration, building the `_query/data_source` and `_query/dataset` registration bodies, executing them, and verifying
that data is queryable. This skill produces a registered dataset. It does not build dashboards or charts.

For building dashboards on a registered FDS dataset, use the **kibana-fds-dashboards** skill after this skill confirms
that data is queryable. For writing ES|QL queries against an FDS dataset or any Elasticsearch index, use the
**elasticsearch-esql** skill.

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

### 0. Assess the user's goal and existing datasets

Before registering anything:

1. **Check for existing FDS datasets.** Call `GET /_query/dataset` and `GET /_query/data_source`. If a dataset already
   covers the same bucket and prefix, skip registration — tell the user and jump to Step 8.

2. **Surface ingest boundaries.** Read `start_timestamp` and `ignore_older` from the stream vars (Step 3). These limit
   what the agent ingests into `logs-*` indices — they do **not** affect FDS. If the user's time range falls outside
   the agent's ingest window, present the options:

   | Option | When to choose |
   | ------ | -------------- |
   | **FDS dataset** | Exploration, dashboards, ad-hoc analysis. Immediate access, no ingestion cost. |
   | **Adjust `start_timestamp`** | Alerting, ML, or index-backed features that require ingested data. |
   | **Both** | Immediate FDS access plus ongoing alerting/ML after re-ingestion. |

   This skill does not modify Fleet policies. If the user only needs to adjust the integration, advise and stop.

### 1. Verify connectivity and feature status

Call `GET /` to confirm the cluster is reachable and version is 9.5+. Call `GET kbn:/api/status` to confirm Kibana is
reachable. If either fails, stop and surface the error.

### 2. Identify the integration

Call `GET kbn:/api/fleet/package_policies`. Filter for policies that have an `inputs[]` entry with `type == "aws-s3"`,
then within that input look for `streams[]` whose `data_stream.dataset` matches the user's intent (e.g. the user says
"VPC flow" → look for `aws.vpcflow`). Show matching policies and ask the user to pick one if more than one exists.

**If no matching policy exists**, do not stop. Ask the user to supply the S3 bucket name, AWS region, object prefix,
file format, and `@timestamp` column details directly, then skip to Step 4 (determine layout and format).

### 3. Extract S3 details from the policy

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

### 4. Determine layout and format

The policy does not record the object layout or file format. Read the integration reference file for the known layout,
format, `@timestamp` mapping, and partition spec. For AWS VPC Flow Logs, read
[references/aws-vpcflow.md](references/aws-vpcflow.md). For Application, Network, and Classic Load Balancer access logs,
read [references/aws-elb.md](references/aws-elb.md). For other integrations, no reference file exists yet — ask the
user for the details below.

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

### 5. Build and confirm the data source body

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

### 6. Build and confirm the dataset body

Pre-fill the `PUT /_query/dataset/<name>` body using the integration reference. Build the `resource` glob from the
Fleet policy's `bucket_list_prefix` (not the reference layout) plus the suffix from that reference. The template below
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

**Headerless files.** When `header_row` is `false`, columns are positional (`col0`, `col1`, ...). Ask the user whether
they want named columns. If yes, ask them to provide the field names in order, or to point at that source's official
documentation. If they provide names, set `"dynamic": false` and map every column with `"path": "colN"`, including
`@timestamp`. Type a field as `keyword` when the sample shows a non-numeric token in an otherwise numeric column. Type
counts and byte sizes that are always numeric as `long`. If the user declines, leave `dynamic` at its default and tell
them the columns stay `col0`, `col1`, and so on. Do not invent field names. Open
[references/aws-elb.md](references/aws-elb.md) only after the user confirms the source is an AWS load balancer access
log.

Propose a dataset name derived from the integration (e.g. `vpc_flow`). Show the body. Wait for the user to confirm or
edit.

### 7. Execute registration

Register the data source and dataset using the native ES|QL CLI commands.

**Create the data source.** Write the data source body from Step 5 to a JSON file and execute:

`PUT /_query/data_source/<data_source_name>` with `--input-file <path>`.

The call may succeed even with an invalid bucket or wrong credentials — registration is lazy. Cluster-state encryption
errors do fail immediately. S3 access errors surface at query time in Step 8. Show any error and ask the user to fix.

**Create the dataset.** Write the dataset body from Step 6 to a JSON file and execute:

`PUT /_query/dataset/<dataset_name>` with `--input-file <path>`.

Common failures: data source does not exist (register it first), resource glob matches no files, `partition_spec` syntax
error. Show the error and ask the user to fix.

**Verify both registrations.** Call `GET /_query/data_source/<data_source_name>` and
`GET /_query/dataset/<dataset_name>` to confirm the definitions were stored correctly.

### 8. Probe for data

Run `POST /_query` with a bounded window matching one partition unit. Start from the current hour (UTC) and step
backward:

```esql
FROM <dataset>
| WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>")
| LIMIT 1
```

If 0 rows, step backward by one partition unit (one hour for hourly partitions). Try at most 3 windows. If all return 0
rows, ask the user for a time range that has data. Do not widen the window — that scans more files.

### 9. Hand off

Once `POST /_query` returns rows, guide the user on **which source to query**.

**Check index quality.** If the integration is ingesting, verify the index has usable data:

```esql
FROM logs-<dataset>-* | STATS total = COUNT(*), parsed = COUNT(<key_field>) | LIMIT 1
```

If `parsed` ≪ `total`, the index is broken (e.g. Parquet ingested through a text-only pipeline). Use FDS exclusively.

**Choose the source.** The index and FDS dataset have different field names and types (see the integration reference).

- **Prefer the index** when it has properly parsed data **and** the user needs alerting, ML, or GeoIP enrichment.
- **Prefer FDS** for ad-hoc stats (flat column names, native numeric types), time ranges outside the index, or when
  the index is broken.
- **Always inspect the mapping** (`GET /{index}/_mapping`) before querying the index — FDS and index column names
  differ.

Report: the dataset name, a confirmed time window with data, and which source to use. Hand off to
**kibana-fds-dashboards** for dashboards or **elasticsearch-esql** for queries.

## Edge Cases

- **Feature not enabled:** `GET /_query/data_source` returns a feature-not-enabled error. Tell the user to set
  `esql.federation.enabled: true` and restart the node, and `xpack.dataFederation.enabled: true` in Kibana.
- **Cluster-state encryption missing:** `PUT /_query/data_source` fails when credentials need encrypted storage. Tell
  the user to configure the Elasticsearch keystore.
- **No Fleet policy:** The user has S3 data but no installed integration. Skip Steps 2–3 and collect bucket, region,
  prefix, format, and `@timestamp` details from the user directly. Proceed from Step 4.
- **SQS-only policy:** `collect_s3_logs` is `false` or `bucket_arn` is empty (the default for all AWS data streams). Ask
  the user for bucket details.
- **Access Point ARN only:** Policy has `access_point_arn` but no `bucket_arn`. Ask the user for the underlying bucket
  name — the access point ARN does not expose it.
- **Access keys are secrets:** Fleet returns `::es_redacted::`. Ask for `role_arn` instead.
- **Unrecognised extension:** On Elasticsearch before 9.6, plain text (`.log`, `.txt`, `.log.gz`) is not supported.
  Report and stop. On 9.6+, set `settings.format` to `"csv"` or `"tsv"` to force that reader, then verify with a test
  query. A failed test means stop.
- **Compressed structured files:** `.json.gz` or `.csv.gz` may work. Verify with a test query before committing.
- **GCS or Azure data:** This skill covers S3 only. GCS and Azure Blob require different `type` values in the data
  source body. Report the limitation and stop.
- **Dataset name conflict:** `PUT /_query/dataset` fails if the name collides with an index, alias, or view. Suggest a
  different name.
- **Dataset already exists:** Skip registration — tell the user the existing dataset name and jump to the probe step.
- **0 rows on probe:** No files in the tested partition window. Ask for a known time range.
- **Index has unparsed data:** The VPC flow integration does not support Parquet. If
  `STATS total = COUNT(*), parsed = COUNT(<key_field>)` shows `parsed` ≪ `total`, use FDS exclusively. Tell the user
  to reconfigure for plain text if they need indexed data.
- **Prefix layout mismatch:** The policy prefix may differ from the reference (e.g. hive-style `aws-account-id=...`
  vs classic `AWSLogs/<account>/...`). Always use the policy prefix.

## Examples

**"Set up an FDS dataset from my aws.vpcflow integration."** —

1. Call `GET kbn:/api/fleet/package_policies` → find the policy with an `aws-s3` input and `aws.vpcflow` stream.
2. From the stream vars: `bucket_arn: arn:aws:s3:::my-vpc-flow-logs`, `region: us-east-1`,
   `bucket_list_prefix: AWSLogs/...`. From the policy-level vars: `role_arn: arn:aws:iam::123456789012:role/fds-reader`.
3. Read `references/aws-vpcflow.md` for `@timestamp` mapping and hive layout.
4. Confirm data source and dataset bodies with the user.
5. Register via `PUT /_query/data_source/<name>` and `PUT /_query/dataset/<name>`.
6. Probe with `FROM vpc_flow | WHERE @timestamp >= ... | LIMIT 1`.
7. Hand off to **kibana-fds-dashboards**.

**"I need VPC flow data from last month but the integration only starts from yesterday."** —

1. Call `GET /_query/dataset` → find `vpc_flow_ui` already registered with a resource glob covering `year=*/month=*`.
2. Call `GET kbn:/api/fleet/package_policies` → find the policy. Stream var `start_timestamp: 2026-11-13T00:00:00Z`.
3. Explain: `start_timestamp` limits agent ingestion, not FDS. The FDS dataset already covers all months in the bucket.
4. Probe: `FROM vpc_flow_ui | WHERE @timestamp >= TO_DATETIME("2026-10-21T09:00:00.000Z") AND @timestamp < TO_DATETIME("2026-10-21T10:00:00.000Z") | LIMIT 1` → returns rows.
5. Report: "The dataset `vpc_flow_ui` already has October data. Use `FROM vpc_flow_ui | WHERE @timestamp >= ...` to
   query it. No registration or integration change needed."
6. If the user also needs alerts on October data, advise changing `start_timestamp` to `2026-10-01T00:00:00Z` on the
   Fleet policy and waiting for re-ingestion.

**"I want to explore my CloudTrail logs in S3 but there's no integration set up yet."** —

1. Call `GET kbn:/api/fleet/package_policies` → no matching CloudTrail policy found.
2. Ask the user for: S3 bucket name, AWS region, object prefix (e.g. `AWSLogs/<account>/CloudTrail/<region>/`), file
   format (`.json.gz` NDJSON), and `@timestamp` column (CloudTrail uses `eventTime`, ISO 8601).
3. No reference file exists for CloudTrail — confirm the layout is date-prefix (`<YYYY>/<MM>/<DD>/`), not hive.
4. Build data source and dataset bodies. For a plain date-prefix layout, leave `partition_detection` at its default and
   do not set `partition_spec`.
5. Register via `PUT /_query/data_source/<name>` and `PUT /_query/dataset/<name>`, probe, and hand off.

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
   from the sample in Step 8.
5. **Show bodies and wait for confirmation.** Both the data source and dataset bodies contain credentials and access
   details. The user must review them before execution.
6. **One data source per credential set, one dataset per file collection.** Multiple datasets can share a data source if
   they use the same credentials and region.
7. **Check for existing datasets before registering.** Call `GET /_query/dataset` first. If a dataset already covers
   the same bucket and prefix, reuse it instead of creating a duplicate.
8. **Distinguish agent ingest settings from FDS scope.** `start_timestamp` and `ignore_older` are agent filters — they
   do not affect FDS queries. Do not tell the user to change the integration when FDS already covers their time range.
9. **Use the Fleet policy prefix, not the reference template.** The `bucket_list_prefix` from the policy is the actual
   S3 path. The reference file documents common layouts but cannot predict every account's prefix structure.
10. **Choose the right source for the task.** The index and FDS dataset have different schemas and types. Prefer the
    index for alerting, ML, or ECS-enriched fields. Prefer FDS for ad-hoc stats (flat names, native numeric types) or
    time ranges the index does not cover. Always inspect the mapping before querying the index.

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
