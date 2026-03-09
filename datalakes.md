# Data Lakes

## Terms

### Parquet file

  A columnar binary file format designed for efficient storage and querying of large datasets.

  Key characteristics:
  - Columnar: data is stored column-by-column (not row-by-row), so reading just the status column of 1M tickets
  doesn't require reading name, description, etc.
  - Compressed: supports codecs like zstd (which this TDD uses)
  - Self-describing: the file contains its own schema
  - Immutable: once written, you don't edit a parquet file — you write a new one

  In this TDD, each sync produces files like tickets.parquet, users.parquet, etc., stored in S3. They're the "data"
   layer — DuckLake then stores metadata about those files in a .duckdb catalog, enabling SQL queries, time-travel,
   and schema evolution on top.

  Think of it as: parquet = the raw data files, .duckdb catalog = the index/metadata that makes them queryable as a
   database.

### Bloom Filters

  A bloom filter is a compact data structure stored inside the parquet file that answers the question: "is this
  value definitely NOT in this column?"

  Example: you query WHERE ticket_id = 'abc-123'. The bloom filter can tell you instantly — without reading the
  actual data — "that value is definitely not in this row group, skip it." It can't tell you yes, only no (it has
  false positives but no false negatives).

  Result: you skip entire chunks of the file without reading them.

### Filter Pushdown

  Normally you might imagine: read all the data into memory, then filter it. Filter pushdown flips this — the
  filter is "pushed down" into the file reader itself, so only matching rows are ever loaded.

  Example: WHERE status = 'open' — instead of reading all tickets and then discarding closed ones, the parquet
  reader applies the filter while reading and never materializes the closed rows at all.
