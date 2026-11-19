---
name: kibana-fds-dashboards
description: >
  Create and manage Kibana dashboards on ES|QL Data Federation (FDS) datasets with FROM <dataset>, not an Elasticsearch
  index. Every panel carries explicit ES|QL with ?_tstart/?_tend time bounds. Use when the user wants a dashboard or
  Lens visualization on Parquet, NDJSON, CSV, or TSV files registered with _query/dataset. For index-backed dashboards,
  use kibana-dashboards instead.
metadata:
  author: es-analytical-engine, kibana-presentation
  version: 0.3.0
  universal: true
compatibility: >
  Kibana 9.5+ with matching Elasticsearch 9.5+ (ES|QL Data Federation enabled), Enterprise license, self-managed or
  Elastic Cloud Hosted (not Serverless). Requires the `elastic` CLI ≥ 0.4 with `stack kb` support (dedicated
  `dashboards` and `visualizations` commands).
---

# FDS Dashboards

Create, update, and manage Kibana dashboards on ES|QL Data Federation (FDS) datasets. An FDS dataset is queried with
`FROM <dataset>`, not an Elasticsearch index. Every panel carries explicit ES|QL with `?_tstart`/`?_tend` time bounds.
This skill applies FDS-specific overrides on top of the standard dashboard patterns.

For the dataset type (data view vs ES|QL), chart design, grid layout, and panel schema, follow the **kibana-dashboards**
skill. The rules in this skill **override** the standard skill where FDS datasets differ. For writing or debugging the
ES|QL queries inside each panel, use the **elasticsearch-esql** skill. For registering an FDS dataset from a Fleet
integration, use the **observability-fds-dataset-setup** skill.

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

- A registered FDS dataset (`_query/dataset`) with `@timestamp` mapped as `date`. A hive layout also has a
  `partition_spec`. A date-prefix or flat layout does not. If the dataset is not registered, use the
  **observability-fds-dataset-setup** skill first. That skill covers S3 data sources; GCS and Azure Blob require
  different data source types not covered by either skill.
- At least one bounded `POST /_query` window that returns rows. If the probe returns 0 rows, ask the user for a time
  range that has data.
- The **kibana-dashboards** skill covers the Dashboards and Visualizations APIs, chart types, design guidelines, and
  grid layout. Read it before building any dashboard. This skill adds FDS-specific overrides only.

## FDS Overrides

These rules override the corresponding **kibana-dashboards** guidance when the data source is an FDS dataset.

### Always use ES|QL with explicit time bounds

Every panel must use `data_source.type: "esql"` with a query that includes `?_tstart`/`?_tend`. Do not use
`data_source.type: "data_view_reference"` or `data_source.type: "data_view_spec"` — an FDS dataset is not an index and
has no data view.

```esql
FROM <dataset>
| KEEP @timestamp, <columns>
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| STATS `Total` = COUNT(*)
```

### No controls

Do not add `pinned_panels` (controls). Controls read an index mapping. An FDS dataset is not an index — controls fail
silently or show no options.

### Time range

Use an absolute `time_range` for cold or historical S3 data that is not appended. Set `from` and `to` to the sample
window discovered in Step 1 (for example `{ "from": "<SAMPLE_START>", "to": "<SAMPLE_END>" }`). The default
last-15-minutes range is almost always empty for cold data. Use a relative range (for example `now-2d`) when the user
confirms data is continuously appended.

### Panel cost tiers

Every panel fires an S3 query on dashboard load. Limit the **summary tier** (top-level panels outside sections) to
approximately 8 data panels. A markdown panel for data notes does not count — it queries nothing.

**Detail sections.** Group additional panels by topic in sections. Set `collapsed: true` unless the user asks for the
section to be open. Section panels query S3 only when the user expands the section — no cost at load time. Add as many
detail panels as the data warrants.

### Do not filter on partition columns

Do not add `WHERE year = 2026 AND month = 10` to queries. When the dataset has a `partition_spec`, pruning comes from
`@timestamp` through that spec. Adding partition column filters is redundant and breaks when the time picker moves.

### Do not rename `@timestamp`

Do not `EVAL` or `RENAME` `@timestamp`. The time picker depends on that name. A `partition_spec`, when one is set, does
too.

## Critical Rule

