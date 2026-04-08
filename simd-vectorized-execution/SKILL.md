---
name: simd-vectorized-execution
description: Reference for SIMD, vectorized query execution, and compiled query execution in database systems. Auto-loaded when discussing SIMD intrinsics, vectorization, query compilation, DuckDB execution model, pipeline breakers, morsel-driven parallelism, vector-at-a-time processing, or data-centric code generation.
---

# SIMD and Vectorized Execution in Database Systems

Comprehensive reference covering the execution models, SIMD techniques, and
compilation approaches used in modern analytical database engines, with emphasis
on DuckDB's architecture.

---

## 1. Execution Model Evolution

### 1.1 Volcano / Iterator Model (1990s)

**Graefe.** "Volcano — An Extensible and Parallel Query Evaluation System." 1994.

Each operator implements `Open()`, `Next()`, `Close()`. `Next()` returns one tuple.
Parent calls child's `Next()` — pull-based, one-tuple-at-a-time.

**Problems:** Virtual function call per tuple per operator. Poor instruction cache
locality (bouncing between operator implementations). No SIMD opportunity. Branch
misprediction on `Next()` null-check.

### 1.2 Block-Oriented / Vectorized Model (2005+)

**Boncz, Zukowski, Nes.** "MonetDB/X100: Hyper-Pipelining Query Execution." CIDR 2005.

`Next()` returns a **vector** (block) of ~1024 tuples instead of one tuple.
Operators process vectors via tight loops over arrays of column values.

**Key insight:** Amortize per-call overhead over 1024 tuples. Tight loops auto-vectorize
to SIMD. Data stays in L1/L2 cache during one operator's processing. Branch prediction
works well on homogeneous data.

**This is DuckDB's execution model.**

### 1.3 Data-Centric / Compiled Model (2011+)

**Neumann.** "Efficiently Compiling Efficient Query Plans for Modern Hardware." PVLDB 2011.

Generate native code (LLVM IR) that fuses multiple operators into a single loop.
No per-tuple function calls. Data stays in CPU registers across operators.

**Produce/consume paradigm:** Each operator implements `produce()` (generate tuples)
and `consume()` (process tuple from child). Code generation traverses the tree,
emitting a single tight loop per pipeline.

**Pipeline breakers** (hash build, sort, aggregate accumulate) split pipelines.
Between breakers, tuples flow through registers without materialization.

**HyPer** pioneered this; **Umbra** evolved it with adaptive compilation (see below).

### 1.4 Vectorized vs Compiled: The Definitive Comparison

**Kersten, Leis, Kemper, Neumann, Pavlo, Boncz.** "Everything You Always Wanted to
Know About Compiled and Vectorized Queries But Were Afraid to Ask." PVLDB 11(13), 2018.

Same algorithms, same data structures, two execution models. Findings:

| Workload | Winner | Why |
|---|---|---|
| Compute-intensive (expressions, predicates) | Compiled | Register allocation across operators, no interpretation |
| Memory-bound (hash joins, large scans) | Vectorized | Batch prefetching hides latency |
| Small data (fits L3 cache) | Compiled | All data register/cache-resident |
| Wide SIMD (AVX-512) | Vectorized | Columnar batches exploit SIMD naturally |
| Cold queries (first execution) | Vectorized | No compilation latency |

**Conclusion:** The performance gap is small (<2x). Choice is more about engineering
trade-offs than raw performance. DuckDB chose vectorized for simplicity and portability.

---

## 2. DuckDB's Vectorized Execution Engine

### 2.1 Core Architecture

DuckDB processes queries in a pull-based vectorized pipeline:
- **Vector size:** 2048 tuples (STANDARD_VECTOR_SIZE, configurable at compile time)
- **Column-oriented vectors:** Each column is a flat array in a `Vector` object
- **Selection vectors:** Bitmask/index array for filtering without materializing
- **Null handling:** Separate validity bitmap per vector (not inline with data)

### 2.2 The Vector Class

```cpp
class Vector {
    VectorType type;          // FLAT, CONSTANT, DICTIONARY, SEQUENCE
    LogicalType logical_type;
    data_ptr_t data;          // Pointer to raw column data
    ValidityMask validity;    // Null bitmap
    // For DICTIONARY: child vector + selection vector
    // For CONSTANT: single value, no array
};
```

**Vector types:**
- **FLAT:** Raw array of values. The common case after materialization.
- **CONSTANT:** Single value repeated for entire vector. Avoids allocation for literals.
- **DICTIONARY:** Index array + child vector. Defers decompression. Used for dictionary-encoded
  columns and string data.
- **SEQUENCE:** Start + increment. For generated series, row IDs.

