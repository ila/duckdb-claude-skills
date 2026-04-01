---
name: duckdb-internals
description: Reference for DuckDB internals, architecture, and extension development APIs. Auto-loaded when working with DuckDB source code, writing optimizer/parser extensions, or discussing DuckDB execution, storage, compression, transactions, or performance.
---

# DuckDB Internals Reference

## 1. Architecture Overview

DuckDB is an in-process OLAP database. No server — it runs embedded in the host process.
Core design: columnar storage, vectorized execution, push-based pipeline model.

**Query processing pipeline:**

```
SQL text → Parser → Binder → Logical Planner → Optimizer → Physical Plan → Execution
```

1. **Parser**: SQL string → AST (SQLStatement, QueryNode, TableRef, ParsedExpression). No catalog awareness.
2. **Binder**: Resolves tables, columns, types via catalog. Produces BoundStatement with typed expressions.
3. **Logical Planner**: BoundStatement → LogicalOperator tree.
4. **Optimizer**: 32+ passes on the logical plan (see Section 4).
5. **Column Binding Resolver**: Converts BoundColumnRefExpression → BoundReferenceExpression (DataChunk indices).
6. **Physical Plan Generator**: LogicalOperator → PhysicalOperator tree.
7. **Execution**: Push-based vectorized execution on DataChunks.

## 2. Vectorized Execution

Inspired by MonetDB/X100 (Boncz, Zukowski, Nes 2005).

**Vector**: Single column of data. Fixed batch size: `STANDARD_VECTOR_SIZE = 2048` tuples.

**Vector types:**
- **Flat**: Contiguous array. Standard uncompressed format.
- **Constant**: Single value for all rows. Avoids duplication (e.g., literal `'duckdb'` for 1000 rows).
- **Dictionary**: Child vector + selection vector with indices. Preserves compression during execution.
- **Sequence**: Offset + increment. Efficient for row IDs and sequential patterns.

**UnifiedVectorFormat**: Generic view over any vector type. Avoids combinatorial explosion of type-specific code.

**DataChunk**: Collection of Vectors (one per output column). Pushed through operators.

**Strings**: `string_t` inlines strings ≤ 12 bytes. Longer strings use heap pointers. 4-byte prefix stored for fast comparison early-out.

**Nulls**: Validity mask (bitmask). One bit per row. Checked via `ValidityMask`.

**Lists**: `list_entry_t` with offset + length into a child vector. Supports nested lists.

**Structs/Maps**: Child vectors per field. Maps are `LIST[STRUCT(key, value)]` internally.

## 3. Push-Based Execution Model

Adopted in DuckDB v0.3.1 (Nov 2021), replacing the original pull-based Volcano model.

**How it works:**
- Operators push DataChunks downstream (not pulled by parent).
- Each operator decides its own parallelism.
- Pipelines: chains of operators between pipeline breakers.

**Pipeline breakers** (operators that must materialize): hash join build side, hash aggregate, sort, window functions.

**Morsel-driven parallelism** (Leis et al. 2014):
- Data is split into morsels (chunks of rows).
- Worker threads process morsels independently.
- No central coordinator — each operator manages its own parallelism.
- Pipeline-aware: parallelism boundaries align with pipeline structure.

**Source/Sink model:**
- **Source**: Produces DataChunks (e.g., table scan).
- **Sink**: Consumes DataChunks, materializes state (e.g., hash table build).
- **Operator**: Transforms DataChunks in-flight (e.g., filter, projection).

## 4. Optimizer Passes (Complete List, in Order)

From `src/optimizer/optimizer.cpp`, the exact execution order:

