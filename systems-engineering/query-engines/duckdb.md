# DuckDB

## 30-Second Intuition

DuckDB is an embedded, in-process OLAP database — SQLite's deployment model applied to analytical workloads. It runs inside your Python/Go/Java process (no server, no cluster, no network hop to the query engine) and hits single-node throughput that rivals a small Spark cluster for anything that fits on one machine. The one fact that matters operationally: DuckDB's speed comes from **zero shuffle, not just vectorization** — every thread operates on shared in-process memory, so the network-bound redistribution step that dominates Spark's `groupBy`/`join` cost (see `compute/spark.md`) simply doesn't exist here. That's the actual reason a "cluster-scale" query finishes in seconds on a laptop: it was never a distributed computation to begin with.

---

## Resource-Layer Map

| Layer | Role DuckDB plays | What it's optimizing for |
|---|---|---|
| CPU | SIMD-friendly vectorized execution: operators process 2048-row columnar batches, not one row at a time | Let the CPU run tight, homogeneous-type loops that auto-vectorize (AVX2/AVX-512), instead of interpreting a row-at-a-time plan tree with per-row dispatch overhead |
| Memory | Fixed-size buffer pool (default 80% of available RAM) holds hot data/hash tables/sort buffers; LRU eviction under pressure | Keep working sets in cache-hot memory as long as possible; spill only when genuinely necessary, and make the spill transparent to the query rather than a hard failure |
| Disk | Spill files for oversized hash tables/sort buffers/window state; otherwise disk I/O is only the initial Parquet/CSV read | Disk is the overflow valve, not a required step — a query whose intermediate state fits in the buffer pool never touches disk beyond the original scan |
| Network | HTTPFS range requests against S3/GCS/Azure — read only the byte ranges actually needed, in parallel across files | Minimize bytes transferred over the wire via footer-first reads + row-group/file pruning, since for a remote-storage query network is the dominant cost, not CPU |
| GPU | Not part of the core engine | DuckDB's execution model targets CPU SIMD, not GPU kernels — no native GPU execution path exists in the core engine |

The sharpest contrast with Spark (`compute/spark.md`): Spark's resource-layer table has network as "the dominant cost for wide transformations" because data must move between physically separate executors. DuckDB's network row is *only* about pulling remote storage bytes in — once data is in this one process's memory, there is no further network cost, because there's only one process. This is the direct explanation for "DuckDB has no shuffle": shuffle exists to redistribute data across independent workers, and DuckDB doesn't have independent workers, it has threads sharing one address space.

---

## The Signature Mechanism: Vectorized Execution + Morsel-Driven Parallelism