### 2.3 Pipeline Execution

```
Pipeline 1: Scan → Filter → Project → Hash Build (breaker)
Pipeline 2: Scan → Hash Probe → Aggregate Build (breaker)
Pipeline 3: Aggregate Scan → Order By (breaker)
Pipeline 4: Order Scan → Result
```

Within each pipeline, operators pass vectors:
1. Source operator fills a DataChunk (collection of vectors, one per column)
2. Each intermediate operator transforms the DataChunk in-place
3. Sink operator consumes the DataChunk (builds hash table, sorts, etc.)

### 2.4 Morsel-Driven Parallelism

**Leis, Boncz, Kemper, Neumann.** "Morsel-Driven Parallelism: A NUMA-Aware Query
Evaluation Framework." SIGMOD 2014.

DuckDB uses morsel-driven parallelism for intra-query parallelism:
- Data is divided into **morsels** (~10K-100K rows)
- Worker threads pull morsels from a shared pool
- Each thread processes its morsel through the entire pipeline
- **Sinks** (hash table builds, aggregates) use thread-local pre-aggregation +
  global combine, or partitioned parallel builds

Advantages: automatic load balancing, NUMA-aware scheduling (threads prefer local morsels),
no synchronization within a morsel.

### 2.5 Adaptive Execution (Umbra)

**Neumann.** "Evolution of a Compiling Query Engine." PVLDB 14(11), 2021.
**Kohn, Leis, Neumann.** "Adaptive Execution of Compiled Queries." ICDE 2018.

Problem: LLVM compilation takes 100s of ms for complex queries.
Solution: Custom bytecode IR ("Flying Start") that can be both:
1. Interpreted immediately (zero latency)
2. Compiled to native code in background (high throughput)

At runtime: start interpreting, switch to compiled code at pipeline boundaries when
ready. For short queries, interpretation completes before compilation finishes.

### 2.6 Incremental Fusion (InkFuse)

**Wagner, Neumann.** "Incremental Fusion: Unifying Compiled and Vectorized Query
Execution." ICDE 2024.

Decomposes operators into a finite set of **sub-operators** (hash lookup, comparison,
aggregate update). Pre-generates both vectorized primitives and fused compiled versions.
Start with pre-generated vectorized primitives (zero latency), fuse in background.

Bridges the vectorized/compiled gap: gets compilation benefits without compilation latency.

---

## 3. SIMD in Database Systems

### 3.1 SIMD Instruction Set Landscape

| ISA | Width | Registers | Key Instructions | Platforms |
|---|---|---|---|---|
| SSE4.2 | 128-bit | 16 x xmm | packed int/float, string ops | x86 (2008+) |
| AVX2 | 256-bit | 16 x ymm | gather, permute, blend | x86 (2013+) |
| AVX-512 | 512-bit | 32 x zmm | mask registers, compress/expand, conflict detect | x86 (2016+, limited) |
| NEON | 128-bit | 32 x v | lane-wise ops, poly multiply | ARM (v7+) |
| SVE/SVE2 | 128-2048-bit | 32 x z | predicated, scalable, gather/scatter | ARM (v8.2+) |

**DuckDB's approach:** Minimal use of explicit SIMD intrinsics. Instead, write tight
scalar loops that the compiler auto-vectorizes. This ensures portability across x86,
ARM, RISC-V. Where explicit SIMD is used, it's behind `#ifdef` guards.

### 3.2 SIMD-Friendly Database Operations

**Selection/Filtering:**
```
// Scalar (auto-vectorizable):
for (idx_t i = 0; i < count; i++) {
    result[i] = data[i] > threshold;
}
// Compiler emits: VPCMPGTD (AVX2) or CMGT (NEON)
```

**Hash computation:**
- CRC32 instruction (SSE4.2) for hash table probing
- Multiply-shift hashing vectorizes well

**String operations:**
- PCMPESTRI/PCMPISTRM (SSE4.2) for string comparison, LIKE patterns
- FSST (Fast Static Symbol Table) compression: vectorized dictionary lookup

**Aggregation:**
- SUM: Vectorized horizontal reduction (VPHADDW/VPHADDD)
- MIN/MAX: VPMINSD/VPMAXSD across vector
- COUNT: POPCNT on validity bitmask

**Join (hash probe):**
- Vectorized probing: compute hash for batch of keys, gather from hash table
- SIMD gather (VPGATHERDD): load non-contiguous memory into SIMD register
- Conflict detection (VPCONFLICTD, AVX-512): handle hash collisions in parallel

**Sorting:**
- Sorting networks: fixed comparison sequences that vectorize perfectly
- In-register sort of 4-16 elements via min/max + shuffle
- Merge step: vectorized merge of sorted runs