| # | Pass | Description |
|---|------|-------------|
| 1 | `EXPRESSION_REWRITER` | Constant folding, algebraic simplification, case/conjunction/comparison simplification, regex/like optimization, move constants |
| 2 | `CTE_INLINING` | Inline CTEs instead of materializing (first pass) |
| 3 | `SUM_REWRITER` | Rewrites `SUM(x + C)` → `SUM(x) + C * COUNT(x)` |
| 4 | `FILTER_PULLUP` | Pulls filters up in the plan tree |
| 5 | `FILTER_PUSHDOWN` | Pushes filters down toward scans. Duplicates filters across equivalency sets |
| 6 | `CTE_FILTER_PUSHER` | Derives and pushes filters into materialized CTEs |
| 7 | `REGEX_RANGE` | Converts regex patterns to range filters |
| 8 | `IN_CLAUSE` | Rewrites IN clauses to MARK/INNER joins or equality checks |
| 9 | `DELIMINATOR` | Removes redundant DelimGets/DelimJoins from decorrelated subqueries |
| 10 | `CTE_INLINING` | Second pass after initial optimizations |
| 11 | `EMPTY_RESULT_PULLUP` | Pulls up empty results to short-circuit computation |
| 12 | `WINDOW_SELF_JOIN` | Replaces some window computations with self-joins |
| 13 | `JOIN_ORDER` | Join ordering via DPccp algorithm (Moerkotte & Neumann). Rewrites cross products + filters into joins |
| 14 | `JOIN_ELIMINATION` | Eliminates unnecessary joins |
| 15 | `UNNEST_REWRITER` | Rewrites UNNESTs in DelimJoins to projections |
| 16 | `UNUSED_COLUMNS` | Removes columns not used downstream |
| 17 | `DUPLICATE_GROUPS` | Removes duplicate groups from aggregates |
| 18 | `COMMON_SUBEXPRESSIONS` | Extracts common subexpressions to prevent redundant evaluation |
| 19 | `COLUMN_LIFETIME` | Creates projection maps to project out unused columns early (first pass) |
| 20 | `BUILD_SIDE_PROBE_SIDE` | Determines join build/probe sides based on column lifetime |
| 21 | `COMMON_SUBPLAN` | Converts common subplans into materialized CTEs |
| 22 | `LIMIT_PUSHDOWN` | Pushes LIMIT below PROJECTION |
| 23 | `ROW_GROUP_PRUNER` | Prunes row groups based on zone map statistics |
| 24 | `SAMPLING_PUSHDOWN` | Pushes sampling into scans |
| 25 | `TOP_N` | Transforms ORDER BY + LIMIT into TopN operator: O(M + N log N) instead of O(M log M) |
| 26 | `LATE_MATERIALIZATION` | Defers materializing columns until needed |
| 27 | `STATISTICS_PROPAGATION` | Propagates column statistics through the plan. Builds statistics map. Creates filters from join constraints (e.g., if `t1.a` max=50, pushes `t2.a <= 50`) |
| 28 | `TOP_N_WINDOW_ELIMINATION` | Rewrites `row_number() OVER ... + filter` to aggregate |
| 29 | `COMMON_AGGREGATE` | Removes duplicate aggregate computations |
| 30 | `COLUMN_LIFETIME` | Second pass after all optimizations |
| 31 | `REORDER_FILTER` | Orders filter execution: cheap predicates (equality) before expensive (function calls) |
| 32 | `JOIN_FILTER_PUSHDOWN` | Pushes min/max filters from hash table build side to probe side |

**Extension passes** run before (`pre_optimize_function`) and after (`optimize_function`) the built-in passes. OpenIVM uses `pre_optimize_function` to rewrite plans before DuckDB's optimizers run.

### Join Ordering (DPccp)

Uses the DPccp algorithm from "Dynamic Programming Strikes Back" (Moerkotte & Neumann, 2006). Enumerates connected subgraph complement pairs for optimal join ordering. Recognizes equality conditions as join opportunities and eliminates cross products.

### Subquery Decorrelation

Based on "Unnesting Arbitrary Queries" (Neumann & Kemper, 2015) and its updated version "Improving Unnesting of Complex Queries." All correlated subqueries are decorrelated — DuckDB generates special internal join types (MARK, SINGLE, DELIM) to handle EXISTS, IN, scalar subqueries. Top-down decorrelation strategy since PR #17294.

### Compressed Materialization

Compresses data on-the-fly at materializing operators (sort, join, aggregate). Uses column statistics to convert to smaller types at runtime (e.g., encode string columns as uint64 via length information). Removes redundant compressions when materializing operators are chained.

## 5. Storage

### Format

Single-file database. Data organized hierarchically:
- **Row Groups**: Horizontal partitions of ~120K rows.
- **Column Data**: One per column within a row group.
- **Column Segments**: Horizontal partitions within a column. Typically one 256 KB data block.

Fixed-size blocks enable in-place ACID modifications and avoid fragmentation.

