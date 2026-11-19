# AWS ELB Access Logs — FDS Integration Reference

Use this reference when the user confirms they want named columns for Application Load Balancer (ALB), Network Load
Balancer (NLB), or Classic Load Balancer (CLB) access logs. Field names follow the AWS access-log documentation for
that load balancer. Do not invent names, and do not copy a name from one load balancer onto another.

These logs are headerless, space-delimited text. Register them as CSV. On Elasticsearch 9.6+, `settings.format: "csv"`
forces the CSV reader onto `.log` and `.log.gz` files. On earlier versions, stop — those extensions are not supported.

## Object layout

AWS writes ELB access logs under a plain date prefix, not hive `key=value` partitions:

```text
s3://<bucket>/<prefix>/<account-id>/elasticloadbalancing/<region>/<YYYY>/<MM>/<DD>/<file>
```

No partition pruning is available. Leave `partition_detection` at its default and do not set `partition_spec`. The
resource glob must cover the three date directories:

- ALB and NLB: `*/*/*/*.log.gz`
- CLB: `*/*/*/*.log` (Classic Load Balancer samples in this reference are uncompressed `.log` files)

Every query scans all files the glob matches. Always bound `@timestamp` in the query.

## Shared CSV settings

| Setting             | ALB               | NLB               | CLB               |
| ------------------- | ----------------- | ----------------- | ----------------- |
| `format`            | `"csv"`           | `"csv"`           | `"csv"`           |
| `delimiter`         | `" "`             | `" "`             | `" "`             |
| `header_row`        | `false`           | `false`           | `false`           |
| `mode`              | `"quoted"`        | `"plain"`         | `"quoted"`        |
| `schema_resolution` | `first_file_wins` | `first_file_wins` | `first_file_wins` |
| `mappings.dynamic`  | `false`           | `false`           | `false`           |

`mode: "quoted"` is required when a field can contain spaces inside quotes (the ALB `request_line` and `user_agent`, and
the CLB `request` and `user_agent`). NLB fields are not quoted.

## Sentinels

AWS writes `-` for "not applicable" and `-1` for a timeout or error in timing fields. Those tokens are not numbers.
Map every field that can contain `-` or `-1` as `keyword`, then convert it in ES|QL:

```esql
EVAL target_time = CASE(target_processing_time == "-1" OR target_processing_time == "-", null, TO_DOUBLE(target_processing_time))
```

Do not map those fields as `double` or `integer`. `schema_resolution: first_file_wins` will otherwise reject a later
file that contains `-`.

Byte counts (`received_bytes`, `sent_bytes`) are always numeric. Map them as `long`.

## ALB — 36 fields

`@timestamp` is `col1` (`date_nanos`). The AWS timestamp includes fractional seconds.

| Path   | Name                         | Type         | Notes                                      |
| ------ | ---------------------------- | ------------ | ------------------------------------------ |
| col0   | `type`                       | keyword      |                                            |
| col1   | `@timestamp`                 | date_nanos   |                                            |
| col2   | `elb`                        | keyword      |                                            |
| col3   | `client_port`                | keyword      | `ip:port` — split with DISSECT             |
| col4   | `target_port`                | keyword      | `-` when no target                         |
| col5   | `request_processing_time`    | keyword      | seconds; `-1` or `-`                       |
| col6   | `target_processing_time`     | keyword      | seconds; `-1` or `-`                       |
| col7   | `response_processing_time`   | keyword      | seconds; `-1` or `-`                       |
| col8   | `elb_status_code`            | keyword      |                                            |
| col9   | `target_status_code`         | keyword      | `-` when no target response                |
| col10  | `received_bytes`             | long         |                                            |
| col11  | `sent_bytes`                 | long         |                                            |
| col12  | `request_line`               | keyword      | `"METHOD url protocol"`                    |
| col13  | `user_agent`                 | keyword      | quoted                                     |
| col14  | `ssl_cipher`                 | keyword      |                                            |
| col15  | `ssl_protocol`               | keyword      |                                            |
| col16  | `target_group_arn`           | keyword      |                                            |
| col17  | `trace_id`                   | keyword      |                                            |
| col18  | `domain_name`                | keyword      |                                            |
| col19  | `chosen_cert_arn`            | keyword      |                                            |
| col20  | `matched_rule_priority`      | keyword      | `-` when no rule                           |
| col21  | `request_creation_time`      | keyword      |                                            |
| col22  | `actions_executed`           | keyword      |                                            |
| col23  | `redirect_url`               | keyword      |                                            |
| col24  | `error_reason`               | keyword      |                                            |
| col25  | `target_port_list`           | keyword      |                                            |
| col26  | `target_status_code_list`    | keyword      |                                            |
| col27  | `classification`             | keyword      |                                            |
| col28  | `classification_reason`      | keyword      |                                            |
| col29  | `conn_trace_id`              | keyword      |                                            |
| col30  | `transformed_host`           | keyword      |                                            |
| col31  | `transformed_uri`            | keyword      |                                            |
| col32  | `request_transform_status`   | keyword      |                                            |
| col33  | `ip_address`                 | ip           |                                            |
| col34  | `target_error_code`          | keyword      |                                            |
| col35  | `elb_error_code`             | keyword      |                                            |

