# Column Identification for Headerless CSV and TSV

Use this reference when a headerless CSV or TSV file has no matching reference file (or the user rejected a fingerprint
proposal). This procedure collects column names and types from the user through one of three paths, validates the
candidate mapping against sample data, and produces a confirmed `properties` block for the dataset registration in
Step 6.

## Entry point

Ask the user:

> "The file has **N columns** (`col0` through `colN`). To build a usable dataset, I need column names and types.
> Please choose one:
>
> **A)** Name the log type (e.g. 'nginx combined', 'CloudFront standard') — I will look up the official format
> documentation.
>
> **B)** Provide the column mapping directly — as a typed field list, ordered names, or a JSON properties block.
>
> **C)** I don't know — show me sample data and I will identify the columns."

If [csv-fingerprints.md](csv-fingerprints.md) already collected schema and sample data (TEMP #1), mention the column
count and inferred types in the question. If fingerprinting was skipped (the user stated a type that had no reference
file), register TEMP #1 now to get the schema before asking — the sample data is needed by all three paths for
validation.

## Path A — User names a known log type

The user states the log type but no pre-built reference file exists for it (e.g. "nginx combined", "CloudFront
standard", "S3 server access logs").

### Configurable format warning

Some log formats allow custom field configurations. When the user names one of these, warn before proceeding:

- **nginx** — `log_format` directive in `nginx.conf` can redefine fields entirely
- **Apache** — `LogFormat` directive defines custom patterns
- **syslog / rsyslog** — templates control the output format
- **Application frameworks** — (Log4j, Logback, etc.) custom pattern layouts

> "**Note:** [format name] supports custom configurations. The official documentation describes the default format. Your
> files may differ. The validation step will catch any mismatches."

### Documentation lookup

1. If the user provides a documentation URL, fetch it with `WebFetch`.
2. If no URL is provided, search with `WebSearch` for `"<type> access log format fields site:<vendor-domain>"` (e.g.
   `"nginx combined log format fields site:nginx.org"`). Fetch the top result with `WebFetch`.
3. Extract the field list from the fetched page. Look for a table or ordered list of field names, positions, and types.
4. Map each field to a positional `colN` based on its order in the documentation.
5. Assign types using these rules:
   - Timestamp fields → `date` or `date_nanos` (with `format` if needed)
   - IP address fields → `ip`
   - Byte counts, status codes, numeric-only fields → `long` or `integer`
   - Fields that can contain sentinel values (`-`, `-1`, `NA`) → `keyword`
   - Everything else → `keyword`
6. Designate one field as `@timestamp` (with `path: "colN"`). If the documentation names the timestamp field, use it.
   Otherwise ask the user.

Build a candidate `properties` mapping and proceed to the [validation gate](#validation-gate).

**If the web fetch fails or returns no usable field list**, report the failure and ask the user to provide the mapping
directly (fall through to Path B) or to show sample data (fall through to Path C).

## Path B — User provides their own mapping

Accept any of these input forms:

### Ordered field names (simplest)

```text
client_ip, remote_user, timestamp, request, status, bytes, referer, user_agent
```

Assign positions left to right: `client_ip` → `col0`, `remote_user` → `col1`, etc. Apply type inference:

- The field the user designates as the timestamp → `date` or `date_nanos` (ask for `format` if the value is not ISO
  8601)
- `keyword` for everything else unless the user specifies otherwise

Ask: "Which of these is the timestamp field?"

### Typed field list

```text
client_ip:ip, remote_user:keyword, @timestamp:date_nanos, request:keyword, status:integer, bytes:long
```

Parse `name:type` pairs. Assign positions left to right. If the user includes `@timestamp`, use it directly. If they
name the timestamp field something else, map it to `@timestamp` with a `path` attribute.

Supported types: `keyword`, `text`, `long`, `integer`, `double`, `float`, `date`, `date_nanos`, `ip`, `boolean`.

### JSON properties block

```json
{
  "@timestamp": { "type": "date_nanos", "path": "col0" },
  "client_ip": { "type": "ip", "path": "col3" },
  "status": { "type": "integer", "path": "col5" }
}
```

Use as-is. Verify it includes `@timestamp`. If not, ask the user which field is the timestamp.

### Documentation URL

The user provides a URL instead of field names. Treat this the same as Path A: fetch the URL, extract the field list,
build a candidate mapping, and proceed to validation.

---

For all Path B input forms, build a candidate `properties` mapping and proceed to the [validation gate](#validation-gate).

## Path C — Unknown format, show sample data

The user cannot name the log type or provide a mapping. Show them the data so they can identify the columns.

### Evidence collection

If [csv-fingerprints.md](csv-fingerprints.md) already collected schema and sample data from TEMP #1, reuse those
results. Otherwise, register TEMP #1 now:

```json
{
  "data_source": "<data_source_name>",
  "resource": "<same resource glob>",
  "settings": { "<same settings>" },
  "mappings": { "dynamic": true }
}
```

Run the following queries (all with a time bound if TEMP #1 has a timestamp candidate, otherwise use `LIMIT` only):

1. **Schema** — `FROM _temp_<name> | LIMIT 0` → column names and inferred types.
2. **Sample** — `FROM _temp_<name> | LIMIT 10` → sample values for each column.
3. **Cardinality** — for each column:
   `FROM _temp_<name> | WHERE ... | STATS distinct = COUNT_DISTINCT(<colN>)` → distinct value count.

### Evidence table

Present the results as a table:

```text
| Column | Inferred type | Sample values (first 3)              | Distinct |
|--------|---------------|--------------------------------------|----------|
| col0   | keyword       | 192.168.1.50, 10.0.0.1, 172.16.0.5  | >100     |
| col1   | keyword       | -, -, -                              | 1        |
| col2   | keyword       | admin, anonymous, guest              | 3        |
| col3   | date          | 2026-11-17T10:23:01Z, ...            | >100     |
| col4   | keyword       | GET /api/health HTTP/1.1, ...        | 12       |
| col5   | integer       | 200, 404, 500                        | 5        |
| col6   | integer       | 1234, 567, 8901                      | >100     |
| col7   | keyword       | https://example.com, -, -            | 8        |
| col8   | keyword       | Mozilla/5.0..., curl/8.5.0           | 15       |
| col9   | double        | 0.003, 0.015, 0.001                  | >100     |
```

Ask the user:

> "Which column is the timestamp? Please provide names for the columns you want to use in queries. You can name all
> columns or only the ones you need — unnamed columns will be dropped from the schema.
>
> Example: `col0: client_ip, col3: @timestamp, col4: request, col5: status`"

**Do not suggest column names from data patterns.** The agent shows observable facts (types, sample values, cardinality).
The user provides the semantic layer (names, meanings).

### Build the mapping

From the user's response, build a candidate `properties` mapping with `"dynamic": false`. Only include columns the user
named. Set `@timestamp` with the appropriate `path` and type. Apply the type from the inferred schema unless the user
overrides it.

Delete TEMP #1 and proceed to the [validation gate](#validation-gate).

## Validation gate

This gate applies to all candidate mappings **except** pre-built reference file mappings (`aws-elb.md`,
`aws-vpcflow.md`), which are pre-verified and trusted.

### Procedure

1. **Register a validation dataset.** Create `_validate_<name>` with `dynamic: false` and the candidate `properties`
   mapping:

   `PUT /_query/dataset/_validate_<name>` with `--input-file <path>`.

2. **Sample with named columns.** Run:

   ```esql
   FROM _validate_<name>
   | KEEP @timestamp, <named_col_1>, <named_col_2>, ...
   | WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>")
   | LIMIT 5
   ```

   Use the same time window from the Step 8 probe or from TEMP #1's sample.

3. **Show a side-by-side comparison.** Present each named column with its sample values:

   ```text
   | Column name              | Type      | Sample values                        |
   |--------------------------|-----------|--------------------------------------|
   | @timestamp               | date_nanos| 2026-11-17T10:23:01.540467Z          |
   | client_ip                | ip        | 192.168.1.50                         |
   | request                  | keyword   | GET /api/health HTTP/1.1             |
   | status                   | integer   | 200                                  |
   | bytes                    | long      | 1234                                 |
   ```

4. **Ask the user to confirm.** Present the table and ask:

   > "Do the column names match the sample values? If any column is misaligned (e.g. `client_ip` shows a timestamp
   > instead of an IP), please correct the mapping."

5. **Handle the response:**

   - **User confirms** — delete `_validate_<name>`. The mapping is ready for the final dataset registration in Step 7.
   - **User corrects** — update the mapping, delete the old `_validate_<name>`, re-register with the corrected mapping,
     and repeat from step 2.
   - **User abandons** — delete `_validate_<name>`. Stop.

### Common misalignment causes

| Symptom | Likely cause | Fix |
| ------- | ------------ | --- |
| All values are shifted by one column | Off-by-one in the positional mapping | Shift all `path: "colN"` values by 1 |
| Timestamp column shows a string, not a date | Wrong `type` or `format` | Change the type to `date` or `date_nanos`; add `format` if needed |
| Numeric column shows `null` for some rows | Sentinel values (`-`, `-1`) in a numeric column | Change the type to `keyword`; use `CASE` + `TO_DOUBLE` in queries |
| Column shows `Unknown column` error | The column was not included in `properties` | Add the missing column to the mapping |
| Fewer columns than expected | File has fewer fields than the documentation describes | The documentation may describe a newer version; map only the columns that exist |
| More columns than expected | File has more fields than the documentation describes | The documentation may describe an older version; ask the user about the extra columns |

## Cleanup

Delete every temporary dataset before proceeding to Step 7 (final registration) or before stopping on abandonment.
Temporary datasets use the prefixes `_temp_` and `_validate_`. Verify deletion with
`GET /_query/dataset/_temp_<name>` — it should return a not-found error.

At no point should two temporary datasets coexist. Delete TEMP #1 before creating TEMP #2 (the validation dataset).
