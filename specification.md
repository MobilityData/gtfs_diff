# GTFS Diff Specification

This document serves as the index for all versions of the GTFS Diff specification.

| Version | Format | Status | Description |
|---------|--------|--------|-------------|
| [v1](spec/v1/specification.md) | CSV | Stable | Original 8-column CSV format for expressing differences between two GTFS files. |
| [v2](spec/v2/specification.md) | JSON | Draft | Structured JSON format with summary, per-file diffs, row capping, and metadata. |

## Choosing a version

- **v1** is the original format, best suited for simple diffs that are easy to read in a spreadsheet or apply as patches.
- **v2** is designed for richer tooling and UIs: it provides aggregate summaries, capped row details for performance, explicit primary keys, and metadata about feed sources and unsupported files.

Both versions describe the same conceptual changes (file, column, and row-level diffs between two GTFS archives) but use different serialization formats and levels of detail.
