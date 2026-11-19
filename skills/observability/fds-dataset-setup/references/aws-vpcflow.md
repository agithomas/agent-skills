# AWS VPC Flow Logs — FDS Integration Reference

This reference provides the details that a Fleet `aws.vpcflow` package policy does not carry but that an FDS dataset
registration requires.

## `@timestamp` Mapping

VPC Flow Logs record the start of each flow as a Unix epoch in the `start` column.

```json
{
  "@timestamp": {
    "type": "date",
    "path": "start",
    "format": "epoch_second"
  }
}
```

The `end` column (also epoch seconds) marks the capture-window end and does not need a mapping for `@timestamp`.

## Object Layout

AWS delivers VPC Flow Logs to S3 in two known prefix styles. Both use the same hive partition columns
(`year`, `month`, `day`, `hour`) for the date structure. The prefix **before** the date partitions differs:

### Classic prefix (most common)

```text
s3://<bucket>/AWSLogs/<account-id>/vpcflowlogs/<region>/year=<YYYY>/month=<MM>/day=<DD>/hour=<HH>/<file>.parquet
```

The account ID and region are bare path segments (not hive keys).

### Hive-partitioned prefix

```text
s3://<bucket>/AWSLogs/aws-account-id=<account-id>/aws-service=vpcflowlogs/aws-region=<region>/year=<YYYY>/month=<MM>/day=<DD>/hour=<HH>/<file>.log.parquet
```

The account ID, service, and region are hive-style key=value segments. File names end in `.log.parquet`.
The `*.parquet` glob matches both `.parquet` and `.log.parquet`.

Use the Fleet policy's `bucket_list_prefix` as the source of truth — do not substitute one layout for the other.
Partition values may be zero-padded (`day=01`) or not (`day=1`); hive detection handles both automatically.

### Flat-path layout (unsupported)

Some accounts use a date-prefix layout without hive keys:

```text
s3://<bucket>/AWSLogs/<account-id>/vpcflowlogs/<region>/<YYYY>/<MM>/<DD>/<file>.log.gz
```

`.log.gz` plain text is **not supported** by Data Federation. If the user has this layout, they must re-configure the
VPC Flow Log to deliver Parquet files, or stop.

## Partition Spec

For the hive layout:

```text
year(@timestamp), month(@timestamp), day(@timestamp), hour(@timestamp)
```

This tells the query planner which files to skip based on the `@timestamp` filter. Without it, every query scans all
files in the resource glob.

## File Format

- **Parquet** is the recommended format. Schema comes from file metadata. Numeric columns (`bytes`, `packets`, `start`,
  `end`, `srcport`, `dstport`, `protocol`) are typed correctly.
- **CSV** and **NDJSON** are possible if the user configured a custom delivery, but rare. Numeric columns may be
  `keyword` and require `TO_LONG` or `TO_DOUBLE` casts before aggregation.
- **Plain text** (`.log`, `.log.gz`) is not supported.

## Known Columns (Parquet, default v2/v3/v5 format)

| Column         | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| `version`      | integer | Flow log version (2, 3, or 5)                |
| `account_id`   | keyword | AWS account ID                               |
| `interface_id` | keyword | Network interface ID (ENI)                   |
| `srcaddr`      | keyword | Source IP address                            |
| `dstaddr`      | keyword | Destination IP address                       |
| `srcport`      | integer | Source port                                  |
| `dstport`      | integer | Destination port                             |
| `protocol`     | integer | IANA protocol number (6=TCP, 17=UDP, 1=ICMP) |
| `packets`      | long    | Packet count in the capture window           |
| `bytes`        | long    | Byte count in the capture window             |
| `start`        | long    | Start of capture window (Unix epoch seconds) |
| `end`          | long    | End of capture window (Unix epoch seconds)   |
| `action`       | keyword | `ACCEPT` or `REJECT`                         |
| `log_status`   | keyword | `OK`, `NODATA`, or `SKIPDATA`                |