```json
{
  "data_source": "<data_source_name>",
  "resource": "s3://<bucket>/<prefix>/*/*/*/*.log.gz",
  "settings": {
    "format": "csv",
    "delimiter": " ",
    "header_row": false,
    "mode": "quoted",
    "schema_resolution": "first_file_wins"
  },
  "mappings": {
    "dynamic": false,
    "properties": {
      "@timestamp": { "type": "date_nanos", "path": "col1" },
      "type": { "type": "keyword", "path": "col0" },
      "elb": { "type": "keyword", "path": "col2" },
      "client_port": { "type": "keyword", "path": "col3" },
      "target_port": { "type": "keyword", "path": "col4" },
      "request_processing_time": { "type": "keyword", "path": "col5" },
      "target_processing_time": { "type": "keyword", "path": "col6" },
      "response_processing_time": { "type": "keyword", "path": "col7" },
      "elb_status_code": { "type": "keyword", "path": "col8" },
      "target_status_code": { "type": "keyword", "path": "col9" },
      "received_bytes": { "type": "long", "path": "col10" },
      "sent_bytes": { "type": "long", "path": "col11" },
      "request_line": { "type": "keyword", "path": "col12" },
      "user_agent": { "type": "keyword", "path": "col13" },
      "ssl_cipher": { "type": "keyword", "path": "col14" },
      "ssl_protocol": { "type": "keyword", "path": "col15" },
      "target_group_arn": { "type": "keyword", "path": "col16" },
      "trace_id": { "type": "keyword", "path": "col17" },
      "domain_name": { "type": "keyword", "path": "col18" },
      "chosen_cert_arn": { "type": "keyword", "path": "col19" },
      "matched_rule_priority": { "type": "keyword", "path": "col20" },
      "request_creation_time": { "type": "keyword", "path": "col21" },
      "actions_executed": { "type": "keyword", "path": "col22" },
      "redirect_url": { "type": "keyword", "path": "col23" },
      "error_reason": { "type": "keyword", "path": "col24" },
      "target_port_list": { "type": "keyword", "path": "col25" },
      "target_status_code_list": { "type": "keyword", "path": "col26" },
      "classification": { "type": "keyword", "path": "col27" },
      "classification_reason": { "type": "keyword", "path": "col28" },
      "conn_trace_id": { "type": "keyword", "path": "col29" },
      "transformed_host": { "type": "keyword", "path": "col30" },
      "transformed_uri": { "type": "keyword", "path": "col31" },
      "request_transform_status": { "type": "keyword", "path": "col32" },
      "ip_address": { "type": "ip", "path": "col33" },
      "target_error_code": { "type": "keyword", "path": "col34" },
      "elb_error_code": { "type": "keyword", "path": "col35" }
    }
  }
}
```

## NLB — 22 fields

`@timestamp` is `col2` (`date`). TLS listener logs can contain additional fields after `col21`. Those fields are not in
the AWS field list this reference maps. `dynamic: false` drops them. Do not add names for them unless the user supplies
the AWS documentation for that log version.

`tls_handshake_time` is milliseconds as text. `-` means the handshake failed. `chosen_cert_arn` is `-` when no
certificate was chosen. The AWS name for `col16` is `tls_keyexchange`.

| Path   | Name                            | Type    |
| ------ | ------------------------------- | ------- |
| col0   | `type`                          | keyword |
| col1   | `version`                       | keyword |
| col2   | `@timestamp`                    | date    |
| col3   | `elb`                           | keyword |
| col4   | `listener`                      | keyword |
| col5   | `client_port`                   | keyword |
| col6   | `destination_port`              | keyword |
| col7   | `connection_time`               | keyword |
| col8   | `tls_handshake_time`            | keyword |
| col9   | `received_bytes`                | long    |
| col10  | `sent_bytes`                    | long    |
| col11  | `incoming_tls_alert`            | keyword |
| col12  | `chosen_cert_arn`               | keyword |
| col13  | `chosen_cert_serial`            | keyword |
| col14  | `tls_cipher`                    | keyword |
| col15  | `tls_protocol_version`          | keyword |
| col16  | `tls_keyexchange`               | keyword |
| col17  | `domain_name`                   | keyword |
| col18  | `alpn_fe_protocol`              | keyword |
| col19  | `alpn_be_protocol`              | keyword |
| col20  | `alpn_client_preference_list`   | keyword |
| col21  | `tls_connection_creation_time`  | keyword |