**Never run `FROM <dataset>` without a time bound.** Every query — probes, samples, profiles, panel queries — must
include `WHERE @timestamp >= ... AND @timestamp < ...`. An unbounded query scans every file in S3 and will time out or
run up cost. The only exception is `FROM <dataset> | LIMIT 0` to list columns, and only when the dataset is Parquet and
has a `partition_spec`.

## Recognised Inputs

The user may supply some or all of these. Parse them when present. Do not guess missing values.

| Input             | Used in  | Meaning                                                |
| ----------------- | -------- | ------------------------------------------------------ |
| `DATASET`         | Step 1   | The `_query/dataset` name, e.g. `vpc_flow_ui`          |
| `SAMPLE_START`    | Step 1   | ISO start of the bounded sample window                 |
| `SAMPLE_END`      | Step 1   | ISO end of the bounded sample window                   |
| `DASHBOARD_TITLE` | Step 3/6 | Title for the dashboard                                |
| `FIELD_NOTES`     | Step 3   | Column meanings, sentinel values, and "no data" tokens |

## Process

Run all read-only work (Steps 1–2) in one pass before asking the user anything, **except**: if the dataset has no
`partition_spec` and the initial probe returns 0 rows, stop and ask the user for a time range before continuing. Do not
try additional probe windows — every query reads the same files, so a different date cannot find data faster.

### 1. Confirm the dataset

**Verify connectivity.** Call `GET kbn:/api/status`. Read `version.number` to confirm Kibana is 9.5+. If the call fails,
stop and surface the error.

If the user did not name a dataset, call `GET /_query/dataset` to list all registered datasets. Show the names and ask
the user to pick one. The **observability-fds-dataset-setup** skill hands off with a specific dataset name.

Read the dataset definition with `GET /_query/dataset/<name>`. Record: format, `schema_resolution`, `partition_spec`,
and `@timestamp` mapping. That definition is the column list.

**Parquet with a `partition_spec`.** Listing columns may also use:

```esql
FROM <dataset> | LIMIT 0
```

**Find the sample window.** If `SAMPLE_START` and `SAMPLE_END` are given, use them. Otherwise the probe depends on the
layout.

For Parquet with a `partition_spec`, probe for the latest partition that has data. Pick a window matching the finest
partition unit (one hour for hourly partitions):

```esql
FROM <dataset>
| WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>")
| LIMIT 1
```

Start from the current hour (UTC) and step backward by one partition unit. Try at most 3 windows. If all return 0 rows,
ask the user for a time range that has data. Do not widen the window — on a partitioned Parquet dataset, a wider window
reads more files.

For every other layout (no `partition_spec`, or a format other than Parquet), do not run `LIMIT 0` and do not step
through extra windows. A tighter `@timestamp` filter does not reduce the files opened — every query reads every file
the glob matches. Probe a single 1-day window from `now-1d` to `now` (current UTC day):

```esql
FROM <dataset>
| WHERE @timestamp >= TO_DATETIME("<now-1d>") AND @timestamp < TO_DATETIME("<now>")
| LIMIT 1
```

One probe, one chance. If it returns 0 rows, stop and ask the user for a time range that has data. Do not try
additional windows, do not widen the range, and do not guess dates from examples in this document.

### 2. Sample and profile

Use the window from Step 1. Every query must include the time filter. Use the same profiling approach as for any ES|QL
dataset:

1. **Sample** —
   `FROM <dataset> | KEEP @timestamp, <columns> | WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>") | LIMIT 20`
2. **Profile** —
   `FROM <dataset> | WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>") | STATS rows = COUNT(*), <col>_nulls = COUNT(*) - COUNT(<col>), <col>_distinct = COUNT_DISTINCT(<col>)`
   for every column except `@timestamp` and partition columns.
3. **Frequent values** — for up to 6 low-cardinality columns (distinct ≤ 20), run
   `FROM <dataset> | WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>") | EVAL v = <col> | STATS n = COUNT(*) BY v | SORT n DESC | LIMIT 10`.

### 3. Confirm inputs with the user

Do not invent a vendor, a product, or a log format. Prior knowledge is not evidence — only the sample and the user's
confirmations are.

**If `FIELD_NOTES` was provided**, list each input as `- KEY: value` (DATASET, FORMAT, SAMPLE window and row count,
DASHBOARD_TITLE, FIELD_NOTES as given). Add: "To override any value, reply with `- KEY: new value`." Do not re-ask
column meanings. On approval, continue to Step 4.

