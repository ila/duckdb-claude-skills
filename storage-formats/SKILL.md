---
name: storage-formats
description: Reference for data lake storage formats and table formats — Parquet, Delta Lake, Apache Iceberg, Apache Hudi, DuckLake, Lance, F3 (Faster Frozen Format), ORC, and their interaction with DuckDB. Auto-loaded when discussing storage formats, data lakes, table formats, Parquet, Iceberg, Delta Lake, DuckLake, or change data capture.
---

# Storage Formats and Table Formats Reference

Comprehensive reference for columnar file formats, open table formats, and their
relevance to DuckDB extension development.

---

## 1. Columnar File Formats

### 1.1 Apache Parquet

**The dominant open columnar format.** Self-describing, column-oriented, designed for
analytics workloads. Created by Twitter + Cloudera (2013), now Apache top-level project.

**Physical layout:**
```
File:
  Row Group 0 (typically 128MB - 1GB):
    Column Chunk (col_a): Page 0 | Page 1 | ... | Page N
    Column Chunk (col_b): Page 0 | Page 1 | ...
    ...
  Row Group 1:
    ...
  Footer (schema, row group metadata, column statistics)
```

**Key features:**
- **Row groups:** Horizontal partitions (~128MB-1GB each). Unit of parallel read.
- **Column chunks:** One per column per row group. Unit of I/O.
- **Pages:** Within a column chunk (~1MB). Unit of compression/encoding.
- **Encodings:** PLAIN, DICTIONARY, RLE, DELTA_BINARY_PACKED, DELTA_LENGTH_BYTE_ARRAY,
  DELTA_BYTE_ARRAY, BYTE_STREAM_SPLIT. Encoding chosen per page.
- **Compression:** Snappy (default), ZSTD (best ratio), LZ4, Gzip, Brotli. Per-column.
- **Statistics:** Min/max per column chunk and page. Enables predicate pushdown.
- **Nested types:** Via Dremel encoding (repetition + definition levels).
- **Bloom filters:** Optional per-column. Probabilistic membership test for point lookups.
- **Page index:** Column index + offset index for fine-grained page skipping.

**DuckDB integration:** Native Parquet reader/writer. `read_parquet()`, `COPY TO`,
`parquet_metadata()`, `parquet_schema()`. Supports filter pushdown, projection pushdown,
row group pruning via statistics. Parquet is DuckDB's primary external format.

**Limitations:**
- Immutable — no in-place updates. Modifications require rewriting entire files.
- No transactions, no schema evolution, no time travel (these are table format concerns).
- Row group boundaries are fixed at write time — suboptimal for evolving data.

### 1.2 Apache ORC (Optimized Row Columnar)

Created by Hortonworks for Hive (2013). Similar to Parquet but with differences:

