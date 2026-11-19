# FDS ES|QL Recipes

Copy one recipe into the dashboard panel `config.data_source.query` field. Replace `<dataset>`, column names, and the
agreed "no data" predicate. Keep `@timestamp`, `?_tstart`, and `?_tend`.

All recipes assume the dataset has `@timestamp` mapped as `date` with a `partition_spec`. Without that mapping, the time
picker does not bind and queries scan all files.

## Probe

Find the latest partition that has data. Not for dashboard panels — use literal timestamps, not `?_tstart`/`?_tend`.

```esql
FROM <dataset>
| WHERE @timestamp >= TO_DATETIME("<start>") AND @timestamp < TO_DATETIME("<end>")
| LIMIT 1
```

If 0 rows, step backward by one partition unit. Try at most 3 windows.

## Metric

```esql
FROM <dataset>
| KEEP @timestamp
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| STATS `Total` = COUNT(*)
```

## Metric with a Filter

```esql
FROM <dataset>
| KEEP @timestamp, action
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND action == "ACCEPT"
| STATS `Accepted` = COUNT(*)
```

## XY Over Time

Time series chart. Say "bar", "area", or "line" in the panel title so Lens picks the series type.

```esql
FROM <dataset>
| KEEP @timestamp
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| STATS `Events` = COUNT(*) BY `Time` = BUCKET(@timestamp, 100, ?_tstart, ?_tend)
```

Use the alias `Time` in `x.column`. The alias is an FDS override of the kibana-dashboards raw `BUCKET(...)` column.
See the **kibana-dashboards** chart-types reference for axis configuration only (`scale`, `domain`, and `label`).

## XY Over Time with Breakdown

```esql
FROM <dataset>
| KEEP @timestamp, action
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND action != "-"
| EVAL act = action
| STATS `Flows` = COUNT(*) BY `Time` = BUCKET(@timestamp, 100, ?_tstart, ?_tend), act
```

## XY Horizontal Bar (Top-N)

Top-N breakdown. Use `type: "lnsXY"` with a horizontal bar layout. Exclude the agreed "no data" token.

```esql
FROM <dataset>
| KEEP @timestamp, srcaddr
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND srcaddr != "-"
| EVAL `Source IP` = srcaddr
| STATS `Flows` = COUNT(*) BY `Source IP`
| SORT `Flows` DESC
| LIMIT 10
```

## Pie

Use `type: "pie"`. The dashboard API stores this as `lnsPie`. For a donut, add `"styling": { "donut_hole": "m" }`. Only
when the user confirmed the category and it has fewer than about 7 values. Otherwise use an XY horizontal bar.

```esql
FROM <dataset>
| KEEP @timestamp, protocol
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| EVAL proto = CASE(protocol == 6, "TCP", protocol == 17, "UDP", protocol == 1, "ICMP", TO_STRING(protocol))
| STATS `Flows` = COUNT(*) BY proto
| SORT `Flows` DESC
| LIMIT 10
```

## Data Table

Use this when a row-level table is warranted, such as access logs, flow logs, or events. Skip it for a metrics-only
source. `LIMIT 100` is the row source. Set `"styling": { "paging": 20 }` so Lens pages that result. `paging` must be
10, 20, 30, 50, or 100. Keep the columns a reader needs to inspect one event.

```esql
FROM <dataset>
| KEEP @timestamp, srcaddr, dstaddr, srcport, dstport, action, bytes, packets
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| SORT @timestamp DESC
| LIMIT 100
```

```json
"styling": { "paging": 20 }
```

## Ratio

```esql
FROM <dataset>
| KEEP @timestamp, action
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND action IN ("ACCEPT", "REJECT")
| EVAL rejected = CASE(action == "REJECT", 1, 0)
| STATS `Reject Rate` = ROUND(100.0 * SUM(rejected) / COUNT(*), 2)
```

The denominator includes only the values the user said belong in it. Do not add NULL or a placeholder.

## Sum of a Text-Format Number

Use this when the column is `keyword` (CSV, TSV, NDJSON). A Parquet numeric column does not need the cast.

```esql
FROM <dataset>
| KEEP @timestamp, bytes
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND bytes != "-"
| EVAL b = TO_DOUBLE(bytes)
| STATS `Total Bytes` = SUM(b)
```

## CSV Sentinel Token

`IS NULL` does not match a CSV token unless the dataset sets `null_value`. Compare the token the user confirmed.

```esql
FROM <dataset>
| KEEP @timestamp, bytes
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND bytes != "NA"
| STATS `Rows with bytes` = COUNT(bytes)
```

## DISSECT a Composite Field

Headerless logs often store `ip:port` or a quoted request line in one keyword column. Split it before grouping. Keep
the source column in `KEEP`.

```esql
FROM <dataset>
| KEEP @timestamp, client_port
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
    AND client_port != "-"
| DISSECT client_port "%{client_ip}:%{client_port_num}"
| STATS `Requests` = COUNT(*) BY client_ip
| SORT `Requests` DESC
| LIMIT 10
```

For a request line of the form `METHOD url protocol`, use
`DISSECT request_line "%{method} %{url} %{protocol}"` and group by `url` or `method`. Drop rows that do not match the
pattern before charting them.

## Sentinel-Guarded Aggregation

Use this when a keyword column stores a number plus an AWS sentinel. `-` means not applicable. `-1` means a timeout or
error. `TO_DOUBLE` on either token fails the query. Convert the sentinel to null, then aggregate the alias.

```esql
FROM <dataset>
| KEEP @timestamp, target_processing_time
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| EVAL target_time = CASE(
    target_processing_time == "-1" OR target_processing_time == "-",
    null,
    TO_DOUBLE(target_processing_time)
  )
| WHERE target_time IS NOT NULL
| STATS `Target p95` = PERCENTILE(target_time, 95)
```

A Parquet numeric column does not need this guard. A text column with no sentinel uses the "Sum of a Text-Format
Number" recipe instead.

## Dotted Field

Nested NDJSON or Parquet fields use dotted names. Quote with backticks.

```esql
FROM <dataset>
| KEEP @timestamp, `user.id`
| WHERE @timestamp >= ?_tstart AND @timestamp < ?_tend
| EVAL uid = `user.id`
| STATS `Events` = COUNT(*) BY uid
| SORT `Events` DESC
| LIMIT 10
```