**If `FIELD_NOTES` was not provided**, show: DATASET, FORMAT, SAMPLE window and row count, DASHBOARD_TITLE (default:
dataset name title-cased), and the column profile table. Ask for FIELD_NOTES (column meanings, "no data" tokens). On CSV
or TSV, call out that columns may need `TO_LONG` or `TO_DOUBLE` casts — see
[references/formats.md](references/formats.md). Wait for the answer before proceeding.

### 4. Plan panels

Classify each confirmed column (excluding `@timestamp` and partition columns):

| Profile              | Chart rule                                        |
| -------------------- | ------------------------------------------------- |
| `distinct = 1`       | Constant — mention in description, do not chart   |
| `distinct 2–10`      | Pie (donut) or color split on time series         |
| `distinct 11–100`    | Top-N xy horizontal bar (`SORT DESC \| LIMIT 10`) |
| `distinct > 100`     | Top-N bar; never use as breakdown dimension       |
| Numeric, nulls < 50% | Metric (SUM/AVG) and/or time series               |
| Nulls ≥ 90%          | Skip — mention in the description or data notes   |

After excluding "no data" tokens, re-evaluate distinct counts. A column with only 1 remaining real value is a constant.

Classify the column the panel groups on. When the query derives that column with `DISSECT` or `CASE`, profile the
derived value and apply the table to that distinct count. The raw column's cardinality is the wrong signal.

**Derived metrics.** If two columns could yield a useful derived value (e.g. flow duration from `@timestamp` and `end`),
suggest it to the user. If the user did not confirm the semantics, list the suggestion but do not chart it. When an
epoch-second column needs date arithmetic, cast it first:
`DATE_DIFF("second", @timestamp, TO_DATETIME(end * 1000))`.

### 5. Write ES|QL for each panel

Copy the appropriate recipe from [references/esql-recipes.md](references/esql-recipes.md). Substitute the dataset,
columns, and "no data" filters. Read the format traps in [references/formats.md](references/formats.md).

FDS-specific rules (general ES|QL rules are in the **elasticsearch-esql** skill):

- `FROM <dataset>`, then `KEEP`, then `WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend`.
- Time series: `STATS ... BY \`Time\` = BUCKET(@timestamp, 100, ?_tstart, ?_tend)`. Always alias the BUCKET, and use
  `Time` in `x.column`. This alias is an FDS override of the kibana-dashboards raw `BUCKET(...)` column. Follow
  kibana-dashboards for axis `scale`, `domain`, and `label` only.
- Exclude agreed "no data" tokens with `!=`, not `IS NULL` (unless `null_value` is set on the dataset).
- Cast keyword numerics with `TO_LONG` or `TO_DOUBLE` before aggregating (text formats only; Parquet numerics do not
  need casts).
- For a keyword numeric field that can contain a sentinel (`-`, `-1`), guard the cast:
  `EVAL value = CASE(field == "-1" OR field == "-", null, TO_DOUBLE(field))`, then aggregate `value`. See the sentinel
  recipe in [references/esql-recipes.md](references/esql-recipes.md).

**Run every panel query before saving it.** Substitute the sample window timestamps from Step 1 for `?_tstart` and
`?_tend`. If the query fails, fix it before proceeding.

### 6. Build the dashboard

Write the dashboard JSON body for `PUT kbn:/api/dashboards/{id}`. Follow the **kibana-dashboards** skill for the full
panel schema (panel types, axis keys, column references, grid layout, and chart design). The FDS overrides above (ES|QL
only, no controls, time range, panel cost tiers) apply on top.

**Data notes.** Add a full-width markdown panel (`w: 48`, `h: 3`) as a top-level panel above the summary metrics, only
when a reader would misread a chart without it: a sentinel the sample confirmed, a unit the column does not show, or a
constant the sample proved. Do not add a note just to repeat the dataset name. Include only what the sample and
`FIELD_NOTES` confirmed. Do not paste the schema, the bucket, the delimiter, or the mapping. Do not add a `ROW`-based
"Window start" metric — the time picker already shows the selected range.

**Summary tier** (top-level panels, no section). KPI metrics, one pie/donut breakdown, and the primary time series.
Keep this tier to approximately 8 data panels. A data-notes markdown does not count.

