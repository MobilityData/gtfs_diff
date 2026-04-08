# GTFS Diff Specification v2

A GTFS file is a ZIP archive containing a number of CSV files. A "GTFS Diff v2" file is a single JSON document that describes all differences between two GTFS archives, combining aggregate summaries with capped per-file details.

## Design goals

1. **Single file output**: one JSON document contains summary + capped details.
2. **Capped for performance**: max 50 row changes per file; full counts always preserved in summary.
3. **Explicit scope**: files outside the supported scope are reported in `metadata.unsupported_files` rather than silently ignored.
4. **Spec-aligned**: compatible with the existing [GTFS Diff v1 specification](../v1/specification.md).

## Supported scope (v2.0)

This version of the schema only supports files defined in the official [GTFS Schedule reference](https://gtfs.org/documentation/schedule/reference/).
Any unsupported file in the GTFS archive (including non-`.txt` files like `readme.pdf`, `locations.geojson`, etc.) is **not diffed**. Instead, it is reported in `metadata.unsupported_files`.

## Schema overview

```
GtfsDiffOutput
├── metadata
│   ├── schema_version
│   ├── generated_at
│   ├── row_changes_cap_per_file
│   ├── base_feed               # source (URL or local path), downloaded_at
│   ├── new_feed                # source (URL or local path), downloaded_at
│   └── unsupported_files[]     # files skipped by the diff engine
├── summary                     # true aggregate counts (drives file tree sidebar)
│   └── files[]                 # per-file: name + true counts by action
└── file_diffs[]                # one entry per changed supported file
    ├── file_name
    ├── file_action             # "added" | "deleted" | "modified"
    ├── columns_added[]
    ├── columns_deleted[]
    ├── row_changes
    │   ├── primary_key[]
    │   ├── columns[]           # union of base + new; base column order first
    │   ├── added[]             # capped
    │   ├── deleted[]           # capped
    │   └── modified[]          # capped
    └── truncated               # cap metadata (omitted counts)
```

## Fields description

### `metadata`

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `schema_version` | String | Required | The version of this schema (e.g. `"2.0.0"`). |
| `generated_at` | String (ISO 8601) | Required | Timestamp of when the diff was generated. |
| `row_changes_cap_per_file` | Integer | Required | Maximum number of row changes included per file in `file_diffs`. |
| `base_feed` | Object | Required | Information about the base (old) GTFS archive. |
| `base_feed.source` | String | Required | URL or local path to the base feed. |
| `base_feed.downloaded_at` | String (ISO 8601) | Required | When the base feed was downloaded. |
| `new_feed` | Object | Required | Information about the new GTFS archive. |
| `new_feed.source` | String | Required | URL or local path to the new feed. |
| `new_feed.downloaded_at` | String (ISO 8601) | Required | When the new feed was downloaded. |
| `unsupported_files` | Array | Required | List of files in the archives that are outside the supported scope. |
| `unsupported_files[].file_name` | String | Required | File name as it appears in the archive. |
| `unsupported_files[].present_in` | String. Enum: `base`, `new`, `both` | Required | Which archive(s) contain this file. |

### `summary`

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `total_changes` | Integer | Required | Total number of changes across all files. |
| `files_added` | Integer | Required | Number of files added. |
| `files_deleted` | Integer | Required | Number of files deleted. |
| `files_modified` | Integer | Required | Number of files modified. |
| `files` | Array | Required | Per-file summary with true (uncapped) counts. |
| `files[].file_name` | String | Required | Name of the GTFS file. |
| `files[].status` | String. Enum: `added`, `deleted`, `modified` | Required | The file-level status. |
| `files[].columns_added` | Integer | Optional | Number of columns added. Present when > 0. |
| `files[].columns_deleted` | Integer | Optional | Number of columns deleted. Present when > 0. |
| `files[].rows_added` | Integer | Optional | True count of rows added. Present when > 0. |
| `files[].rows_deleted` | Integer | Optional | True count of rows deleted. Present when > 0. |
| `files[].rows_modified` | Integer | Optional | True count of rows modified. Present when > 0. |

### `file_diffs[]`

Each entry represents one changed supported file.

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `file_name` | String | Required | Name of the GTFS file. |
| `file_action` | String. Enum: `"added"`, `"deleted"`, `"modified"` | Required | How this file changed between the two archives. |
| `columns_added` | Array of String | Required | List of column names added to this file. |
| `columns_deleted` | Array of String | Required | List of column names deleted from this file. |
| `row_changes` | Object | Conditionally required | Present when the file has row-level changes (i.e. `file_action` is `"modified"`). |
| `row_changes.primary_key` | Array of String | Required | Column(s) that uniquely identify a row in this file. |
| `row_changes.columns` | Array of String | Required | Union of all columns across both the base and new versions of the file. The order matches the base feed's original column order; columns only present in the new feed are appended at the end. |
| `row_changes.added` | Array | Required | Added rows (capped). Each entry has `identifier` (primary key values), `raw_value` (the CSV row from the new file, field order matching `columns`), and `new_line_number` (1-based line number in the new CSV file). |
| `row_changes.deleted` | Array | Required | Deleted rows (capped). Each entry has `identifier` (primary key values), `raw_value` (the CSV row from the base file, field order matching `columns`), and `base_line_number` (1-based line number in the base CSV file). |
| `row_changes.modified` | Array | Required | Modified rows (capped). Each entry has `identifier` (primary key values), `raw_value` (the base CSV row, field order matching `columns`), `base_line_number` (1-based line number in the base CSV file), `new_line_number` (1-based line number in the new CSV file), and `field_changes` (array of `{field, base_value, new_value}`). |
| `truncated` | Object | Optional | Present only when row changes exceed the cap. |
| `truncated.is_truncated` | Boolean | Required | Always `true` when present. |
| `truncated.omitted_count` | Integer | Required | Number of row changes omitted due to the cap. |

## Capping behavior

The cap applies to the **combined** count of `added + deleted + modified` rows per file, in first-encountered order. A file with 30 added, 15 deleted, and 200 modified rows (245 total) hits the cap at 50 and reports `omitted_count: 195`.

File-level changes (`file_action`) and column-level changes (`columns_added`, `columns_deleted`) are **not capped**.

## Unsupported files behavior

When the diff engine encounters a file that is not in the supported scope:

1. It is **not** added to `file_diffs[]`.
2. It is **not** counted in `summary.total_changes` or `summary.files_*`.
3. It is listed in `metadata.unsupported_files[]` with `file_name` and `present_in`.

This keeps the diff focused on what was actually compared, while still surfacing skipped files so the UI can show a "files not diffed" section.

## JSON Schema

A formal JSON Schema for validation is available at [`json_schema/gtfs_diff_v2_schema.json`](json_schema/gtfs_diff_v2_schema.json).

## Example

See [`examples/example_output.json`](examples/example_output.json) for a full example, and below for a walkthrough.

### Walkthrough

Given two GTFS archives where:
- `shapes.txt` was added as a new file
- `stop_times.txt` had 1213 row changes (120 added, 45 deleted, 1048 modified)
- `stops.txt` had a column deleted and 8 row changes (2 added, 1 deleted, 5 modified)
- `readme.pdf` and `custom_notes.txt` are non-GTFS files present in the archives

The v2 output will:
- Report `readme.pdf` and `custom_notes.txt` in `metadata.unsupported_files`
- Show true counts in `summary` (total_changes: 1213)
- Cap `stop_times.txt` row details to 50, reporting `omitted_count: 1163`
- Show all 8 `stops.txt` row changes (under the cap, no `truncated` field)
- Show `shapes.txt` as a file-level addition with no `row_changes`

## Full example

The [examples](examples/) folder contains a complete example output.