Two ideas, borrowed from and refined past academic research (X100/MonetDB for vectorization, TUM's HyPer project for morsel-driven parallelism), that compound with each other:

**Vectorized execution**: instead of processing one row through the entire operator tree at a time (the classic "Volcano" iterator model — high per-row overhead from repeated virtual function calls) or compiling one giant function per query (whole-stage codegen, Spark's approach — fast but a compile-time cost and less flexible for adaptive execution), DuckDB passes **vectors** — columnar batches of up to `STANDARD_VECTOR_SIZE` (2048) values — between operators. Each operator processes a whole 2048-value batch of one column at a time, in a tight loop over homogeneous data that compilers auto-vectorize into SIMD instructions (e.g. AVX2 comparing 8 int32s per instruction). This is a middle ground between per-row interpretation (too much overhead) and full query compilation (too rigid): batching amortizes per-operator dispatch cost while keeping the plan flexible and interpretable at each operator boundary.

**Morsel-driven parallelism**: the actual unit of parallel work is a **morsel** — a chunk of roughly 10,000 rows (small relative to the whole table, large relative to a single vector) — dynamically pulled from a shared work queue by whichever worker thread is free next. There is no static partitioning of data to threads ahead of time; a thread that finishes its morsel immediately grabs the next one from the queue. This is the direct fix for a classic parallel-scan problem: static partitioning (e.g. "thread 1 gets rows 0-999999, thread 2 gets rows 1000000-1999999") stalls the whole query on whichever thread's partition happens to be slowest (a skewed row group, a page-cache miss); dynamic morsel assignment means a fast thread just does more morsels instead of sitting idle waiting for a slow one.

```
10M-row scan, ~10K rows per morsel → ~1000 morsels in the shared queue

Thread 1: [morsel 1] [morsel 5] [morsel 9]  ... (grabs next when free)
Thread 2: [morsel 2] [morsel 6] [morsel 10] ...
Thread 3: [morsel 3] [morsel 7]             ... (this morsel was slow —
Thread 4: [morsel 4] [morsel 8] [morsel 11] ...  thread 4 just picks up
                                                   more work instead of
                                                   waiting on thread 3)

Within each morsel, vectorized operators process it in 2048-row batches —
morsels are the parallelism unit; vectors are the SIMD/cache-locality unit.
```

Pipelines of operators (scan → filter → hash-build, for instance) execute morsel-by-morsel without materializing the full intermediate result — one morsel flows through the whole pipeline before the next morsel starts, which is what keeps memory footprint bounded regardless of total table size. This combination — no shuffle (single shared-memory process) + no static partitioning stalls (dynamic morsel queue) + SIMD-friendly batching (vectors) — is the concrete, three-part reason DuckDB's resource-layer table above says CPU is optimized for tight vectorized loops and network is only about storage I/O, never inter-worker redistribution.

**Making the thread pool and morsel queue concrete**: DuckDB's morsel scheduler runs exactly as many worker threads as `threads` is set to (default: `std::thread::hardware_concurrency()`, i.e. all logical cores). This is directly settable and inspectable:

```sql
-- Inspect and override the worker thread count (the morsel-queue consumer count)
SELECT current_setting('threads');
SET threads = 8;
PRAGMA threads=4;              -- PRAGMA form is equivalent to SET for this setting
```

`EXPLAIN ANALYZE` makes the morsel/vector machinery visible per-operator — real rows processed and wall-clock time per pipeline stage, not just an estimated plan:

```sql
EXPLAIN ANALYZE
SELECT user_id, SUM(revenue) AS total_revenue
FROM read_parquet('s3://bucket/events/*.parquet')
WHERE event_date >= '2024-01-01'
GROUP BY user_id;
```

```
┌─────────────────────────────┐
│      HASH_GROUP_BY          │
│  ──────────────────────     │
│      Rows:   842,113        │
│      Time:   0.44s          │  ← build+probe of the GROUP BY hash table,
└─────────────┬───────────────┘    fed one morsel (~10K rows) at a time
┌─────────────┴───────────────┐
│      TABLE_SCAN (parquet)   │
│  ──────────────────────     │
│  Projections: user_id,      │
│    revenue, event_date      │
│  Rows scanned: 1,113,402    │  ← already post row-group-pruning; see the
│      Time:    0.62s         │    literal EXPLAIN in the walkthrough below
└──────────────────────────────┘
```

Each operator box's row count and timing is the aggregate across every morsel and every thread that touched it — this is what "morsel-driven parallelism" looks like from the outside, not just in the ASCII diagram above.

---

## High-to-Low Walkthrough: `read_parquet(...)` to Bytes Off S3

The literal query traced through every step below:

```sql
SELECT user_id, SUM(revenue) AS total_revenue
FROM read_parquet('s3://bucket/events/*.parquet')
WHERE event_date >= '2024-01-01'
GROUP BY user_id;
```

```
SELECT user_id, SUM(revenue) FROM read_parquet('s3://.../*.parquet')
WHERE event_date >= '2024-01-01' GROUP BY user_id
      │
      ▼
Parser + Binder                  DuckDB's own PEG parser (PostgreSQL-flavored
                                   dialect) produces a logical plan; the
                                   binder resolves table/column references
      │
      ▼
Optimizer                         rule-based + cost-based rewrites: predicate
                                   pushdown (push the WHERE filter down toward
                                   the scan), projection pushdown (only
                                   user_id/revenue/event_date ever get read),
                                   join reordering if applicable
      │
      ▼
Physical plan → Pipelines         plan is linearized into pipelines suitable
                                   for morsel-driven execution; operator
                                   choice happens here (e.g. hash aggregate
                                   for GROUP BY)
      │
      ▼
HTTPFS: footer-first read         before any row data, DuckDB issues an HTTP
(network)                          range request for the LAST ~1MB of the
                                   remote Parquet file (the footer lives at
                                   the end) — this reveals row group count,
                                   per-row-group min/max stats, and column
                                   offsets, all from one small request
      │
      ▼
Row-group + column pruning        using the footer stats, the query's
                                   event_date >= '2024-01-01' predicate rules
                                   out row groups whose max event_date is
                                   below that threshold; only user_id/revenue/
                                   event_date columns are ever targeted, not
                                   all columns in the file (see
                                   data-formats/parquet.md for how Parquet's
                                   own row-group/page stats make this possible)
      │
      ▼
Targeted range requests           DuckDB issues additional range requests for
(network)                          ONLY the surviving row groups' relevant
                                   column chunks — not the whole file, and in
                                   parallel across multiple files if the glob
                                   matches many
      │
      ▼
Morsels + vectors                 fetched bytes are decoded and handed out
(CPU, in-process memory)           as morsels to the worker thread pool; each
                                   morsel flows through filter → hash-
                                   aggregate as 2048-row vectors
      │
      ▼
Result materialized                returned as a DataFrame/Arrow table/etc,
                                    entirely within the calling process
```

No step in this chain crosses a process or network boundary except the S3 range requests themselves — everything after the bytes land locally is single-process, shared-memory execution.

### What Gets Pushed Down: Reading the `EXPLAIN` Output

`EXPLAIN` (no `ANALYZE`) shows the *planned* pushdown before execution — this is the artifact to check when you want to confirm filter/projection/row-group pruning will actually happen, without paying for a full run:

```sql
EXPLAIN SELECT user_id, SUM(revenue) AS total_revenue
FROM read_parquet('s3://bucket/events/*.parquet')
WHERE event_date >= '2024-01-01'
GROUP BY user_id;
```

```
┌───────────────────────────┐
│    HASH_GROUP_BY           │
│    #Groups: ~842113        │
└─────────────┬───────────────┘
┌─────────────┴───────────────┐
│         PROJECTION           │
│    user_id, revenue          │  ← only 2 of the file's ~10 columns
└─────────────┬───────────────┘     survive the projection pushdown
┌─────────────┴───────────────┐
│       PARQUET_SCAN           │
│  Filters: event_date>=       │  ← predicate pushed all the way to the
│    2024-01-01                │     scan, not applied after a full read
│  Projections: user_id,       │
│    revenue, event_date       │
│  Row groups total: 48        │
│  Row groups pruned: 31       │  ← eliminated via footer min/max stats,
│  Row groups scanned: 17      │     never fetched over the network at all
└───────────────────────────────┘
```

For a lower-level, byte-exact view of which row groups and column chunks were fetched, `EXPLAIN ANALYZE` combined with HTTP-level introspection is the next step down:

```sql
-- Cross-check actual bytes transferred against the plan above
SET enable_http_metadata_cache = true;   -- cache footer/metadata across queries
SELECT * FROM duckdb_settings() WHERE name LIKE '%http%';
```

The three pushdowns visible in the plan — **projection** (only `user_id`/`revenue`/`event_date` ever get decoded), **predicate/filter** (the `event_date` comparison is evaluated against row-group statistics, not row-by-row after the fact), and **row-group pruning** (31 of 48 groups are skipped entirely, meaning DuckDB never issues an HTTP range request for their column chunks) — are the concrete mechanism behind the "Row-group + column pruning" and "Targeted range requests" steps in the diagram above.

---

## Deep Internals

### Buffer Pool and Spill

DuckDB manages a fixed-size buffer pool (default: 80% of available RAM) holding data pages, hash tables for GROUP BY/joins, and sort buffers, with LRU eviction when full. When an intermediate structure (a hash aggregation table, a hash join's build side, an ORDER BY sort buffer, window function state) exceeds available memory, DuckDB spills it to a temp file and processes it partition-by-partition — a classic external (disk-backed) algorithm, transparent to the query: it completes, just slower. This is a real behavioral difference from a naive in-memory-only engine that would simply OOM past a size threshold — spill is what lets DuckDB process datasets larger than RAM at all, at a cost, rather than failing outright.

```sql
-- Cap the buffer pool explicitly instead of trusting the 80%-of-RAM default
SET memory_limit = '8GB';

-- Point spill files at fast local storage, not a slow network mount
SET temp_directory = '/fast-nvme/tmp';

-- Confirm both took effect
SELECT current_setting('memory_limit'), current_setting('temp_directory');
```

```sql
-- Introspect current memory usage and on-disk database size
PRAGMA database_size;

-- Per-buffer-manager-block memory accounting (finer-grained than database_size)
SELECT * FROM duckdb_memory();
```

```
┌──────────────────┬────────────┬───────────────┐
│ database_size    │ block_size │ memory_usage  │
│ ─────────────    │ ────────── │ ───────────── │
│ 42.3GB           │ 262144     │ 7.6GB / 8.0GB │  ← buffer pool near the
└──────────────────┴────────────┴───────────────┘     memory_limit ceiling
```

**Forcing and observing a spill** — a `GROUP BY` whose hash table is deliberately larger than the memory cap will spill partitions to `temp_directory`, visible in `EXPLAIN ANALYZE` as extra I/O time on the aggregate operator:

```sql
SET memory_limit = '200MB';   -- deliberately small, to force a spill
EXPLAIN ANALYZE
SELECT user_id, approx_count_distinct(session_id), SUM(revenue)
FROM read_parquet('s3://bucket/events/*.parquet')
GROUP BY user_id;
-- HASH_GROUP_BY operator's profile shows a nonzero "spilled to disk" byte count
-- once the build-side hash table exceeds the 200MB cap
```

### Hash Join

The smaller input is built into an in-memory hash table (stored in the buffer pool); the larger input is probed against it morsel-by-morsel, with multiple threads probing concurrently once the build phase completes. If the build side doesn't fit in memory, both sides get partitioned by hash key and spilled, processed partition-by-partition — the same fundamental external hash join algorithm used by disk-based databases for decades, just with DuckDB's vectorized/morsel machinery underneath each partition's processing.

```sql
EXPLAIN ANALYZE
SELECT e.user_id, e.revenue, u.country
FROM read_parquet('s3://bucket/events/*.parquet') e
JOIN read_parquet('s3://bucket/users/*.parquet') u USING (user_id);
```

```
┌───────────────────────────────┐
│           HASH_JOIN            │
│  ───────────────────────       │
│  Build side (users): 240,112   │  ← smaller relation built into the
│  Probe side (events): 8,401,933│     hash table; events streamed/probed
│  Time (build): 0.18s           │
│  Time (probe): 1.02s           │
└───────────────────────────────┘
```

**Window functions and ASOF joins** are useful concrete illustrations of vectorized execution on real SQL — both process ordered batches of rows through the same vector pipeline rather than requiring a separate execution model:

```sql
-- Running 7-day revenue total per user, computed via vectorized window execution
SELECT user_id, event_date, revenue,
       SUM(revenue) OVER (
           PARTITION BY user_id ORDER BY event_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS trailing_7d_revenue
FROM read_parquet('s3://bucket/events/*.parquet');

-- ASOF JOIN: match each trade to the most recent quote at or before its timestamp
SELECT t.trade_id, t.ts AS trade_ts, q.ts AS quote_ts, q.price
FROM trades t
ASOF JOIN quotes q ON t.symbol = q.symbol AND t.ts >= q.ts;
```

### Reading Iceberg/Delta Directly

```sql
INSTALL iceberg; LOAD iceberg;
SELECT * FROM iceberg_scan('s3://bucket/warehouse/db/table');
-- reads Iceberg metadata JSON → snapshot → manifest files → data files,
-- pruning at the manifest level the same way Parquet row-group stats
-- prune within a file (see data-formats/apache-iceberg.md)
```

```sql
-- Time-travel read against a specific snapshot, and a write against a v2 table
INSTALL iceberg; LOAD iceberg;
SELECT * FROM iceberg_scan('s3://bucket/warehouse/db/table', snapshot_from_id => 619123456789);

INSERT INTO iceberg_scan('s3://bucket/warehouse/db/table')
SELECT * FROM read_parquet('s3://bucket/staging/new_batch.parquet');
```

```sql
-- Delta reads/writes: append-only INSERT and history are supported as of 1.5.3;
-- overwrite/DDL/MERGE/DELETE/UPDATE on Delta are NOT yet shipped (see Gotchas)
INSTALL delta; LOAD delta;
SELECT * FROM delta_scan('s3://bucket/warehouse/delta_table');

-- Time-travel via history
SELECT * FROM delta_scan('s3://bucket/warehouse/delta_table', version => 12);

-- Append-only write (the one Delta write path DuckDB currently supports)
INSERT INTO delta_scan('s3://bucket/warehouse/delta_table')
SELECT * FROM read_parquet('s3://bucket/staging/new_batch.parquet');
```

As of the 1.4/1.5 line (1.4 LTS shipped September 2025, refined through 1.5.5 in July 2026), **Iceberg write support has moved well past read-only**: `INSERT`, `MERGE INTO`, and `ALTER TABLE` are supported for Iceberg v2 tables, delete/update landed in 1.4.2, and 1.5.3 added Iceberg v3 format support plus bucket/truncate partition transforms. **Delta write support** graduated from experimental in 1.5.2 (stabilized in 1.5.3): `INSERT` (append-only), history time-travel, and Unity Catalog-governed read/write are available; overwrite mode, DDL (CREATE/ALTER), and delete/update/merge on Delta remain roadmap items, not yet shipped — don't assume Delta write parity with Iceberg's broader support.

### Query Planning Pipeline

```
SQL string → Parser (own PEG parser, PostgreSQL dialect) → Binder (resolve
refs) → Optimizer (rule-based + cost-based, using column stats from Parquet
footers or the catalog) → Physical plan (operator selection: hash vs.
sort-merge join, etc.) → Pipeline builder (linearize for morsel execution)
→ Executor (morsel scheduler + thread pool + vectorized operators)
```

Each stage is directly inspectable via `EXPLAIN` output-format controls, without running the query:

```sql
-- Logical plan only (post-optimizer, pre-physical-plan)
SET explain_output = 'optimized_only';
EXPLAIN SELECT user_id, SUM(revenue) FROM read_parquet('s3://bucket/events/*.parquet')
GROUP BY user_id;

-- Physical plan (operator selection: which join/aggregate strategy was chosen)
SET explain_output = 'physical_only';
EXPLAIN SELECT user_id, SUM(revenue) FROM read_parquet('s3://bucket/events/*.parquet')
GROUP BY user_id;

-- Both, side by side (the default)
SET explain_output = 'all';
```

---

## Comparative

**vs. Spark** (`compute/spark.md`): the decisive difference is the shuffle. Spark's wide transformations (`groupBy`, `join`) redistribute data across physically separate executor processes over the network — that redistribution is what shuffle *is*, and it's the dominant cost in most large Spark jobs. DuckDB has no equivalent step: all threads share one process's memory, so a `GROUP BY` is "scan morsels in parallel, build/probe a shared or partitioned hash table," never "serialize rows, send over network, deserialize on the other side." This is precisely why DuckDB is not a Spark replacement for genuinely-distributed-scale data — it's a different regime entirely, valid exactly as long as the working set fits (with spill) on one machine.

**vs. Polars**: both are embedded, columnar, in-process, Arrow-interoperable. The practical difference is interface-first design, not raw capability: DuckDB is SQL-first (with a lazy Relation API as an alternative), Polars is DataFrame/method-chaining-first (with `.sql()` as an alternative) — pick based on whether your team thinks in SQL or in DataFrame pipelines; both support lazy execution and out-of-core (spilling) processing, though DuckDB's Iceberg/Delta/HTTPFS support is native and more mature than Polars' connector-based equivalents as of 2026.

```python
# The same aggregation, DuckDB SQL-first vs. Polars DataFrame-first —
# both execute vectorized, in-process, no shuffle:
import duckdb
duckdb.sql("""
    SELECT user_id, SUM(revenue) FROM read_parquet('events/*.parquet')
    GROUP BY user_id
""")

import polars as pl
pl.scan_parquet("events/*.parquet") \
    .group_by("user_id") \
    .agg(pl.col("revenue").sum()) \
    .collect()

# Polars can also just run the SQL directly, via its own SQL context:
pl.sql("SELECT user_id, SUM(revenue) FROM read_parquet('events/*.parquet') GROUP BY user_id")
```

---

## Key Gotchas

- **Single-writer limitation**: DuckDB allows multiple concurrent readers but only one writer connection to a given `.duckdb` file at a time (optimistic single-writer via OS file locks, no MVCC/row-level locking like Postgres). Don't design a multi-process web service or multiple Airflow workers to write to the same `.duckdb` file — use Parquet-as-storage (DuckDB reads, pipeline writes Parquet) to sidestep this entirely, or MotherDuck for managed multi-writer coordination.
  ```sql
  -- A second connection attempting to write here will fail fast, not hang:
  -- Conflicting lock is held! ... file is already open in <other process>
  ATTACH 'analytics.duckdb' AS db (READ_ONLY);  -- readers should attach read-only
  ```
- **S3 cold start per query**: the first query against a new remote file pays footer-fetch latency (~100-500ms per file) before any pruning can happen; subsequent queries against already-touched files are faster. Latency-sensitive services should keep a warm connection/cached metadata rather than cold-querying S3 per request.
  ```sql
  SET enable_http_metadata_cache = true;  -- cache footers across queries on one connection
  SET enable_object_cache = true;         -- cache decoded Parquet metadata objects too
  ```
- **Memory accounting is buffer-pool-only**: `memory_limit` governs DuckDB's own buffer pool, not Python-side objects (a returned Pandas DataFrame or Arrow table lives outside that pool) — total process RAM usage is `memory_limit` *plus* whatever your host language materializes from the result, not capped by `memory_limit` alone.
  ```python
  # con.execute(...).df() materializes a full Pandas DataFrame OUTSIDE memory_limit;
  # prefer .fetch_arrow_table() / streaming .fetch_df_chunk() for large results
  con.execute("SELECT * FROM read_parquet('events/*.parquet')").fetch_arrow_table()
  ```
- **Inconsistent schemas across a Parquet glob silently break `SELECT *`**: data lakes commonly have files with slightly different column sets/types across partitions.
  ```sql
  SELECT * FROM read_parquet('s3://bucket/events/*.parquet', union_by_name=true);
  ```
- **Extensions load per-connection**: `httpfs`, `iceberg`, `delta`, etc. must be `LOAD`ed on each connection — in multiprocessing contexts (e.g. Lambda cold starts, worker pools), every new process/connection needs its own `INSTALL`/`LOAD`, it isn't inherited from a prior process.
  ```sql
  INSTALL httpfs; LOAD httpfs;
  INSTALL iceberg; LOAD iceberg;
  INSTALL delta; LOAD delta;
  SELECT * FROM duckdb_extensions() WHERE loaded = true;  -- verify per-connection state
  ```
- **Delta write support lags Iceberg's**: as of 1.5.3, Delta only supports append-only `INSERT` plus history/time-travel reads and Unity Catalog integration — reaching for `MERGE INTO` or `DELETE`/`UPDATE` semantics on a Delta table via DuckDB will fail; Iceberg's write surface is materially broader at the same point in time.
  ```sql
  -- Fails today against a delta_scan target — Iceberg's INSERT/MERGE/ALTER TABLE
  -- support (shown above) does NOT extend to Delta yet:
  MERGE INTO delta_scan('s3://bucket/warehouse/delta_table') AS t
  USING staging AS s ON t.id = s.id
  WHEN MATCHED THEN UPDATE SET revenue = s.revenue;  -- NOT YET SUPPORTED for Delta
  ```
- **"Fits on one machine" is a moving, spill-adjusted target, not a hard byte ceiling**: DuckDB can process datasets meaningfully larger than RAM via spill, but spill cost scales with how much of the working set (not the raw source data) exceeds memory — a 10GB source table with a huge skewed GROUP BY can spill far more punishingly than a 500GB table queried with a highly selective filter and small aggregation state. Size intuition around the *intermediate* state, not the source data volume.
  ```sql
  -- Check spill activity directly rather than guessing from source data size:
  PRAGMA database_size;
  SELECT * FROM duckdb_memory() ORDER BY memory_usage_bytes DESC LIMIT 5;
  ```

---

*Grounded against duckdb.org release notes/blog posts and community internals writeups as of September 2026. Current stable release: 1.5.5 (July 2026), on the 1.4 LTS/1.5.x line. Vector size defaults to 2048 rows; morsel size is roughly 10K rows, dynamically assigned from a shared work queue, not statically partitioned. Iceberg/Delta write-support maturity (noted above) is an actively moving target — re-verify exact operation support (MERGE/DELETE/UPDATE/DDL) against the specific version in use before relying on it in a deliverable.*