Version 3+ adds: `vpc_id`, `subnet_id`, `instance_id`, `tcp_flags`, `type`, `pkt_srcaddr`, `pkt_dstaddr`, `az_id`,
`sublocation_type`, `sublocation_id`.

Version 5+ adds: `pkt_src_aws_service`, `pkt_dst_aws_service`, `flow_direction`, `traffic_path`.

## FDS vs Elasticsearch Index Schema

The FDS dataset reads raw Parquet columns. The Elasticsearch index (`logs-aws.vpcflow-*`) receives ECS-mapped fields
from the ingest pipeline. Field names and types differ:

| FDS (Parquet)  | ES index field            | FDS type  | ES index type | Notes                                     |
| -------------- | ------------------------- | --------- | ------------- | ----------------------------------------- |
| `action`       | `aws.vpcflow.action`      | `keyword` | `keyword`     | Also mapped to `event.action`             |
| `bytes`        | `aws.vpcflow.bytes`       | `long`    | `keyword`     | Needs `TO_LONG()` cast for `SUM` in index |
| `packets`      | `aws.vpcflow.packets`     | `long`    | `keyword`     | Needs `TO_LONG()` cast for `SUM` in index |
| `srcaddr`      | `source.address`          | `keyword` | `ip`          | Index has GeoIP enrichment                |
| `dstaddr`      | `destination.address`     | `keyword` | `ip`          | Index has GeoIP enrichment                |
| `srcport`      | `source.port`             | `integer` | `long`        |                                           |
| `dstport`      | `destination.port`        | `integer` | `long`        |                                           |
| `protocol`     | `network.iana_number`     | `integer` | `keyword`     |                                           |
| `start`        | `aws.vpcflow.start`       | `long`    | `keyword`     | Index `@timestamp` from ingest pipeline   |
| `end`          | `aws.vpcflow.end`         | `long`    | `keyword`     |                                           |
| `log_status`   | `aws.vpcflow.log_status`  | `keyword` | `keyword`     |                                           |
| `interface_id` | `aws.vpcflow.interface_id`| `keyword` | `keyword`     |                                           |
| `account_id`   | `cloud.account.id`        | `keyword` | `keyword`     |                                           |

**For ad-hoc stats**, prefer FDS — `bytes` and `packets` are already numeric. The index maps them as `keyword`,
requiring `TO_LONG()` casts. **For alerting, ML, or GeoIP**, the index is required — FDS does not support those.

**The VPC flow integration does NOT support Parquet ingestion.** The ingest pipeline expects plain text. If the agent
reads `.log.parquet` files, the index will have rows but parsed fields will be null. Detect with
`FROM logs-aws.vpcflow-* | STATS total = COUNT(*), parsed = COUNT(aws.vpcflow.action)` — if `parsed` ≪ `total`, use
FDS exclusively. To ingest into Elasticsearch, reconfigure the AWS VPC Flow Log for plain text (`.log.gz`).

## "No Data" Tokens

- The string `"-"` is used for fields that have no value in a given row (e.g. `srcaddr` is `"-"` when `log_status` is
  `NODATA` or `SKIPDATA`).
- `log_status == "NODATA"` means no traffic was recorded in the capture window. These rows have `"-"` for most fields.
- `log_status == "SKIPDATA"` means some traffic was skipped due to capacity constraints.

Exclude `"-"` tokens from counts, distinct counts, sums, and top-N charts:

```esql
WHERE srcaddr != "-" AND log_status == "OK"
```

## Example Dataset Registration

Derive the `resource` path from the Fleet policy's `bucket_list_prefix`. Both prefix styles use the same settings:

```json
{
  "data_source": "aws_vpcflow_s3",
  "resource": "s3://<bucket>/<bucket_list_prefix>/year=*/month=*/day=*/hour=*/*.parquet",
  "settings": {
    "schema_resolution": "first_file_wins",
    "partition_detection": "hive",
    "partition_spec": "year(@timestamp), month(@timestamp), day(@timestamp), hour(@timestamp)"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date", "path": "start", "format": "epoch_second" }
    }
  }
}
```
