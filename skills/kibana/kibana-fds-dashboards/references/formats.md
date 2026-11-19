# FDS File Formats

Resolve the format before writing ES|QL. The chart rules in `SKILL.md` apply to every format. The traps below do not.

## Extensions

| Format  | Extensions                   | Schema source                | Whole-file compression                         |
| ------- | ---------------------------- | ---------------------------- | ---------------------------------------------- |
| Parquet | `.parquet`, `.parq`          | File metadata                | None. Compression is inside each column chunk. |
| NDJSON  | `.ndjson`, `.jsonl`, `.json` | Sampled lines, default 20000 | `.gz`, `.gzip`, `.zst`, `.zstd`                |
| CSV     | `.csv`                       | Sampled rows, default 40000  | `.gz`, `.gzip`, `.zst`, `.zstd`                |
| TSV     | `.tsv`                       | Sampled rows, default 40000  | `.gz`, `.gzip`, `.zst`, `.zstd`                |

A trailing compression suffix is not a format. `clicks.csv.gz` is CSV.

`settings.format` wins when it is set to `parquet`, `ndjson`, `csv`, or `tsv`. `auto`, or an absent value, means the
resource pattern must imply exactly one format. A pattern such as `*.{parquet,csv}` does not. A file whose extension
maps to a different registered format is rejected, not skipped.

## Format Override

On Elasticsearch 9.6+, set `settings.format` to `"csv"` or `"tsv"` when the files use an extension the reader does not
recognise, such as `.log` or `.log.gz`. The override forces that reader. Confirm with a bounded test query. On
Elasticsearch before 9.6, `.log` and `.log.gz` are not supported — stop.

Space-delimited access logs still need the CSV settings below. `format: "csv"` does not change the delimiter.

## CSV and TSV Settings

| Setting      | Default                  | When to set it                                                            |
| ------------ | ------------------------ | ------------------------------------------------------------------------- |
| `delimiter`  | `,` for CSV, tab for TSV | `" "` for space-delimited logs. One character.                            |
| `mode`       | reader default           | `"quoted"` when quotes wrap a field, otherwise `"plain"`.                |
| `header_row` | `true`                   | `false` for headerless files. Columns are `col0`, `col1`, unless mapped. |
| `null_value` | unset                    | Token that should read as null. Until set, `-` and `NA` stay strings.    |

A headerless file with `"dynamic": false` can rename `colN` through `mappings.properties.<name>.path`. That mapping
belongs to dataset registration. In the panel query, use the mapped name.

## What to Read from the Dataset

Every format:

- `settings.schema_resolution`: `first_file_wins` (one file supplies the schema), `union_by_name` (merge by name;
  incompatible types become `keyword`), or `strict` (files must agree).
- `mappings.properties`: declared columns override inference. `@timestamp` needs `type: date`, `path`, and `format` when
  the file column is not already a timestamp named `@timestamp`.

CSV and TSV:

- `header_row` (default `true`). `false` names columns `col0`, `col1`, by position. `column_prefix` changes the prefix.
- `delimiter` (default `,` or tab).
- `null_value` (default unset). No token reads as null until this is set.
- `schema_sample_size` when set.

NDJSON:

- `schema_sample_size` when set. Nested objects become dotted names (`user.id`).
- A dotted name matches either a nested key or a flat key with that spelling.

## Traps that Change the Query

- **Text types are provisional.** A column that looks numeric can be `keyword` because of one non-numeric token, or
  because `union_by_name` widened a conflict. Cast with `TO_LONG` or `TO_DOUBLE` before `SUM`, `AVG`, or a numeric
  aggregation. When the token can be `-` or `-1`, guard the cast with `CASE` as in the sentinel recipe. Parquet numeric
  types come from the file and do not need that cast.
- **A sampled column can be missing.** With `union_by_name` or `strict`, each file only contributes rows up to its share
  of `schema_sample_size`. A column that first appears later is not in the schema. With `first_file_wins`, columns that
  exist only in later files are dropped.
- **CSV nulls are strings.** Unless `null_value` is set, `NA`, `NULL`, and `-` are ordinary values. Filter them with
  `==` / `!=`, not `IS NULL`.
- **Parquet type mismatch under `first_file_wins`.** A later file whose type cannot be read as the first file's type
  returns nulls for that column and a warning. The query still succeeds. A declared mapping changes that: the dataset
  `error_mode` decides.
- **Date type is a dataset mapping.** A keyword time column is not fixed by `TO_DATETIME` in the chart. `partition_spec`
  and the time picker need `@timestamp` mapped as `date`.