### 3.3 SIMD for Compression/Decompression

**Bit-unpacking:**
- Core primitive for Parquet/DuckDB column reads
- Shift + mask operations vectorize naturally
- FastLanes layout (see storage-formats skill) achieves >100B ints/sec

**Dictionary decoding:**
- VPGATHERDD (AVX2): gather dictionary values by index in one instruction
- DuckDB defers dictionary decoding when possible (DICTIONARY vector type)

**Delta decoding:**
- Inherently sequential (prefix-sum). Two approaches:
  - Parallel prefix-sum via log(n) SIMD shuffle+add steps
  - FastLanes UTL: reorder data so independent chains align with SIMD lanes

**Frame-of-Reference (FOR):**
- Subtract base, bit-pack residuals — fully vectorizable
- ALP (floating-point) builds on FOR (see storage-formats skill)

### 3.4 Predicate Evaluation and Selection Vectors

DuckDB avoids materializing filtered results. Instead:

```cpp
// Selection vector: indices of qualifying rows
SelectionVector sel;
idx_t count = 0;
for (idx_t i = 0; i < input_count; i++) {
    if (data[i] > threshold) {
        sel[count++] = i;
    }
}
// Downstream operators read through sel, skipping filtered rows
```

This is a **branch-free** selection when compiled with SIMD: the comparison produces
a bitmask, VPCOMPRESSD (AVX-512) or a scalar equivalent compacts qualifying indices.

For complex predicates (AND/OR): combine bitmasks with bitwise AND/OR before
converting to selection vector. One materialization pass at the end.

---

## 4. Key Vectorized Operations in DuckDB

### 4.1 Hash Join

```
Pipeline 1 (Build):
  Scan build-side → vectorized hash computation → insert into hash table
  (Thread-local partitioned build, then merge)

Pipeline 2 (Probe):
  Scan probe-side → vectorized hash computation → batch probe hash table
  → vectorized key comparison → output matching tuples
```

**Hash table structure:** Linear probing with Robin Hood hashing.
Separate chains for collisions. Pointer-based for variable-size keys.

**Vectorized probing:** Process STANDARD_VECTOR_SIZE keys at once:
1. Compute hash for all keys (vectorized multiply-shift)
2. Compute bucket positions (vectorized modulo via multiply-shift)
3. Gather candidate entries from buckets
4. Compare all keys simultaneously
5. Follow collision chains for mismatches

### 4.2 Hash Aggregate

```
Scan → vectorized hash computation → probe aggregate hash table
  → update aggregation states in-place
```

For pre-aggregation (thread-local): small hash table (fits L2 cache).
For global aggregation: partitioned, lock-free parallel build.

**Aggregate state updates:** Each aggregate function provides vectorized
`Update(Vector &input, AggregateState *states[], idx_t count)` that
processes an entire vector of values into an array of state pointers.

### 4.3 Sort

DuckDB implements a hybrid sorting algorithm:
1. **In-register sort** for tiny arrays (4-16 elements via sorting networks)
2. **PDQ sort** (Pattern-defeating quicksort) for medium arrays
3. **External merge sort** for data exceeding memory

**Comparison keys:** Normalized/serialized into a binary-comparable format
so that `memcmp` suffices (no type-specific comparison). This enables
SIMD memcmp for multi-column sort keys.

### 4.4 String Processing

DuckDB's string representation:
- Short strings (<=12 bytes): inlined in the vector
- Long strings: 4-byte prefix + pointer to overflow buffer
- Prefix enables fast inequality checks without pointer chase

**FSST (Fast Static Symbol Table):**
**Boncz, Neumann, Ruhle.** "FSST: Fast Random Access String Compression." PVLDB 2020.

Symbol table of up to 256 entries mapping 1-8 byte substrings to single bytes.
Compression and decompression are branch-free table lookups — fully vectorizable.
DuckDB uses FSST for string column compression in storage.

---

## 5. Advanced SIMD Techniques

### 5.1 Vectorized Bloom Filters

**Polychroniou, Ross.** "Vectorized Bloom Filters for Advanced SIMD Processors."
DaMoN 2014.

Blocked Bloom filter: partition into cache-line-sized blocks. Each key maps to one
block (single cache line access). Within a block, use SIMD to set/test all k hash
positions simultaneously.

Probing: vectorized hash → gather block pointers → SIMD AND test all positions.
Throughput: >1 billion probes/second on modern CPUs.

Used in DuckDB for semi-join optimization and zone map pruning.

### 5.2 SIMD-Accelerated Joins Beyond Hash

**Balkesen, Alonso, Teubner, Ozsu.** "Multi-Core, Main-Memory Joins: Sort vs. Hash
Revisited." PVLDB 2013.