```json
{
  "data_source": "<data_source_name>",
  "resource": "s3://<bucket>/<prefix>/*/*/*/*.log.gz",
  "settings": {
    "format": "csv",
    "delimiter": " ",
    "header_row": false,
    "mode": "plain",
    "schema_resolution": "first_file_wins"
  },
  "mappings": {
    "dynamic": false,
    "properties": {
      "type": { "type": "keyword", "path": "col0" },
      "version": { "type": "keyword", "path": "col1" },
      "@timestamp": { "type": "date", "path": "col2" },
      "elb": { "type": "keyword", "path": "col3" },
      "listener": { "type": "keyword", "path": "col4" },
      "client_port": { "type": "keyword", "path": "col5" },
      "destination_port": { "type": "keyword", "path": "col6" },
      "connection_time": { "type": "keyword", "path": "col7" },
      "tls_handshake_time": { "type": "keyword", "path": "col8" },
      "received_bytes": { "type": "long", "path": "col9" },
      "sent_bytes": { "type": "long", "path": "col10" },
      "incoming_tls_alert": { "type": "keyword", "path": "col11" },
      "chosen_cert_arn": { "type": "keyword", "path": "col12" },
      "chosen_cert_serial": { "type": "keyword", "path": "col13" },
      "tls_cipher": { "type": "keyword", "path": "col14" },
      "tls_protocol_version": { "type": "keyword", "path": "col15" },
      "tls_keyexchange": { "type": "keyword", "path": "col16" },
      "domain_name": { "type": "keyword", "path": "col17" },
      "alpn_fe_protocol": { "type": "keyword", "path": "col18" },
      "alpn_be_protocol": { "type": "keyword", "path": "col19" },
      "alpn_client_preference_list": { "type": "keyword", "path": "col20" },
      "tls_connection_creation_time": { "type": "keyword", "path": "col21" }
    }
  }
}
```

## CLB — 15 fields

`@timestamp` is `col0` (`date_nanos`). The AWS name for `col1` is `elb`, not `elb_name`. The AWS name for `col11` is
`request`, not `request_line`. Timing fields use the same `-` and `-1` sentinels as ALB.

| Path   | Name                       | Type       |
| ------ | -------------------------- | ---------- |
| col0   | `@timestamp`               | date_nanos |
| col1   | `elb`                      | keyword    |
| col2   | `client_port`              | keyword    |
| col3   | `backend_port`             | keyword    |
| col4   | `request_processing_time`  | keyword    |
| col5   | `backend_processing_time`  | keyword    |
| col6   | `response_processing_time` | keyword    |
| col7   | `elb_status_code`          | keyword    |
| col8   | `backend_status_code`      | keyword    |
| col9   | `received_bytes`           | long       |
| col10  | `sent_bytes`               | long       |
| col11  | `request`                  | keyword    |
| col12  | `user_agent`               | keyword    |
| col13  | `ssl_cipher`               | keyword    |
| col14  | `ssl_protocol`             | keyword    |

```json
{
  "data_source": "<data_source_name>",
  "resource": "s3://<bucket>/<prefix>/*/*/*/*.log",
  "settings": {
    "format": "csv",
    "delimiter": " ",
    "header_row": false,
    "mode": "quoted",
    "schema_resolution": "first_file_wins"
  },
  "mappings": {
    "dynamic": false,
    "properties": {
      "@timestamp": { "type": "date_nanos", "path": "col0" },
      "elb": { "type": "keyword", "path": "col1" },
      "client_port": { "type": "keyword", "path": "col2" },
      "backend_port": { "type": "keyword", "path": "col3" },
      "request_processing_time": { "type": "keyword", "path": "col4" },
      "backend_processing_time": { "type": "keyword", "path": "col5" },
      "response_processing_time": { "type": "keyword", "path": "col6" },
      "elb_status_code": { "type": "keyword", "path": "col7" },
      "backend_status_code": { "type": "keyword", "path": "col8" },
      "received_bytes": { "type": "long", "path": "col9" },
      "sent_bytes": { "type": "long", "path": "col10" },
      "request": { "type": "keyword", "path": "col11" },
      "user_agent": { "type": "keyword", "path": "col12" },
      "ssl_cipher": { "type": "keyword", "path": "col13" },
      "ssl_protocol": { "type": "keyword", "path": "col14" }
    }
  }
}
```