### Compression (16 types)

| Algorithm | Used for | Description |
|-----------|----------|-------------|
| `CONSTANT` | Any | All values identical in segment |
| `RLE` | Integer, string | Run-length encoding for repeated consecutive values |
| `DICTIONARY` | String | Map values to integer codes |
| `BITPACKING` | Integer | Pack values using minimum bits needed (per 1024 values) |
| `PFOR_DELTA` | Integer | Patched Frame of Reference |
| `FSST` | String | Fast Static Symbol Table compression |
| `DICT_FSST` | String | Dictionary + FSST hybrid |
| `CHIMP` | Float | Time-series floating-point compression |
| `PATAS` | Float | Floating-point compression |
| `ALP` | Float | Adaptive Lossless floating-Point (Afroozeh, Kuffo, Boncz) |
| `ALPRD` | Float | ALP with Row-by-row Decoding |
| `ZSTD` | Any | Zstandard general-purpose compression |
| `ROARING` | Boolean/null | Roaring bitmaps |
| `EMPTY` | Null-only | Empty/null-only columns |

Auto-selected per column segment based on data characteristics analysis.

### Indexes

**Min-Max Index (Zone Maps)**: Automatic on all columns. Tracks min/max per row group. Enables row group pruning during scans. No user action needed.

**ART (Adaptive Radix Tree)**: Created manually via `CREATE INDEX` or automatically for PRIMARY KEY / UNIQUE constraints. Trie-based with adaptive node sizes (4, 16, 48, 256 children). Cache-efficient. O(k) point queries. Best for highly selective queries (< 0.1% selectivity). Must fit in memory during creation.

> **Note:** ART indexes slow down writes (maintained on every INSERT/DELETE/UPDATE). Only use them when point query speedup outweighs write overhead.

## 6. Operator Implementations

### Hash Join

- **Build phase**: Each thread builds a thread-local `JoinHashTable` via radix partitioning.
- **Probe phase**: Probe side scans partitions, matching against build-side hash table.
- **Radix partitioning**: Starts at 4 bits (16 partitions), scales to 12 bits (4096 partitions) if partitions don't fit in memory.
- Physical join types: `HASH_JOIN` (general), `MERGE_JOIN` (sorted inputs), `CROSS_JOIN` (cartesian), `ASOF_JOIN` (temporal).

### Hash Aggregate (GROUP BY)

- `GroupedAggregateHashTable`: Linear-probing hash table storing group keys + aggregate states.
- **Parallel strategy**: Each thread builds its own hash table. Partitioning kicks in when a thread exceeds 10,000 rows. Radix-partitioned data can spill to disk for external aggregation.
- **DISTINCT aggregates**: Separate radix hash table per distinct input signature. Deduplicates within each partition. Multiple aggregates sharing identical distinct inputs reuse the same table.

### Window Functions

- Partitions are hash-distributed into ~1024 chunks using O(N) hashing.
- Each chunk is sorted on ORDER BY columns independently.
- **Streaming optimization**: When no PARTITION BY or ORDER BY, window functions (ROW_NUMBER, running SUM) stream without materializing the full relation.
- Multi-threaded: different partitions run on separate threads.

### DISTINCT

Implemented as GROUP BY on all output columns. At the physical level, uses the same `RadixPartitionedHashTable` as GROUP BY.

## 7. Transaction Management

**Isolation level**: SERIALIZABLE via MVCC (Multi-Version Concurrency Control).

**Core structures:**
- `DuckTransaction`: holds `start_time`, `transaction_id`, `commit_id`, undo buffer, local storage.
- `UndoBuffer`: Stores old row versions for rollback and MVCC reads. Written to WAL on commit.
- `LocalStorage`: Stages uncommitted appends/deletes/updates in thread-local row groups.

**Row versioning (ChunkInfo):**
- Each row has `insert_id` and `delete_id` (transaction IDs).
- Visibility check: row visible if `insert_id <= reader.start_time AND (delete_id > reader.start_time OR delete_id == INVALID)`.
- `GetSelVector()` returns only visible rows for a transaction during scans.

**Commit/Rollback:**
- Commit: undo buffer → WAL → mark entries as committed → advance commit_id.
- Rollback: apply undo entries in reverse to restore old state.
- Cleanup: once `lowest_active_transaction > version`, undo entries are no longer needed.