Sort-merge join with SIMD: vectorized sorting networks + vectorized merge.
Competitive with hash join for sorted/nearly-sorted inputs. Better cache behavior
for skewed data.

**Partitioned hash join:** Radix partitioning with SIMD scatter (VPSCATTERDD) to
distribute tuples to partitions. Each partition fits in L2 cache.

### 5.3 SIMD for Selection Scans

**Li, Patel.** "BitWeaving: Fast Scans for Main Memory Data Processing." SIGMOD 2013.

Two layouts for SIMD-friendly predicate evaluation:
- **BitWeaving/V (Vertical):** Bit-sliced storage. Each bit position stored separately.
  Predicate evaluation via bitwise AND/OR across bit slices.
- **BitWeaving/H (Horizontal):** Pack multiple values into one SIMD word with delimiter
  bits. Evaluate predicates on packed representation.

Both achieve near-memory-bandwidth scan performance.

### 5.4 Vectorized Regular Expressions

**Mytkowicz, Musuvathi, Schulte.** "Data-parallel Finite-State Machines." ASPLOS 2014.

NFA/DFA simulation in SIMD: process multiple characters simultaneously using
VPSHUFB (byte shuffle) as a table lookup. Each SIMD lane processes one character
position. State transitions via shuffle instructions.

DuckDB uses RE2 for regex but applies vectorized fast-path checks (literal prefix,
LIKE pattern special cases) before falling back to full regex.

---

## 6. Practical SIMD Guidelines for DuckDB Extension Development

### 6.1 Prefer Auto-Vectorization

```cpp
// GOOD: Compiler auto-vectorizes this loop
for (idx_t i = 0; i < count; i++) {
    result_data[i] = input_data[i] * factor + offset;
}

// BAD: Explicit SIMD (non-portable, harder to maintain)
__m256i vec = _mm256_loadu_si256((__m256i*)input_data);
vec = _mm256_mullo_epi32(vec, factor_vec);
vec = _mm256_add_epi32(vec, offset_vec);
_mm256_storeu_si256((__m256i*)result_data, vec);
```

**Rules for auto-vectorization:**
- Use simple `for` loops with predictable iteration counts
- Avoid data-dependent branches inside loops (use conditional moves/masks)
- Avoid pointer aliasing (use `__restrict__` or separate arrays)
- Access memory sequentially (stride-1 patterns)
- Keep loop body simple (compiler gives up on complex bodies)

### 6.2 Profile Before Optimizing

DuckDB's vectorized model already handles most SIMD opportunities. Manual SIMD is
only worth it for:
- Hot inner loops that process >10M values (compression, hashing)
- Operations where the compiler's auto-vectorization provably fails
- Platform-specific fast paths (CRC32 for hashing, POPCNT for bitmasks)

### 6.3 Selection Vector Awareness

Always handle selection vectors in operator implementations:
```cpp
void MyOperator(Vector &input, Vector &result, idx_t count) {
    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(count, input_data);
    // input_data.sel contains selection vector
    // input_data.validity contains null bitmap
    for (idx_t i = 0; i < count; i++) {
        auto idx = input_data.sel->get_index(i);
        if (input_data.validity.RowIsValid(idx)) {
            // process input_data.data[idx]
        }
    }
}
```

### 6.4 Avoid Pipeline Breakers When Possible

Every materialization point (writing vectors to memory) is a potential performance
cliff. Prefer:
- In-place vector transformation over creating new vectors
- Selection vectors over copying qualifying rows
- CONSTANT/DICTIONARY vector types over flattening

---

## 7. Key Papers

- MonetDB/X100: Boncz et al., CIDR 2005
- HyPer compiled execution: Neumann, PVLDB 2011
- Morsel-driven parallelism: Leis et al., SIGMOD 2014
- Compiled vs Vectorized: Kersten et al., PVLDB 2018
- Adaptive execution (Umbra): Neumann, PVLDB 2021
- Adaptive compiled queries: Kohn et al., ICDE 2018
- InkFuse: Wagner, Neumann, ICDE 2024
- FastLanes: Afroozeh, Boncz, PVLDB 2023
- ALP: Afroozeh et al., SIGMOD 2024
- FSST: Boncz et al., PVLDB 2020
- BitWeaving: Li, Patel, SIGMOD 2013
- Vectorized Bloom filters: Polychroniou, Ross, DaMoN 2014
- Sort vs Hash joins: Balkesen et al., PVLDB 2013
- Data-parallel FSMs: Mytkowicz et al., ASPLOS 2014
- Relaxed Operator Fusion: Menon et al., SIGMOD 2017