**Detail sections.** Group additional panels by topic (e.g. "Top Sources and Destinations", "Ports and Protocol").
Add one `data_table` of recent records only when a single row is worth reading, such as access logs, flow logs, or
events. Skip it for a metrics-only or aggregate source. When you add it, use `SORT @timestamp DESC | LIMIT 100`,
`"styling": { "paging": 20 }`, and only the columns needed to inspect one event. Section panels query S3 only when the
user expands the section. Set `collapsed: true` unless the user asks for the section to be open.

### 7. Verify

After `PUT kbn:/api/dashboards/{id}`, call `GET kbn:/api/dashboards/{id}` to confirm the dashboard was created. Compare
the returned config against what was sent and look for unexpected defaults. A duration `format` with no `decimals`
comes back as `decimals: 0`, which shows a sub-second processing time as 0. Set `decimals` explicitly (for example `1`)
on those metrics and series. Report the id and title.

## Edge Cases

- **0 rows (Parquet with `partition_spec`):** The probed partition has no files. Step backward by one partition unit,
  up to 3 times. If all return 0 rows, ask the user for a time range.
- **0 rows (no `partition_spec`):** The 1-day probe found no data. Stop and ask the user for a time range. Do not retry
  with different dates — every query reads the same files.
- **Glob pattern discovered too many files:** A hive `partition_spec` is not pruning, or a date-prefix or flat layout
  has no `partition_spec` and the glob is wider than the data. Report which case it is.
- **0 rows on `.gz`/`.zst` text file:** An unsupported compression and format combination. On Elasticsearch 9.6+, a
  `.log.gz` file can be read when the dataset sets `settings.format` to `csv` or `tsv`. If that setting is already set
  and the probe is still empty, report and stop.
- **Column missing (text dataset):** Schema inference did not sample it. Do not invent it.
- **User names an index:** Stop and use the **kibana-dashboards** skill.
- **No dataset registered:** Stop and use the **observability-fds-dataset-setup** skill.

## Examples

**"Build a dashboard on my vpc_flow FDS dataset."** —

1. Read the dataset definition. Confirm format (Parquet), `@timestamp` mapping, and `partition_spec`.
2. Sample 20 rows from the latest hourly partition with data.
3. Profile columns: `srcaddr` (high cardinality), `dstaddr` (high cardinality), `action` (2 values: ACCEPT/REJECT),
   `protocol` (3 values: 6/17/1), `bytes` (numeric), `packets` (numeric).
4. Plan: 3 metrics (Total Flows, Total Bytes, Reject Rate), 1 pie (Action), 1 time series (Flows/min), 1 horizontal bar
   (Top Source IPs), 1 horizontal bar (Top Dest Ports). Add a data-notes markdown because `action = "-"` is no recorded
   action and protocol 6/17/1 means TCP/UDP/ICMP. Add a recent-rows table because one flow is worth reading. Those two
   panels are for this source, not for every dataset.
5. Write ES|QL for each panel from `references/esql-recipes.md`.
6. Build and PUT the dashboard. Use an absolute `time_range` for cold data, or a relative range when the user confirms
   data is continuously appended.
7. Verify with GET.

**"Add a section to an existing FDS dashboard."** —

1. `GET kbn:/api/dashboards/{id}` — read the current dashboard.
2. Add a new section entry in `panels` with the requested panels.
3. `PUT kbn:/api/dashboards/{id}` — send the full updated body.
4. Verify with GET.

## Operations

| HTTP API (shorthand)           | `elastic` CLI command                                                           |
| ------------------------------ | ------------------------------------------------------------------------------- |
| `GET kbn:/api/status`          | `elastic kb system get-status`                                                  |
| `GET /_query/dataset`          | `elastic es esql get-dataset`                                                   |
| `GET /_query/dataset/<name>`   | `elastic es esql get-dataset --name '<name>'`                                   |
| `POST /_query`                 | `elastic es esql query --format tsv --query "<esql>"`                           |
| `PUT kbn:/api/dashboards/{id}` | `elastic kb dashboards upsert-dashboard --id '<id>' --input-file '<path.json>'` |
| `GET kbn:/api/dashboards/{id}` | `elastic kb dashboards get-dashboard --id '<id>'`                               |