- **Stripes** (= Parquet's row groups), typically 256MB
- **Lightweight indexes:** Min/max/sum/count per stripe and per 10K-row "row index entry"
- **ACID support:** Built-in support for delta files (base + side files for updates)
- **Better compression** for many workloads (more aggressive encoding selection)
- **Worse ecosystem reach** than Parquet (Hive-centric)

DuckDB supports ORC reading via community extensions but Parquet is preferred.

### 1.3 Lance (Lance Format v2)

Modern columnar format designed for ML/AI workloads. Created by LanceDB (2023+).

- **Random access:** O(1) row lookup by ID (unlike Parquet which requires scanning)
- **Vector search:** Native support for ANN (approximate nearest neighbor) indexes
- **Fast updates:** Append-only log with deletion vectors (no file rewrite for deletes)
- **Versioning:** Built-in dataset versioning via manifest chain
- **Encodings optimized for embeddings:** Flat, IVF-PQ, product quantization

**DuckDB integration:** `lance` community extension.

### 1.4 F3 (Faster Frozen Format)

**Research format from CWI (DuckDB's birthplace).** Designed for fast analytical queries
on frozen (immutable) data.

**Key ideas:**
- **Adaptive encoding:** Per-column, per-block encoding selection based on data distribution
- **Lightweight compression:** Prioritize decompression speed over compression ratio
- **Vectorized scanning:** Format designed to feed directly into vectorized execution
  engines (like DuckDB's) without transcoding
- **Zone maps:** Per-block min/max for predicate pushdown
- **Cache-line alignment:** Data layout respects CPU cache line boundaries

F3 achieves faster scan performance than Parquet by reducing decompression overhead,
at the cost of slightly larger file sizes.

**Relevance to DuckDB:** DuckDB's native storage format is influenced by F3's design
philosophy — adaptive encoding, lightweight compression, vectorized-friendly layout.

### 1.5 FastLanes Compression Layout and File Format

**Afroozeh, Boncz.** "The FastLanes Compression Layout: Decoding >100 Billion Integers
per Second with Scalar Code." PVLDB 16(9), 2023.
**Afroozeh, Boncz.** "The FastLanes File Format." PVLDB 18(11), 2025.

FastLanes is both a **compression layout** and a **next-gen file format** from CWI.

**The core problem:** Lightweight compression (LWC) schemes — bit-packing, FOR, DELTA,
RLE, dictionary — are not inherently SIMD-friendly. DELTA decoding is a prefix-sum
(sequential). Traditional implementations suffer from data dependencies.

**Innovation 1: Virtual 1024-bit SIMD Register (MM1024).**
FastLanes processes blocks of 1024 values as if the hardware had a single 1024-bit
register. Data organized into T rows of (1024/T) lanes (T = element bit-width).
No data dependencies between lanes. The layout auto-vectorizes to ANY SIMD width —
128-bit (SSE), 256-bit (AVX2), 512-bit (AVX-512), or NEON — using identical scalar code.

**Innovation 2: Unified Transposed Layout (UTL).**
For inherently sequential encodings (DELTA), FastLanes physically reorders tuples
within each 1024-value block using a specific permutation ("04261537" order). After
transposition, adjacent elements in a SIMD word come from **independent DELTA chains**,
eliminating cross-lane dependencies regardless of SIMD width.

**Supported encodings:**
- **Bit-Packing:** Fully parallel unpacking across all lanes
- **Frame-of-Reference (FOR):** Base value + bit-packed residuals
- **DELTA:** Independent prefix-sum streams via UTL (no sequential dependency)
- **RLE:** Mapped to DELTA+DICT to leverage efficient DELTA kernel
- **Dictionary:** Bit-packed dictionary codes via interleaved layout

**Performance:** Decoding speed exceeds 40-60 values per CPU cycle single-threaded.
Over 100 billion integers/second. Scalar auto-vectorized code often matches or
exceeds hand-written SIMD intrinsics.

**FastLanes File Format (2025):** A complete file format (Parquet successor) built on
the compression layout. Adds multi-column compression (MCC) via expression-based
cascaded encodings — columns that correlate can be jointly compressed.

**GitHub:** [cwida/FastLanes](https://github.com/cwida/FastLanes)

### 1.6 ALP (Adaptive Lossless floating-Point Compression)

**Afroozeh, Kuffo, Boncz.** "ALP: Adaptive Lossless floating-Point Compression."
SIGMOD 2024. Best Artifact Award.

**The problem:** IEEE 754 doubles have pseudo-random bit patterns because most decimal
values (10.12, 3.14) can't be exactly represented in binary. Traditional integer-oriented
compression (DELTA, FOR) fails. XOR-based schemes (Gorilla, Chimp, Patas) have complex
branching that defeats SIMD.

**ALP algorithm (for decimal-origin floats — the common case):**
1. **Sample** values from row-group to find best exponent `e` and factor `f`
2. **Encode:** Multiply each double by 10^e, round to nearest integer
3. **Verify losslessness:** Decode and check exact equality with original
4. Values that don't round-trip exactly → exceptions (stored separately with positions)
5. Successful integers compressed via **FastLanes FOR + Bit-Packing**

Example: `10.12` with e=2 → `10.12 * 100 = 1012` → FOR + bit-packed as integer.

**ALP-RD (Real Doubles — for non-decimal floats):**
For scientific/computed data where the decimal trick fails:
1. **Bit-split** each 64-bit IEEE 754 representation at position `p`
2. **Left part** (high bits): dictionary-encoded (few distinct patterns)
3. **Right part** (low bits): bit-packed to exactly `64-p` bits
4. Split position `p` chosen at row-group level to minimize left-part variance

**Performance vs alternatives (41 real-world datasets):**
- Compression ratio: ALP ~4.3x avg. Patas ~2.1x. Chimp ~2.4x. Comparable to Zstd.
- Decompression: ALP ~2.6 doubles/cycle. ~8x faster than Patas. ~55x faster than Gorilla.
- SUM query on ALP data: only 1.4x slower than uncompressed (vs 3.7x Patas, 7.7x Chimp)

**DuckDB integration:** Integrated in DuckDB v0.10.0 (Feb 2024). Replaces both Chimp
and Patas as the floating-point compression scheme. Uses DuckDB's two-phase
analyze/compress architecture. Vectors of 1024 values.

**GitHub:** [cwida/ALP](https://github.com/cwida/ALP)

---

## 2. Open Table Formats

Table formats add ACID transactions, schema evolution, time travel, and change tracking
ON TOP of file formats (typically Parquet). They manage metadata about which files
constitute the current table state.

### 2.1 Delta Lake

**Created by Databricks (2019).** The most widely deployed open table format.

**Architecture:**
```
table_path/
  _delta_log/                    # Transaction log (JSON + Parquet checkpoints)
    00000000000000000000.json    # Version 0: initial table creation
    00000000000000000001.json    # Version 1: add files
    00000000000000000010.checkpoint.parquet  # Checkpoint every 10 versions
  part-00000-....parquet         # Data files
  part-00001-....parquet
```

**Transaction log:** Sequence of JSON actions (AddFile, RemoveFile, Metadata, Protocol).
Each commit = one JSON file. Optimistic concurrency via file-system atomic rename.
Checkpoints (Parquet) consolidate log periodically for fast reads.

**Key features:**
- **ACID transactions:** Serializable isolation via optimistic concurrency control
- **Time travel:** `SELECT * FROM table VERSION AS OF 5` or `TIMESTAMP AS OF '2024-01-01'`
- **Schema evolution:** Add/rename/drop columns, type widening
- **Change Data Feed (CDF):** Tracks row-level changes (insert/update/delete) between versions.
  Stored as separate `_change_data/` Parquet files with `_change_type` column.
- **Deletion vectors:** Bitmap marking deleted rows within a Parquet file (avoids rewrite)
- **Liquid clustering:** Replaces static partitioning with adaptive data layout
- **UniForm:** Auto-generates Iceberg metadata alongside Delta metadata for interop

**DuckDB integration:** `delta` extension. `SELECT * FROM delta_scan('path')`.
Supports time travel, predicate pushdown, CDF reading.

**Relevance to IVM:** Delta Lake's Change Data Feed is essentially a delta table —
it tracks exactly which rows were inserted/updated/deleted between versions. This maps
directly to OpenIVM's `delta_<table>` concept. An IVM system on top of Delta Lake
could read CDF instead of maintaining separate delta tables.

### 2.2 Apache Iceberg

**Created by Netflix (2017), now Apache top-level project.** Emphasis on correctness,
large-scale tables (petabyte+), and multi-engine compatibility.

**Architecture:**
```
metadata/
  v1.metadata.json              # Metadata file: schema, partition spec, snapshot list
  v2.metadata.json
  snap-1234.avro                # Manifest list: points to manifest files
  manifest-abc.avro             # Manifest: list of data files with column stats
data/
  part-00000.parquet            # Data files (Parquet, ORC, or Avro)
```

**Three-level metadata:**
1. **Metadata file:** Schema, partition spec, sort order, current snapshot pointer
2. **Manifest list:** Points to manifest files, with partition-level summary stats
3. **Manifest files:** List individual data files with per-file, per-column statistics

This hierarchy enables O(1) snapshot reads and efficient partition pruning even for
tables with millions of files.

**Key features:**
- **Hidden partitioning:** Partition transforms (year, month, day, hour, bucket, truncate)
  applied transparently. Users query without knowing the partition scheme.
- **Schema evolution:** Full support (add, drop, rename, reorder, type promotion).
  Each column has a unique ID; renames don't break readers.
- **Time travel:** Via snapshot IDs or timestamps
- **Incremental reads:** `IncrementalAppendScan` API reads only new files added since
  a given snapshot. Basis for streaming/IVM use cases.
- **Row-level deletes:** Two modes:
  - Copy-on-write: Rewrite affected data files
  - Merge-on-read: Write delete files (positional or equality) alongside data files
- **Branching/tagging:** Git-like branching of table state (Iceberg v2)

**DuckDB integration:** `iceberg` extension. `SELECT * FROM iceberg_scan('path')`.
Supports snapshot selection, metadata inspection.

**Relevance to IVM:** Iceberg's incremental scan API (read only new files since snapshot X)
is a natural fit for delta detection. Combined with merge-on-read delete files, it
provides the "before and after" information needed for IVM delta computation.

### 2.3 Apache Hudi (Hadoop Upserts Deletes Incrementals)

**Created by Uber (2016).** Designed specifically for incremental data processing.

**Two table types:**
- **Copy-on-Write (CoW):** Updates rewrite entire Parquet files. Read-optimized.
- **Merge-on-Read (MoR):** Updates written to row-based log files, merged on read. Write-optimized.

**Key features:**
- **Upsert primitive:** First-class support for UPSERT (insert or update by key)
- **Incremental queries:** `SELECT * FROM table WHERE _hoodie_commit_time > X`
  Returns only rows that changed since commit X.
- **Record-level indexing:** Bloom filter, HBase, or bucket index for fast record lookup
- **Compaction:** Background merging of log files into base files (MoR → CoW)
- **Clustering:** Re-organize data layout for query performance

**Relevance to IVM:** Hudi was designed from the ground up for incremental data pipelines.
Its incremental query API directly provides delta rows. The record-level index enables
efficient point lookups during delta application (like probing the MV during MERGE).

### 2.4 DuckLake

**DuckDB Labs (2025).** A new open table format that stores metadata in a DuckDB
database file rather than in the data lake filesystem.

**Architecture:**
```
ducklake.db                     # Metadata database (DuckDB file)
  tables/                       # Table metadata, schemas, snapshots
  data_files/                   # File registry with column statistics
data/
  *.parquet                     # Data files (Parquet)
```

**Key design decisions:**
- **Metadata in DuckDB:** Instead of JSON/Avro metadata files on object storage (Iceberg/Delta),
  DuckLake stores all metadata in a DuckDB database. This enables:
  - SQL queries over metadata (`SELECT * FROM ducklake_table_info(...)`)
  - ACID transactions on metadata via DuckDB's MVCC
  - Sub-millisecond metadata operations (no S3 round-trips for metadata)
- **Data in Parquet:** Actual data stored as Parquet files on any storage backend
- **Catalog integration:** Registers as a DuckDB catalog — tables appear as regular tables

**Key features:**
- **Instant table creation/schema changes:** Metadata-only operations, no file I/O
- **Change tracking:** Built-in support for tracking which data files were added/removed
  per transaction — the metadata DB records the full history
- **Snapshot isolation:** Via DuckDB's MVCC on the metadata database
- **Multi-engine access:** Metadata DB can be read by any system that reads DuckDB files;
  data files are standard Parquet

**Relevance to IVM:** DuckLake's metadata-in-DuckDB design means IVM metadata (delta tables,
view definitions, refresh timestamps) could be co-located with table format metadata in
a single DuckDB file. Change tracking is a metadata query, not a filesystem scan.

---

## 3. Change Data Capture (CDC) Across Formats

CDC is the mechanism for detecting and capturing data changes — the foundation for
any IVM system that reads from external tables.

| Format | CDC Mechanism | Granularity | API |
|---|---|---|---|
| **Delta Lake** | Change Data Feed | Row-level (insert/update_pre/update_post/delete) | `table_changes()` |
| **Iceberg** | Incremental scan + delete files | File-level (new files) + row-level (deletes) | `IncrementalAppendScan` |
| **Hudi** | Incremental query | Row-level via commit timeline | `_hoodie_commit_time > X` |
| **DuckLake** | Metadata transaction log | File-level (added/removed per txn) | SQL on metadata DB |
| **DuckDB native** | WAL / delta tables | Row-level | OpenIVM's `delta_<table>` |

**For IVM systems:** The ideal CDC mechanism provides:
1. Row-level granularity (which rows changed)
2. Before+after images (old and new values for updates)
3. Monotonic ordering (changes in commit order)
4. Efficient access (no full scan of unchanged data)

Delta Lake's CDF is closest to this ideal. Iceberg's incremental scan provides
file-level granularity; row-level requires comparing file contents.

---

## 4. Storage Format Implications for IVM

### 4.1 Immutability and Delta Tables

All modern formats are append-oriented (Parquet files are immutable). This means:
- **Inserts:** Append new Parquet file. Delta detection: new files since last snapshot.
- **Deletes:** Write deletion vector or delete file. Delta detection: scan delete files.
- **Updates:** Delete + insert (CoW) or log entry (MoR). Delta detection: depends on mode.

For IVM, the CoW model is simpler (each new file is a self-contained delta), but MoR
is more write-efficient. The choice affects how delta tables are populated.

### 4.2 Column Statistics for Cost Models

All formats provide per-file and per-column statistics (min, max, count, null_count).
These statistics can feed IVM cost models:
- Delta size estimate: count of rows in new/changed files
- Base table size: total row count across all active files
- Selectivity estimate: min/max overlap between delta and base table columns

### 4.3 Partitioning and IVM

Table-level partitioning (by date, region, etc.) enables partition-scoped IVM:
- If delta only affects partitions [P1, P3], the MV refresh can scope to those partitions
- For partition-level aggregates, unchanged partitions can be skipped entirely
- Iceberg's hidden partitioning is particularly clean for this

### 4.4 Time Travel and IVM Correctness

Time travel (reading table state at a past point in time) enables:
- **Consistent snapshots:** IVM refresh reads all base tables at the same logical time
- **Debugging:** Compare MV state with full recompute at any historical point
- **Recovery:** Roll back to a known-good state if IVM produces incorrect results

---

## 5. DuckDB Storage Internals

DuckDB's native storage (`.duckdb` files) uses its own format:

**Block-based storage:**
- Fixed-size blocks (256KB default)
- Row groups (~122K rows each)
- Per-column segments within row groups
- Adaptive encoding per segment (constant, dictionary, RLE, bitpacking, FSST for strings)

**Compression:** Chimp (floating point), ALP (floating point), FSST (strings),
bitpacking (integers), dictionary, RLE, constant.

**Buffer manager:** LRU-based with memory limit. Spills to disk when memory is exceeded.

**MVCC:** Multi-version concurrency control for transactions. Undo buffer tracks
per-row version chains.

**WAL:** Write-ahead log for crash recovery. Flushed on commit.

**Relevance to IVM:** DuckDB's row group structure means IVM delta tables are stored
in the same row-group format as base tables. The MVCC undo buffer could theoretically
be tapped for change detection (instead of separate delta tables), though this would
require deep engine integration.

---

## 6. References

- Apache Parquet: https://parquet.apache.org/documentation/latest/
- Apache Iceberg: https://iceberg.apache.org/docs/latest/
- Delta Lake: https://docs.delta.io/latest/
- Apache Hudi: https://hudi.apache.org/docs/overview
- DuckLake: https://ducklake.select/ | https://duckdb.org/2025/06/03/ducklake.html
- Lance: https://lancedb.github.io/lance/
- F3: Boncz et al., "Faster Frozen Format" (CWI technical report)
- FastLanes layout: https://dl.acm.org/doi/10.14778/3598581.3598587 | https://github.com/cwida/FastLanes
- FastLanes file format: https://dl.acm.org/doi/10.14778/3749646.3749718
- ALP: https://dl.acm.org/doi/10.1145/3626717 | https://github.com/cwida/ALP
- DuckDB storage: https://duckdb.org/docs/internals/storage
- Parquet encodings: https://parquet.apache.org/docs/file-format/data-pages/encodings/
- Iceberg spec: https://iceberg.apache.org/spec/
- Delta protocol: https://github.com/delta-io/delta/blob/master/PROTOCOL.md
