# CSV Fingerprints

Use this reference after Step 4 confirms a headerless CSV file and the user has not stated the log type. The fingerprint
procedure registers a temporary dataset, inspects the inferred schema and sample values, and matches them against known
formats. A match proposes the format to the user for confirmation. No match falls through to
[column-identification.md](column-identification.md).

## Temporary registration for fingerprinting

Register a temporary dataset with `dynamic: true` and no explicit properties. Use a name prefixed with `_temp_` to
signal its throwaway purpose (e.g. `_temp_fingerprint`).

```json
{
  "data_source": "<data_source_name>",
  "resource": "<same resource glob as the final dataset>",
  "settings": {
    "<same settings as the final dataset>"
  },
  "mappings": {
    "dynamic": true
  }
}
```

`PUT /_query/dataset/_temp_fingerprint` with `--input-file <path>`.

Run two queries:

1. **Schema** — `FROM _temp_fingerprint | LIMIT 0`. Record the column count and each column's inferred type.
2. **Sample** — `FROM _temp_fingerprint | LIMIT 5`. Record sample values, focusing on `col0` and any column inferred as
   `date` or `date_nanos`.

These results feed both the fingerprint table below and, if no match is found, Path C in
[column-identification.md](column-identification.md). Retain the query results for reuse before deleting the temporary
dataset.

## Fingerprint table

Match the observed signals against each row. All conditions in a row must match. Check rows top to bottom; stop at the
first match.

| Column count | col0 sample values         | Timestamp candidate          | Identified format      | Reference file                                     |
| ------------ | -------------------------- | ---------------------------- | ---------------------- | -------------------------------------------------- |
| 36           | `http`, `https`, `h2`      | col1 inferred as `date_nanos`| AWS ALB access logs    | [aws-elb.md](aws-elb.md) — ALB section (36 fields) |
| 22           | `tls`, `tcp`, `udp`        | col2 inferred as `date`      | AWS NLB access logs    | [aws-elb.md](aws-elb.md) — NLB section (22 fields) |
| 15           | ISO 8601 timestamp string  | col0 inferred as `date_nanos`| AWS CLB access logs    | [aws-elb.md](aws-elb.md) — CLB section (15 fields) |
| `<count>`    | `<distinguishing values>`  | `<colN>` inferred as `<type>`| `<format name>`        | `<reference-file.md>` — `<section>`                |

## On match

Propose the identified format to the user with the evidence that led to the match:

> "The file has **36 columns**. `col0` contains values like `http` and `https`. `col1` was inferred as `date_nanos`.
> This matches the **AWS ALB access log** format. Is this correct?"

- **User confirms** — use the full `properties` mapping from the matched reference file section. Delete the temporary
  dataset. Proceed to the validation gate in [column-identification.md](column-identification.md) to verify the mapping
  against sample data before final registration.
- **User rejects** — delete the temporary dataset. Fall through to the three-path ask in
  [column-identification.md](column-identification.md). Pass along the schema and sample results so Path C can reuse
  them without re-registering.

## On no match

No row in the table matched. Delete the temporary dataset. Fall through to
[column-identification.md](column-identification.md) and pass along the schema and sample results for reuse.

## Extensibility (for skill maintainers)

Users do **not** need to edit this file. Formats not in the fingerprint table are handled at runtime through the
three-path flow in [column-identification.md](column-identification.md) (doc lookup, user-provided mapping, or
interactive identification from sample data).

When a format is encountered frequently, a **skill maintainer** can promote it to a trusted fingerprint by:

1. Creating a reference file (e.g. `aws-cloudfront.md`) with the verified `properties` mapping.
2. Adding a row to the fingerprint table above. Use the template row (last row) as a starting point.

Each new row must include structural signals only (column count, positional types, col0 values) — not semantic
inferences from data patterns. Reference files for non-CSV formats (e.g. `aws-vpcflow.md` covers Parquet) do not need
a fingerprint row — Parquet files carry column names in their metadata.

No change to `SKILL.md` is needed when a row is added — the routing instruction in Step 4 already points here.