**Checkpointing**: Captures consistent state across all tables. Prevents concurrent checkpoints via lock. WAL is truncated after successful checkpoint.

## 8. Extension Development API

### Registration Pattern

```cpp
class MyExtension : public Extension {
    void Load(ExtensionLoader &loader) override;
    string Name() override { return "my_ext"; }
};

// Entry point macro (at end of .cpp file)
DUCKDB_CPP_EXTENSION_ENTRY(my_ext, loader) { LoadInternal(loader); }
```

Inside `Load()`, register:
- `loader.RegisterFunction(pragma)` — pragma functions
- `loader.RegisterFunction(scalar)` — scalar functions
- `loader.RegisterType(type)` — custom types
- `ParserExtension::Register(config, parser)` — parser hooks
- `OptimizerExtension::Register(config, optimizer)` — optimizer hooks
- `db_config.AddExtensionOption(name, desc, type, default)` — settings

### Parser Extension

```cpp
// Parse callback: SQL string → custom parse data
ParserExtensionParseResult parse_fn(ParserExtensionInfo *info, const string &query);

// Plan callback: parse data → table function to execute
ParserExtensionPlanResult plan_fn(ParserExtensionInfo *info, ClientContext &ctx,
                                   unique_ptr<ParserExtensionParseData> data);
```

Return `PARSE_SUCCESSFUL` to claim the query, `DISPLAY_ORIGINAL_ERROR` to pass through.

### Optimizer Extension

```cpp
// Pre-optimize: runs BEFORE DuckDB's built-in passes
void pre_optimize_fn(OptimizerExtensionInput &input, unique_ptr<LogicalOperator> &plan);

// Post-optimize: runs AFTER built-in passes
void optimize_fn(OptimizerExtensionInput &input, unique_ptr<LogicalOperator> &plan);
```

OpenIVM uses `pre_optimize_function` — this runs before filter pushdown, join ordering, etc., giving full control over the raw plan.

### Pragma Functions

```cpp
// Query pragma: returns a SQL string to execute
auto pragma = PragmaFunction::PragmaCall("ivm", my_query_fn, {LogicalType::VARCHAR});
loader.RegisterFunction(pragma);

// pragma_query_t signature:
string my_query_fn(ClientContext &context, const FunctionParameters &parameters);
```

### Key Types

| Type | Description |
|------|-------------|
| `ColumnBinding{table_idx, column_idx}` | Maps output column to source table/operator |
| `LogicalType` | SQL type (VARCHAR, BIGINT, LIST, STRUCT, etc.) |
| `Value` | Runtime typed value. `Value::BOOLEAN(true)`, `val.GetValue<int64_t>()` |
| `DataChunk` | Batch of column Vectors. 2048 rows max. |
| `Connection` | Query execution handle. `con.Query(sql)`, `con.BeginTransaction()` |

### Logical Operator Key Fields

| Operator | Key fields |
|----------|------------|
| `LogicalGet` | `table_index`, `function`, `bind_data`, `projection_ids` |
| `LogicalFilter` | `expressions[0]` = filter predicate |
| `LogicalProjection` | `table_index`, `expressions` = output columns |
| `LogicalAggregate` | `group_index`, `aggregate_index`, `groups`, `expressions` = aggregate calls |
| `LogicalComparisonJoin` | `conditions`, `join_type` (INNER/LEFT/RIGHT/OUTER/SEMI/ANTI/MARK) |
| `LogicalSetOperation` | `table_index`, `column_count`, `setop_all` (UNION vs UNION ALL) |

### Enums (Subset)

**JoinType**: INVALID, LEFT, RIGHT, INNER, OUTER, SEMI, ANTI, MARK, SINGLE, RIGHT_SEMI, RIGHT_ANTI

**LogicalOperatorType**: LOGICAL_PROJECTION(1), LOGICAL_FILTER(2), LOGICAL_AGGREGATE_AND_GROUP_BY(3), LOGICAL_WINDOW(4), LOGICAL_LIMIT(6), LOGICAL_ORDER_BY(7), LOGICAL_GET(25), LOGICAL_COMPARISON_JOIN(52), LOGICAL_CROSS_PRODUCT(54), LOGICAL_UNION(75), LOGICAL_INSERT(100), LOGICAL_DELETE(101), LOGICAL_UPDATE(102)

## 9. Key Papers & References

| Paper | Relevance to DuckDB |
|-------|---------------------|
| Raasveldt & Mühleisen, "DuckDB: an Embeddable Analytical Database" (SIGMOD 2019) | Original DuckDB paper. Vectorized Volcano model, embedded design. |
| Boncz, Zukowski, Nes, "MonetDB/X100: Hyper-Pipelining Query Execution" (CIDR 2005) | Vectorized execution model that DuckDB's engine is based on. |
| Leis et al., "Morsel-Driven Parallelism" (SIGMOD 2014) | Parallelism model adopted by DuckDB. Each operator manages its own parallelism. |
| Neumann & Kemper, "Unnesting Arbitrary Queries" (BTW 2015) | Subquery decorrelation algorithm used by DuckDB's optimizer. |
| Moerkotte & Neumann, "Dynamic Programming Strikes Back" (SIGMOD 2006) | DPccp join ordering algorithm. |
| Afroozeh, Kuffo, Boncz, "ALP: Adaptive Lossless floating-Point Compression" (SIGMOD 2023) | ALP compression in DuckDB for floating-point columns. |
| Leis et al., "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases" (ICDE 2013) | ART index data structure. Adaptive node sizes (4/16/48/256). |
| Raasveldt et al., "Lightweight Compression in DuckDB" (DuckDB Blog 2022) | Per-segment compression selection framework. |
| Kuiper, "Compressed Materialization" (DuckDB PR #7644) | On-the-fly compression at materializing operators. |
| Raasveldt, "Push-Based Execution in DuckDB" (DSDSD Workshop) | Transition from pull-based to push-based model. |
| Raasveldt & Mühleisen, "DuckDB-Wasm" (VLDB 2022) | WebAssembly compilation of DuckDB. |

## 10. Online Resources

- [DuckDB Internals Overview](https://duckdb.org/docs/current/internals/overview.html)
- [Execution Format (Vector/DataChunk)](https://duckdb.org/docs/current/internals/vector.html)
- [Optimizers: The Low-Key MVP (blog)](https://duckdb.org/2024/11/14/optimizers)
- [Parallel Grouped Aggregation (blog)](https://duckdb.org/2022/03/07/aggregate-hashtable)
- [External Aggregation (blog)](https://duckdb.org/2024/03/29/external-aggregation)
- [Persistent ART Storage (blog)](https://duckdb.org/2022/07/27/art-storage)
- [Lightweight Compression (blog)](https://duckdb.org/2022/10/28/lightweight-compression)
- [Correlated Subqueries (blog)](https://duckdb.org/2023/05/26/correlated-subqueries-in-sql)
- [Window Functions (blog)](https://duckdb.org/2021/10/13/windowing)
- [Flying Through Windows (blog)](https://duckdb.org/2025/02/14/window-flying)
- [CMU 15-721 DuckDB Lecture (slides)](https://15721.courses.cs.cmu.edu/spring2024/notes/20-duckdb.pdf)
- [Extension Template (GitHub)](https://github.com/duckdb/extension-template)
- [Indexes Documentation](https://duckdb.org/docs/current/sql/indexes.html)
- [Building Extensions](https://duckdb.org/docs/current/dev/building/building_extensions.html)
- [Community Extensions](https://duckdb.org/community_extensions/development)
- [Pre-optimization Hooks PR](https://github.com/duckdb/duckdb/pull/16115)
- [Optimizer Source Code](https://github.com/duckdb/duckdb/blob/main/src/optimizer/optimizer.cpp)

## 11. Key Source Code Locations

```
src/parser/               -- Parser and Transformer
src/planner/              -- Binder and Logical Planner
src/optimizer/            -- All optimizer passes
  optimizer.cpp           -- Main orchestration (pass ordering)
  expression_rewriter.cpp -- Expression simplification rules
  filter_pushdown/        -- Filter pushdown logic
  join_order/             -- DPccp join ordering
src/execution/            -- Physical operators and execution engine
src/parallel/             -- Pipeline, MetaPipeline, task scheduling
src/catalog/              -- Catalog, CatalogEntry, schema management
src/storage/              -- DataTable, RowGroup, ColumnSegment, buffer manager
  index/art/              -- ART index implementation
  compression/            -- All compression algorithms
src/include/duckdb/
  parser/parser_extension.hpp      -- ParserExtension API
  optimizer/optimizer_extension.hpp -- OptimizerExtension API
  planner/operator_extension.hpp   -- OperatorExtension API
  common/types/vector.hpp          -- Vector class
  common/types/data_chunk.hpp      -- DataChunk class
  catalog/catalog.hpp              -- Catalog class
  transaction/transaction.hpp      -- Transaction base class
  transaction/duck_transaction.hpp -- MVCC transaction implementation
```

## 12. Practical Patterns for Extension Development

### Walking the Logical Plan

```cpp
void WalkPlan(unique_ptr<LogicalOperator> &op) {
    switch (op->type) {
        case LogicalOperatorType::LOGICAL_GET:
            // handle table scan
            break;
        case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
            // handle join — cast to access join_type, conditions
            auto &join = op->Cast<LogicalComparisonJoin>();
            break;
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            // handle aggregate — cast to access groups, expressions
            auto &agg = op->Cast<LogicalAggregate>();
            break;
    }
    for (auto &child : op->children) {
        WalkPlan(child);
    }
}
```

### Creating a Plan from SQL

```cpp
Connection con(instance);
Parser parser;
parser.ParseQuery(sql);
Planner planner(*con.context);
planner.CreatePlan(std::move(parser.statements[0]));
auto plan = std::move(planner.plan);  // unique_ptr<LogicalOperator>
```

### Reading Extension Settings

```cpp
Value val;
if (context.TryGetCurrentSetting("my_setting", val) && !val.IsNull()) {
    string s = StringValue::Get(val);
    bool b = val.GetValue<bool>();
}
```

### Working with the Catalog

```cpp
Connection con(instance);
con.BeginTransaction();
auto &catalog = Catalog::GetSystemCatalog(*con.context);

// Register a table function
TableFunction func("my_func", {LogicalType::VARCHAR}, MyFn, MyBind, MyInit);
CreateTableFunctionInfo info(func);
catalog.CreateTableFunction(*con.context, &info);
con.Commit();
```

### Catalog Hierarchy

```
DatabaseManager → AttachedDatabase → Catalog → DuckSchemaEntry → entries
                                                ├── DuckTableEntry
                                                ├── DuckViewEntry
                                                ├── DuckIndexEntry
                                                └── ...
```

Lookup: `Catalog::GetEntry<TableCatalogEntry>(context, catalog, schema, table_name)`

## 13. Row IDs

Row IDs (`row_t`, signed 64-bit integer) are **logical** identifiers for tuple positions. They are NOT physical addresses.

- Managed by `RowIdColumnData` — a specialized column type per row group.
- Delete operations mark rows as deleted in `RowVersionManager` (per-row-group MVCC info) without changing the row ID.
- Used by: `PhysicalDelete`, `PhysicalUpdate`, counting-based upsert (OpenIVM's `DELETE WHERE rowid IN (...)`).
- `rowid` is exposed as a virtual column in SQL: `SELECT rowid, * FROM my_table`.

## 14. Buffer Manager & Out-of-Core

The buffer manager caches pages from persistent storage in memory, evicting when space is needed.

**Key properties:**
- **Eviction policy**: LRU-based with separate queues for different buffer types (BLOCK, MANAGED, TINY, EXTERNAL).
- **Pin/Unpin**: Operators pin blocks for use, unpin when done. Unpinned blocks are eviction candidates.
- **Spill to disk**: When memory is exhausted, unpinned blocks are written to temporary files. Supports configurable swap limit (`SET max_temp_directory_size`).
- **Memory limit**: Configurable via `SET memory_limit = '4GB'`. Only applies to the buffer manager — some allocations (e.g., thread stacks) are outside its control.
- **Out-of-core operators**: Hash join, hash aggregate, sort, and window all support disk spilling. See "External Aggregation in DuckDB" (DuckDB blog, 2024).

## 15. Write-Ahead Log (WAL) & Checkpointing

**WAL**: Sequential log of all committed changes. On each transaction commit, changes are appended to the WAL file.

**What's logged**: CREATE/DROP TABLE/VIEW/INDEX/SCHEMA/SEQUENCE/TYPE, INSERT/DELETE/UPDATE tuples, ALTER, ROW_GROUP_DATA, CHECKPOINT markers.

**Checkpointing**: Periodically, WAL changes are physically applied to the database file ("checkpointing"), after which WAL entries are truncated. Triggered by `CHECKPOINT` statement or automatically.

**Recovery**: On startup, if a WAL file exists, DuckDB replays it to restore committed state.

**Optimistic writes**: During bulk loading, DuckDB can write data blocks directly to the database file before commit (optimistic writing). The WAL tracks which blocks are in-use to prevent overwriting.

## 16. Statistics

Per-column statistics maintained for query optimization (zone map pruning, filter pushdown).

| Stat type | Fields | Used for |
|-----------|--------|----------|
| `NUMERIC_STATS` | `has_min`, `has_max`, `min`, `max` | Zone map pruning, range filters |
| `STRING_STATS` | `min[8]`, `max[8]`, `has_unicode`, `max_string_length` | String comparison shortcuts |
| `BASE_STATS` | `has_null`, `has_no_null`, `distinct_count` | Null elimination, cardinality estimation |
| `LIST_STATS`, `STRUCT_STATS`, `ARRAY_STATS` | Child statistics recursively | Nested type optimization |

**Zone map pruning**: If a row group's max value for column `x` is 50 and the query has `WHERE x > 100`, the entire row group is skipped (zero I/O). The `ROW_GROUP_PRUNER` optimizer pass uses these statistics.

## 17. Sorting

DuckDB uses a **merge-sort** implementation with external sort support:

- All data is serialized into a **unified internal row layout** (same layout used by joins and aggregates).
- Rows are stored in buffer-managed blocks that can spill to disk.
- String pointers are converted to offsets for disk serialization and restored on reload.
- Radix sort used for the first pass on fixed-width keys; merge sort for combining.
- See "Fastest Table Sort in the West" (DuckDB blog, 2021).

## 18. Hash Table Internals

`JoinHashTable` — used by hash join and hash aggregate:

- **Structure**: Linear probing with chaining. Each entry: `[SERIALIZED_ROW][NEXT_POINTER]`.
- **Salt optimization**: When capacity > 8192 entries, stores hash salt per entry for fast comparison without deserializing the full key.
- **Collision resolution**: Hash → first entry → chain via next pointer.
- **Radix partitioning**: For large tables, starts at 16 partitions (4 bits), scales to 4096 (12 bits).

## 19. Appender API (Bulk Loading)

```cpp
Appender appender(con, "my_table");
appender.BeginRow();
appender.Append<int32_t>(42);
appender.Append<string_t>("hello");
appender.EndRow();
appender.Close();  // flushes remaining data
```

Auto-flushes every ~102,400 rows (`STANDARD_VECTOR_SIZE * 100`). Also supports `AppendDataChunk()` for bulk loading from DataChunks. More efficient than individual INSERT statements.

## 20. Result Set API

Two result types:
- **MaterializedQueryResult**: All data in memory. Random access via `GetValue(col, row)`. Row count via `RowCount()`. Used by `con.Query()`.
- **StreamQueryResult**: Streaming. Fetch one DataChunk at a time via `Fetch()`. Lower memory. Used by `con.SendQuery()`.

Both support `Fetch()` (returns DataChunk), `HasError()`, column names/types, and range-based for loops via `QueryResultIterator`.

## 21. Additional Blog Posts & Papers

- [Memory Management in DuckDB (blog, 2024)](https://duckdb.org/2024/07/09/memory-management)
- [Fastest Table Sort in the West (blog, 2021)](https://duckdb.org/2021/08/27/external-sorting)
- [External Aggregation in DuckDB (blog, 2024)](https://duckdb.org/2024/03/29/external-aggregation)
- [Analytics-Optimized Concurrent Transactions (blog, 2024)](https://duckdb.org/2024/10/30/analytics-optimized-concurrent-transactions)
- [Sorting on Insert for Fast Selective Queries (blog, 2025)](https://duckdb.org/2025/05/14/sorting-for-fast-selective-queries)
- [Robust External Hash Aggregation (ICDE 2024, Kuiper, Boncz, Mühleisen)](https://duckdb.org/pdf/ICDE2024-kuiper-boncz-muehleisen-out-of-core.pdf)
