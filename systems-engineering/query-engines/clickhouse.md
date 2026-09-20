# ClickHouse — Deep Technical Reference

> Target: senior backend/distributed systems engineer. Fast recall + deep intuition.

---

## 30-Second Intuition

ClickHouse is a **columnar OLAP database** built for sub-second analytical queries over billions of rows. It is not a general-purpose database.

**The core problem it solves:**

| Scenario | OLTP (Postgres) | Spark SQL | ClickHouse |
|---|---|---|---|
| Point lookup (1 row) | <1ms | seconds | 10–50ms (overkill) |
| `GROUP BY` over 1B rows | minutes/OOM | 30–120s | **0.1–2s** |
| Write throughput | high, row-at-a-time | batch only | high batch, low cardinality inserts |
| Operational complexity | low | high (cluster) | medium |

**Mental model:**
- OLTP stores data row-by-row: `[user_id, ts, event, country, ...]` all on one page → bad for `SELECT count(*) WHERE country='US'` because you load every column.
- ClickHouse stores each column in its own compressed file → a query touching 2 of 50 columns reads **4%** of the data.
- Spark reads the same columnar files (Parquet) but has JVM overhead, scheduling latency, and shuffle cost. ClickHouse's query engine is a tight C++ vectorized pipeline — latency is fundamentally lower.

**When to use ClickHouse:** Real-time analytics dashboards, log/event analytics, time-series aggregation, clickstream analysis.  
**When NOT to use:** Transactional workloads (no real UPDATE), complex multi-table JOINs with many small tables, OLTP.

---

## Storage Engine — MergeTree

MergeTree is ClickHouse's primary storage engine. Everything interesting builds on it.

### How Inserts Create Parts

Every `INSERT` creates a **data part** on disk. Parts are immutable once written.

```
INSERT batch 1  →  part_1 (dir on disk)
INSERT batch 2  →  part_2
INSERT batch 3  →  part_3
                     ↓ background merge
                   part_1_2_3 (merged part, originals deleted)
```

**Part = a directory** containing:
```
/var/lib/clickhouse/data/<db>/<table>/
  └── 20240101_1_1_0/          ← part name: partition_minblock_maxblock_level
        ├── primary.idx         ← sparse primary key index
        ├── column1.bin         ← compressed column data
        ├── column1.mrk3        ← marks file: granule → byte offset
        ├── column2.bin
        ├── column2.mrk3
        ├── checksums.txt
        ├── columns.txt         ← column schema
        └── count.txt           ← row count
```

### The Sparse Index (primary.idx)

ClickHouse does NOT use a B-tree. It uses a **sparse index**: one index entry per granule (~8192 rows by default).

```
Rows:        0     8192   16384  24576   32768
Index:    [key0] [key1]  [key2] [key3]  [key4]
              ↑ granule 0  ↑ g1   ↑ g2   ↑ g3
```

**Why sparse?** A B-tree over 1 billion rows has millions of nodes. A sparse index over 1B rows has ~122,000 entries — fits in RAM easily and enables fast binary search.

**Trade-off:** You can only efficiently filter on the **leftmost prefix** of the sort key. A WHERE on column 3 of the sort key requires scanning more granules.

### Granule — The Unit of I/O

A **granule** is the atomic unit ClickHouse reads. Default = 8192 rows.

```
Column file (column1.bin):
  [granule 0: 8192 rows compressed] [granule 1: 8192 rows] [granule 2: ...]

Marks file (column1.mrk3):
  granule 0 → byte offset 0,       row offset 0
  granule 1 → byte offset 14302,   row offset 8192
  granule 2 → byte offset 27891,   row offset 16384
```

The `.mrk3` file is a lookup table: given a granule number, jump directly to the correct byte in the compressed column file. This makes skipping irrelevant granules O(1).

### Query Execution — Granule Pruning

```sql
SELECT count() FROM events WHERE user_id BETWEEN 1000 AND 2000
-- Sorting key: ORDER BY user_id, ts
```

1. Binary search `primary.idx` for first granule where `user_id >= 1000`
2. Scan forward in index until `user_id > 2000`
3. Load ONLY those granules from `user_id.bin`
4. Filter rows within granules

```
primary.idx:  [800] [950] [1050] [1800] [2100] [2500]
                              ↑               ↑
                         start granule    stop granule
                         (granule 2)      (granule 4)
→ Read only granules 2 and 3, skip the rest
```

**Granule pruning is the core performance mechanism.** Design your sort key so your most common WHERE clauses hit its prefix.

### Primary Key vs Sorting Key

ClickHouse uniquely separates these:

```sql
CREATE TABLE events (
    user_id UInt64,
    ts DateTime,
    event_type String,
    value Float64
)
ENGINE = MergeTree
ORDER BY (user_id, ts)       -- sorting key: physical sort order on disk
PRIMARY KEY (user_id)        -- subset used for the sparse index
```

- `ORDER BY` controls the physical sort order (and thus what can be range-pruned)
- `PRIMARY KEY` controls what goes into `primary.idx` (must be a prefix of ORDER BY)

**Why differ them?** The primary key is loaded into RAM for binary search. You can keep it smaller (fewer columns) to save memory while still having a richer sort order for range scans.

If you don't specify `PRIMARY KEY`, it defaults to `ORDER BY`.

### Skip Indices (Secondary Indices)

Skip indices let ClickHouse skip granules based on metadata stored per-granule.

```sql
ALTER TABLE events ADD INDEX idx_country country TYPE set(100) GRANULARITY 4;
-- GRANULARITY 4 = one skip index entry per 4 granules (4 × 8192 = 32768 rows)
```

| Type | Stores per block | Skips when |
|---|---|---|
| `minmax` | min/max of column | WHERE col > X and max < X |
| `set(N)` | set of distinct values (up to N) | WHERE col = X and X not in set |
| `bloom_filter` | probabilistic membership | WHERE col = X and bloom says not present |
| `ngrambf_v1(n, size, hashes, seed)` | bloom of n-grams | `LIKE '%substring%'` acceleration |
| `tokenbf_v1` | bloom of tokens | `hasToken(col, 'word')` |

**When each helps:**
- `minmax`: monotonic data, time ranges, numeric ranges
- `set`: low-cardinality enum-like columns (status, country code)
- `bloom_filter`: high-cardinality equality lookups (user IDs, UUIDs) — false positives possible
- `ngrambf_v1`: full-text substring search on log messages

**Important:** Skip indices are advisory — they can only skip granules, never guarantee a match. A bloom filter false positive means you read the granule anyway.

### Data Part Lifecycle

```
INSERT → new part (active, level=0)
               ↓ background MergeTreeMutations
         merged part (active, higher level)
               ↓ all replicas confirm
         old parts → obsolete → deleted after TTL
```

ClickHouse tracks part state in ZooKeeper/Keeper and on local disk. The merger runs continuously in background threads. Part level indicates how many times it has been merged (level N = merged N times from level 0).

**Mutation** (UPDATE/DELETE in ClickHouse) rewrites entire parts — it's expensive and async. Prefer append-only designs.

### TTL (Time-To-Live)

**Row TTL** — delete rows after expiry:
```sql
CREATE TABLE logs (
    ts DateTime,
    message String
)
ENGINE = MergeTree
ORDER BY ts
TTL ts + INTERVAL 30 DAY;
```

**Column TTL** — null out a column after expiry (keep row, save space):
```sql
CREATE TABLE events (
    ts DateTime,
    user_id UInt64,
    raw_payload String TTL ts + INTERVAL 7 DAY  -- null after 7 days
)
ENGINE = MergeTree ORDER BY ts;
```

**Move TTL** — hot/warm/cold tiering:
```sql
TTL ts + INTERVAL 7 DAY TO DISK 'warm_ssd',
    ts + INTERVAL 90 DAY TO VOLUME 'cold_hdd';
```

TTL is enforced during merges (not immediately on expiry). Force evaluation: `OPTIMIZE TABLE ... FINAL`.

---

## MergeTree Variants

### ReplacingMergeTree — Deduplication

```sql
CREATE TABLE user_profiles (
    user_id UInt64,
    name String,
    email String,
    updated_at DateTime
)
ENGINE = ReplacingMergeTree(updated_at)  -- version column: keep highest value
ORDER BY user_id;
```

**How it works:** During merge, rows with the same `ORDER BY` key are collapsed — only the one with the highest version column value survives.

**Gotchas:**
- Deduplication is **eventual** — before merge, duplicates exist. Queries may see stale rows.
- Use `SELECT ... FINAL` to force dedup at query time (expensive, disables parallelism):
  ```sql
  SELECT * FROM user_profiles FINAL WHERE user_id = 42;
  ```
- `FINAL` scans all parts and deduplicates in memory — slow on large tables. Prefer `argMax` pattern:
  ```sql
  SELECT user_id, argMax(email, updated_at) as email
  FROM user_profiles GROUP BY user_id;
  ```

**Use when:** You need upsert semantics (e.g., user profile updates, CDC).

### AggregatingMergeTree — Pre-aggregation

Stores **partial aggregate states**, not final values. Combine with materialized views for real-time pre-aggregation.

```sql
CREATE TABLE dau_agg (
    date Date,
    country LowCardinality(String),
    users AggregateFunction(uniq, UInt64)   -- stores HyperLogLog state
)
ENGINE = AggregatingMergeTree
ORDER BY (date, country);
```

During merge, rows with the same key have their `AggregateFunction` states merged using the combinator functions.

**Use when:** You need to pre-aggregate at insert time and query partial results later.

### CollapsingMergeTree — CDC-Style Updates

Uses a `sign` column (+1 or -1). Rows cancel each other out during merge.

```sql
CREATE TABLE orders (
    order_id UInt64,
    amount Float64,
    sign Int8  -- +1 = insert/current state, -1 = cancel old state
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY order_id;

-- Update order: insert cancellation + new state
INSERT INTO orders VALUES (1, 100.0, -1);  -- cancel old
INSERT INTO orders VALUES (1, 150.0, +1);  -- new state
```

**VersionedCollapsingMergeTree** adds a version column so out-of-order arrival still works:
```sql
ENGINE = VersionedCollapsingMergeTree(sign, version)
```

**Use when:** CDC pipelines (Debezium → ClickHouse), mutable fact tables.

### SummingMergeTree — Auto-Summation

Automatically sums numeric columns during merge for rows with the same key:

```sql
CREATE TABLE page_views (
    date Date,
    page_id UInt32,
    views UInt64,
    clicks UInt64
)
ENGINE = SummingMergeTree
ORDER BY (date, page_id);
```

**Use when:** Additive metrics where you never need row-level history — counters, sums.

### Variant Selection Guide

| Variant | Use Case | Key Consideration |
|---|---|---|
| MergeTree | Append-only events, logs | Default choice |
| ReplacingMergeTree | Upserts (profiles, state) | Eventual dedup; use FINAL or argMax |
| AggregatingMergeTree | Pre-aggregation pipeline | Must use -State/-Merge combinators |
| CollapsingMergeTree | CDC with ordered events | Pairs of +1/-1 rows |
| VersionedCollapsingMergeTree | CDC with out-of-order events | Adds version column |
| SummingMergeTree | Simple additive counters | Non-additive columns ignored |

---

## Columnar Execution

### Why Columnar Crushes Row-Oriented for Analytics

```
Row store (Postgres heap page):
  [user_id=1][ts=...][country='US'][event='click'][value=1.5]
  [user_id=2][ts=...][country='UK'][event='view'][value=0.0]
  ...
  ← to compute SUM(value), you load ALL columns from disk

Column store (ClickHouse .bin file):
  value.bin: [1.5][0.0][2.1][0.8][3.3]...
  ← to compute SUM(value), load ONLY this file
```

**Benefits:**
1. **I/O reduction**: Query touching 3 of 50 columns reads 6% of data
2. **Cache efficiency**: CPU cache holds contiguous homogeneous values
3. **SIMD**: Same-type values in registers → AVX-512 can process 16 floats/cycle
4. **Compression**: `['US','US','US','UK','US']` compresses far better than mixed rows

### Vectorized Execution

ClickHouse processes data in **vectors of 8192 rows** (one granule) through a pipeline of operators.

```
Column chunk [8192 floats] → Filter → Aggregate → Output
                               ↑
                         CPU processes 8192 values
                         using SIMD before the next
                         operator sees any results
```

Compare to Volcano/iterator model (Postgres): one row at a time, massive function call overhead. Vectorized execution amortizes function call cost over 8192 rows.

### Compression Codecs

ClickHouse applies codecs in a pipeline before LZ4/ZSTD compression:

```sql
CREATE TABLE metrics (
    ts DateTime CODEC(DoubleDelta, LZ4),
    value Float32 CODEC(Gorilla, LZ4),
    counter UInt64 CODEC(Delta(4), ZSTD(3))
)
ENGINE = MergeTree ORDER BY ts;
```

| Codec | Best For | Why |
|---|---|---|
| `LZ4` | Default, fast decompression | ~2GB/s decompress; good ratio |
| `ZSTD(level)` | Cold/archival data | Better ratio, slower |
| `Delta` | Monotonic integers (counters) | Stores differences; `[100,101,102]` → `[100,1,1]` |
| `DoubleDelta` | Timestamps, monotonic sequences | Delta of deltas; near-zero for steady cadence |
| `Gorilla` | Float time-series | XOR of consecutive floats; leading zeros compress well |
| `T64` | Small integers with outliers | Transposes bits; efficient for clustered values |

**Example — why Delta is magic for time-series:**
```
Raw timestamps:    1700000000, 1700000060, 1700000120, 1700000180
Delta encoding:    1700000000, 60, 60, 60
DoubleDelta:       1700000000, 60, 0, 0  ← nearly all zeros → ~100x compression
```

**Codec chains** apply left to right before final compression. `CODEC(Delta, LZ4)` first delta-encodes, then LZ4 compresses the resulting small numbers.

---

## Query Execution

### Pipeline Architecture

```
SQL text
   ↓ Parser (Lexer + recursive descent)
   ↓ AST (Abstract Syntax Tree)
   ↓ Query Analyzer (type checking, alias resolution)
   ↓ Query Plan (logical: scan, filter, aggregate, sort)
   ↓ Pipeline Builder → Processor DAG
   ↓ Execution (parallel, pull-based)
```

The execution pipeline is a DAG of **Processors** connected by queues. Each processor has input ports and output ports. Data flows as `Chunk` objects (column-oriented blocks of rows).

### Parallel Query Execution

```
Query: SELECT sum(value) FROM events WHERE ts > now() - INTERVAL 1 DAY

Parts on disk: [part_1] [part_2] [part_3] [part_4] [part_5]
                  ↓         ↓         ↓         ↓         ↓
              Thread1   Thread2   Thread3   Thread4   Thread5
              partial   partial   partial   partial   partial
              sum       sum       sum       sum       sum
                  ↘        ↘        ↓       ↙       ↙
                      MergingAggregatedTransform
                              ↓
                          Final sum
```

`max_threads` controls parallelism (default: CPU core count). Each thread processes one or more parts independently.

### Distributed Query Execution

```
Client
  ↓
Initiator node (receives query)
  ↓ fan-out via Distributed table engine
  ├── Shard 1 (local execution → partial result)
  ├── Shard 2 (local execution → partial result)
  └── Shard 3 (local execution → partial result)
  ↓ Initiator collects & merges partial results
Client receives final result
```

Two-stage aggregation:
1. Each shard computes partial `GROUP BY` locally
2. Initiator merges partial results using `sumMerge`, `uniqMerge`, etc.

For `ORDER BY` + `LIMIT`: shards send top-K results; initiator merges and takes global top-K. ClickHouse does NOT sort the full dataset on each shard — it uses a priority queue.

### GROUP BY — Hash Table Approach

```sql
SELECT country, count() FROM events GROUP BY country
```

ClickHouse builds a **hash table** in memory: `{country_value → aggregate_state}`. For high-cardinality GROUP BY (millions of groups), this requires significant RAM.

Two-level aggregation kicks in automatically: if the hash table exceeds a threshold, ClickHouse partitions keys by hash prefix and processes them in two passes (prevents OOM).

Settings:
```sql
SET max_bytes_before_external_group_by = 10000000000;  -- spill to disk at 10GB
SET group_by_two_level_threshold = 100000;              -- switch to two-level
```

### JOIN Internals — The Historical Gotcha (Now Largely Fixed)

Historically, ClickHouse JOINs were **not magic**: the optimizer did not reorder join sides, and you had to manually put the smaller table on the right (the side loaded into the hash table).

```sql
-- Historically "correct": smaller table on the RIGHT (hash table built from right)
SELECT e.user_id, u.country
FROM events e
JOIN (SELECT user_id, country FROM users) u ON e.user_id = u.user_id

-- Historically "slow": if users table is large, hash table is huge; also built from wrong side
SELECT e.user_id, u.country
FROM users u  ← large table on left
JOIN events e ON u.user_id = e.user_id
```

**This has changed as of ClickHouse 24.12+ and especially 25.9+ (2025-2026 releases):**
- As of **24.12**, the query planner automatically places the smaller table on the right side of the join — you no longer have to do this manually for simple two-table joins.
- **25.9** introduced **global join reordering** (`query_plan_optimize_join_order_algorithm`, e.g. `dpsize,greedy`) that determines optimal build/probe order across **more than two tables**, using column statistics. This has produced dramatic speedups on multi-way joins (reported up to ~1,450× on TPC-H SF100 in some benchmarks) and now covers INNER, LEFT/RIGHT, ANTI, SEMI, and FULL joins (previously limited to INNER/LEFT/RIGHT).
- Complementary optimizations — equivalence-set filter pushdown and runtime bloom filters (default since ~Feb 2026) for star-schema fact/dimension joins — further reduce the penalty of "wrong-side" joins.

**Practical guidance for 2026:** putting the smaller table on the right is no longer a hard requirement for correctness/performance on recent versions (24.12+), but it remains good practice for older clusters, for joins the optimizer doesn't reorder, or when statistics are unavailable/stale. Always check `EXPLAIN` to see the actual chosen join order rather than assuming manual placement is still required.

**How ClickHouse hash join works (single join, no reordering applied):**
1. Read RIGHT table entirely → build hash table in memory
2. Stream LEFT table through → probe hash table per row

**Available join algorithms:**
- `hash` (default): right side in memory — fast if right fits in RAM
- `parallel_hash`: multiple hash tables built in parallel
- `grace_hash`: spills to disk; handles right side > RAM
- `merge`: sort-merge join; both sides must be sorted; good for large-large joins
- `direct`: only for Dictionary or EmbeddedRocksDB right side — O(1) lookup

```sql
-- Force join algorithm
SELECT ... JOIN ... USING (...) SETTINGS join_algorithm = 'grace_hash';

-- Control join reordering (25.9+)
SET query_plan_optimize_join_order_algorithm = 'dpsize,greedy';
```

### ORDER BY + LIMIT Optimization

```sql
SELECT * FROM events ORDER BY ts DESC LIMIT 100
```

ClickHouse uses a **partial sort** with a bounded heap of size `LIMIT`. It does not sort the full dataset. As each granule is read, only the top-100 candidates are maintained.

---

## Materialized Views

### How They Work

A materialized view is a **trigger** on INSERT, not a cached query result.

```sql
-- Source table
CREATE TABLE raw_events (ts DateTime, user_id UInt64, event String)
ENGINE = MergeTree ORDER BY ts;

-- Target table
CREATE TABLE event_counts (date Date, event String, cnt UInt64)
ENGINE = SummingMergeTree ORDER BY (date, event);

-- Materialized view: runs on every INSERT to raw_events
CREATE MATERIALIZED VIEW mv_event_counts TO event_counts AS
SELECT toDate(ts) as date, event, count() as cnt
FROM raw_events
GROUP BY date, event;
```

When data is inserted into `raw_events`:
1. Data lands in `raw_events`
2. MV SELECT runs on **the inserted block only** (not the whole table)
3. Result is inserted into `event_counts`
4. SummingMergeTree merges sums in background

### AggregatingMergeTree + MV Pattern

The standard pattern for real-time pre-aggregation:

```sql
-- Target table: stores partial aggregate states
CREATE TABLE dau_states (
    date Date,
    country LowCardinality(String),
    users AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (date, country);

-- MV: insert partial states on each batch
CREATE MATERIALIZED VIEW mv_dau TO dau_states AS
SELECT
    toDate(ts) as date,
    country,
    uniqState(user_id) as users      -- ← -State combinator: stores HLL state
FROM raw_events
GROUP BY date, country;

-- Query: merge states across parts
SELECT date, country, uniqMerge(users) as dau
FROM dau_states
GROUP BY date, country
ORDER BY date;
```

**The `-State` / `-Merge` combinator pair:**
- `uniqState(x)` → serializes HyperLogLog state to binary blob
- `uniqMerge(state_col)` → merges multiple blobs into final count
- Other pairs: `sumState`/`sumMerge`, `avgState`/`avgMerge`, `quantileState`/`quantileMerge`

### Cascading Materialized Views

A MV can target a table that itself has a MV:

```
raw_events → mv_hourly → hourly_counts
                                 ↓
                        mv_daily → daily_counts
```

**Warning:** Cascading MVs are fragile. If one target table schema changes, downstream breaks silently.

### MV Gotchas

- **MVs only fire on INSERT.** `ALTER TABLE ... UPDATE` (mutations) do NOT trigger MVs.
- **MVs see only the inserted block**, not the full table. Aggregations that require cross-row context (like ranking) behave unexpectedly.
- If the MV SELECT fails, the insert to the source table **still succeeds** (by default). Data is lost silently. Enable `materialized_views_ignore_errors = 0` to fail fast.
- Adding a MV to an existing table does NOT backfill. Must do: `INSERT INTO target SELECT ... FROM source`.

---

## Replication & Distribution

### ReplicatedMergeTree

```sql
CREATE TABLE events ON CLUSTER '{cluster}' (
    ts DateTime,
    user_id UInt64
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (user_id, ts);
```

The path `/clickhouse/tables/{shard}/events` is the ZooKeeper node. Macros `{shard}` and `{replica}` are substituted per server from `config.xml`.

**Replication mechanism (pull-based, not push):**
```
Replica A (leader): receives INSERT → writes part locally
                                          ↓ posts part info to ZooKeeper log
Replica B (follower): watches ZK log → sees new part entry
                                          ↓ fetches part from Replica A (HTTP)
                                          ↓ applies locally
```

Key ZooKeeper nodes:
- `/log` — ordered log of mutations
- `/replicas/{name}/queue` — per-replica work queue
- `/parts` — list of active parts across all replicas

### ClickHouse Keeper

A drop-in replacement for ZooKeeper, built in C++ using the **Raft consensus protocol**. Runs as a separate process or embedded. Preferred for new deployments:

```xml
<!-- config.xml -->
<zookeeper>
    <node><host>keeper1</host><port>9181</port></node>
    <node><host>keeper2</host><port>9181</port></node>
    <node><host>keeper3</host><port>9181</port></node>
</zookeeper>
```

Keeper is faster than ZooKeeper for ClickHouse workloads (fewer round-trips, better compression of ZK ops).

### Distributed Table Engine

The Distributed engine is a **virtual table** — it stores no data, only routes queries.

```sql
-- On every node: points to the local ReplicatedMergeTree shards
CREATE TABLE events_distributed ON CLUSTER '{cluster}'
AS events  -- same schema
ENGINE = Distributed(
    '{cluster}',   -- cluster name from config.xml
    'default',     -- database
    'events',      -- local table name
    rand()         -- sharding key: which shard gets each INSERT row
);
```

**INSERT routing:** Each row is hashed by sharding key → sent to the correct shard.

**SELECT routing:** Query fans out to all shards (or a subset for `SAMPLE`). Each shard executes locally, returns partial result to initiator.

**Sharding key design:**
- `rand()`: even distribution, no affinity (can't colocate related rows)
- `user_id`: colocates user's rows on same shard (enables shard-local JOINs)
- `cityHash64(user_id)`: same as above but hashed for even distribution

### ON CLUSTER DDL

```sql
CREATE TABLE events ON CLUSTER '{cluster}' (...) ENGINE = ...;
ALTER TABLE events ON CLUSTER '{cluster}' ADD COLUMN new_col String;
DROP TABLE events ON CLUSTER '{cluster}';
```

`ON CLUSTER` propagates the DDL to all nodes via ZooKeeper. Without it, DDL only runs on the connected node.

**Caveat:** `ON CLUSTER` DDL is not transactional. If a node is down, it misses the DDL. Re-run or use `ON CLUSTER` with retries.

---

## Ingestion Patterns

### Format Throughput Comparison

| Format | Throughput | Use Case |
|---|---|---|
| Native | Highest (~1–3 GB/s) | ClickHouse-to-ClickHouse, client drivers |
| RowBinary | Very high | Binary row format, low parsing overhead |
| Parquet | High | Data lake ingestion, columnar pre-grouped |
| JSONEachRow | Medium | Log pipelines, debugging |
| CSV | Medium | Batch file loads |

**Native format** is ClickHouse's binary columnar wire format — columns are sent pre-grouped, no parsing overhead.

### Async INSERT

Small frequent inserts are dangerous (see "too many parts" below). Async INSERT buffers them server-side:

```sql
-- Enable async insert
SET async_insert = 1;
SET wait_for_async_insert = 0;          -- fire and forget
SET async_insert_max_data_size = 10000000;   -- flush at 10MB
SET async_insert_busy_timeout_ms = 200;      -- flush after 200ms max

INSERT INTO events VALUES (...);  -- buffered; doesn't create a part immediately
```

The server accumulates inserts in memory and flushes a single merged part when size or timeout threshold is hit.

### Buffer Table Engine

In-memory buffer that auto-flushes to a destination table:

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(
    'default',        -- target database
    'events',         -- target table
    16,               -- num buffers (parallelism)
    10,               -- min time (seconds) before flush
    100,              -- max time (seconds) before flush
    10000,            -- min rows before flush
    1000000,          -- max rows before flush
    10000000,         -- min bytes before flush
    100000000         -- max bytes before flush
);

-- Write to buffer; reads from both buffer and target
INSERT INTO events_buffer VALUES (...);
SELECT * FROM events_buffer;  -- merges buffer + target transparently
```

**Drawback:** Buffer data is lost on server crash. Not suitable for critical data.

### Kafka Table Engine

Consume directly from Kafka topics:

```sql
CREATE TABLE kafka_events (
    ts UInt64,
    user_id UInt64,
    event String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'broker1:9092,broker2:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 4;

-- Materialized view to persist consumed data
CREATE MATERIALIZED VIEW mv_kafka_to_events TO events AS
SELECT toDateTime(ts) as ts, user_id, event
FROM kafka_events;
```

**How it works:** The Kafka table is a streaming source. The MV is the consumer — it fires on each poll batch and inserts into the target. Offsets are committed after successful insert.

### The "Too Many Parts" Problem

```
Error: Too many parts (307). Merges are processing significantly slower than inserts.
```

**Cause:** Each INSERT creates a part. If inserts are too frequent and small, parts accumulate faster than the background merger can consolidate them.

**ClickHouse limits:**
- Soft limit: ~300 parts per partition → slows inserts with artificial delay
- Hard limit: ~1000 parts per partition → refuses inserts

**Solutions (in order of preference):**
1. **Batch inserts**: insert larger batches (100K+ rows) less frequently
2. **Async INSERT**: let ClickHouse buffer and batch automatically
3. **Increase merge aggressiveness** (not recommended as primary fix):
   ```sql
   -- In merge_tree settings
   max_parts_in_total = 100000
   ```
4. **Reduce partition granularity**: if partitioned by day, switch to month for high-volume tables

**Rule of thumb:** Target 1 insert per second or less per table. If your pipeline pushes 100 inserts/second, use async insert or Kafka + MV pattern.

---

## Performance Tuning

### Key Settings

```sql
-- Per-query settings
SET max_threads = 16;                        -- parallel query threads (default: ncpu)
SET max_memory_usage = 10000000000;          -- 10GB per-query memory limit
SET max_bytes_before_external_sort = 5000000000;  -- sort spills to disk at 5GB
SET max_bytes_before_external_group_by = 5000000000;

-- Distributed queries
SET max_distributed_connections = 1000;
SET distributed_aggregation_memory_efficient = 1;  -- reduce memory on initiator

-- Approximate queries
SET max_rows_to_read = 1000000000;           -- limit scan scope
SET timeout_before_checking_execution_speed = 5;
```

### Projections

A projection is a **pre-computed sort order or aggregation** stored within the same part:

```sql
ALTER TABLE events ADD PROJECTION proj_by_country (
    SELECT country, event, count() as cnt
    GROUP BY country, event
    ORDER BY country
);

ALTER TABLE events MATERIALIZE PROJECTION proj_by_country;
```

When a query matches the projection's key/aggregation pattern, the optimizer automatically uses the projection instead of the main table. No query changes needed.

```sql
-- This query auto-uses the projection above:
SELECT country, event, count() FROM events GROUP BY country, event;
```

**vs Materialized Views:** Projections are tied to the parent table (same parts, replicated together). MVs are independent tables. Projections are better for alternate sort orders; MVs are better for cross-table aggregations.

### Dictionaries — Replacing Dimension JOINs

A dictionary is an **in-memory hash table** for fast key-value lookups. Replaces slow JOINs for dimension tables.

```sql
CREATE DICTIONARY user_country (
    user_id UInt64,
    country String DEFAULT 'unknown'
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(TABLE 'users' DB 'default'))
LAYOUT(HASHED())          -- hash map: O(1) lookup
LIFETIME(MIN 300 MAX 600) -- refresh every 5-10 minutes
;

-- Use in query: no JOIN needed
SELECT
    event,
    dictGet('user_country', 'country', user_id) as country,
    count()
FROM events
GROUP BY event, country;
```

**Layout options:**
| Layout | Best For |
|---|---|
| `FLAT()` | UInt64 keys 0–N, array lookup (fastest) |
| `HASHED()` | Arbitrary keys, hash map |
| `SPARSE_HASHED()` | Large dictionaries, memory-efficient |
| `RANGE_HASHED()` | Range keys (e.g., IP ranges) |
| `COMPLEX_KEY_HASHED()` | Composite keys |

Dictionaries are loaded into RAM on all nodes. Max practical size: ~tens of millions of rows.

### Sampling — Approximate Queries at Scale

```sql
-- Query ~1% of data (randomly sampled at granule level)
SELECT
    uniq(user_id) * 100 as approx_dau,
    count() * 100 as approx_events
FROM events SAMPLE 0.01
WHERE ts > now() - INTERVAL 1 DAY;
```

Sampling happens during part reading — only 1 in 100 granules are read. Requires `SAMPLE BY` clause in table definition:

```sql
CREATE TABLE events (
    ts DateTime,
    user_id UInt64,
    ...
)
ENGINE = MergeTree
ORDER BY (user_id, ts)
SAMPLE BY user_id;  -- sampling is by user_id hash
```

With `SAMPLE BY user_id`, the 1% sample contains ALL rows for the sampled users (not random rows), so per-user metrics are accurate within the sample.

### allow_experimental_parallel_reading_from_replicas

```sql
SET allow_experimental_parallel_reading_from_replicas = 1;
SET max_parallel_replicas = 3;
```

Fans out reads across replicas of the same shard — effectively multiplies read throughput by replica count. Useful for replica sets with read-heavy workloads.

---

## ClickHouse vs Alternatives

### vs Spark SQL

| Dimension | ClickHouse | Spark SQL |
|---|---|---|
| Latency | Sub-second to seconds | Seconds to minutes |
| Throughput | High for OLAP patterns | Very high for arbitrary transforms |
| Setup complexity | Medium (cluster) | High (cluster + scheduler) |
| SQL completeness | Good (some gaps) | Excellent (ANSI SQL + extensions) |
| Flexibility | Limited (append-heavy, no complex ETL) | High (arbitrary Python/Scala UDFs) |
| Cost | Storage + compute together | Compute elastic (separate from storage) |

**Rule:** ClickHouse for dashboards/queries that must return in <1s. Spark for complex ETL, ML pipelines, or queries that touch petabytes.

### vs Apache Druid

| Dimension | ClickHouse | Druid |
|---|---|---|
| Sub-second at petabyte scale | Harder (requires tuning) | Native (pre-aggregation mandatory) |
| SQL completeness | Better (arbitrary GROUP BY, subqueries) | Limited (no subqueries until recently) |
| Data model flexibility | High (any schema) | Opinionated (time-based segments, dimensions/metrics) |
| Operational complexity | Medium | Very high (Broker/Historical/Coordinator/Overlord/Middlemanager) |
| Exact counts | Yes | Approximate by default (sketches) |

**Rule:** Druid at petabyte scale with strict SLA (<100ms). ClickHouse when you need SQL flexibility and can accept 100ms–2s.

### vs BigQuery / Redshift

| Dimension | ClickHouse (self-hosted) | BigQuery/Redshift |
|---|---|---|
| Latency | Lower (no cold start) | Higher (slot allocation, network) |
| Cost | Lower for steady workloads | Lower for spiky workloads |
| Managed operations | DIY | Fully managed |
| Data egress | None | Can be expensive |
| Scale ceiling | Limited by cluster | Effectively unlimited |

**Rule:** ClickHouse when you have steady query load, control over infra, and latency sensitivity. Cloud DW when you want to avoid ops burden or have unpredictable scale needs.

### vs DuckDB

| Dimension | ClickHouse | DuckDB |
|---|---|---|
| Deployment | Distributed server | Embedded single-node |
| Max data size | Petabytes (distributed) | ~TB (RAM/disk of one machine) |
| Concurrency | High (server, multi-client) | Low (single process) |
| Integration | Network, HTTP API | In-process (Python, Go, Java...) |
| Setup | Cluster configuration | Zero-config (`import duckdb`) |

**Rule:** DuckDB for local analytics, data science notebooks, single-node ETL. ClickHouse for production multi-user analytics over large datasets.

---

## Concrete Examples

### 1. Clickstream Events Table with TTL and Skip Index

```sql
CREATE TABLE clickstream (
    ts           DateTime,
    user_id      UInt64,
    session_id   UInt64,
    page_url     String,
    country      LowCardinality(String),
    event_type   LowCardinality(String),
    duration_ms  UInt32,
    value        Float32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)          -- one partition per month
ORDER BY (user_id, ts)             -- sort by user for user-centric queries
PRIMARY KEY (user_id)              -- sparse index on user_id only
TTL ts + INTERVAL 90 DAY          -- delete rows after 90 days
SETTINGS index_granularity = 8192;

-- Skip index: fast country lookups without it being in the sort key
ALTER TABLE clickstream ADD INDEX idx_country country TYPE set(50) GRANULARITY 2;

-- Skip index: fast event_type lookups
ALTER TABLE clickstream ADD INDEX idx_event event_type TYPE set(20) GRANULARITY 2;

-- Bloom filter for session_id lookups
ALTER TABLE clickstream ADD INDEX idx_session session_id TYPE bloom_filter(0.01) GRANULARITY 1;
```

**Usage:**
```sql
-- Fast: user_id in sort key → binary search
SELECT * FROM clickstream WHERE user_id = 12345 AND ts > now() - INTERVAL 7 DAY;

-- Fast: country skip index prunes granules
SELECT count() FROM clickstream WHERE country = 'US' AND ts > '2024-01-01';

-- Fast: session skip index prunes granules
SELECT * FROM clickstream WHERE session_id = 9876543210;
```

### 2. Real-Time DAU with AggregatingMergeTree + Materialized View

```sql
-- Source table
CREATE TABLE events (
    ts       DateTime,
    user_id  UInt64,
    country  LowCardinality(String),
    event    LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (ts, user_id);

-- Aggregation target: stores HyperLogLog states
CREATE TABLE dau_by_country (
    date     Date,
    country  LowCardinality(String),
    dau      AggregateFunction(uniq, UInt64),   -- HyperLogLog state
    events   AggregateFunction(count)            -- count state
)
ENGINE = AggregatingMergeTree
ORDER BY (date, country);

-- Materialized view: fires on each INSERT to events
CREATE MATERIALIZED VIEW mv_dau_by_country
TO dau_by_country AS
SELECT
    toDate(ts)       AS date,
    country,
    uniqState(user_id) AS dau,
    countState()       AS events
FROM events
GROUP BY date, country;

-- Query: merge states for final result
SELECT
    date,
    country,
    uniqMerge(dau)   AS daily_active_users,
    countMerge(events) AS total_events
FROM dau_by_country
WHERE date >= today() - 30
GROUP BY date, country
ORDER BY date, daily_active_users DESC;
```

**Result:** Sub-second DAU queries regardless of raw event volume, because the heavy `uniq(user_id)` computation happens at insert time.

### 3. Dictionary for User Profile Lookups

```sql
-- Source table (can be updated separately)
CREATE TABLE users (
    user_id  UInt64,
    country  String,
    tier     Enum8('free'=1, 'pro'=2, 'enterprise'=3),
    signup_date Date
)
ENGINE = ReplacingMergeTree
ORDER BY user_id;

-- Dictionary: in-memory hash map, refreshed every 10 minutes
CREATE DICTIONARY user_profile (
    user_id     UInt64,
    country     String DEFAULT 'unknown',
    tier        String DEFAULT 'free',
    signup_date Date   DEFAULT '1970-01-01'
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(
    HOST 'localhost'
    PORT 9000
    DB 'default'
    TABLE 'users'
    USER 'default'
))
LAYOUT(HASHED())
LIFETIME(MIN 600 MAX 1200);

-- Query: no JOIN, O(1) lookup per row
SELECT
    event,
    dictGet('user_profile', 'country', user_id)             AS country,
    dictGet('user_profile', 'tier', user_id)                AS tier,
    count()                                                  AS events,
    uniq(user_id)                                           AS users
FROM events
WHERE ts > now() - INTERVAL 1 DAY
GROUP BY event, country, tier
ORDER BY events DESC;
```

**Why this beats a JOIN:** A JOIN on `users` reads the users table from disk on every query. The dictionary is already in RAM on every node — lookups are O(1) cache hits.

---

## Key Production Gotchas

### 1. Too Many Parts (Most Common Issue)
Inserting small batches too frequently causes part accumulation faster than merges can keep up. ClickHouse throttles and eventually rejects inserts.
**Fix:** Batch to 100K+ rows per insert, or use async insert / Buffer engine.

### 2. JOIN Order Optimization (Improved in 24.12 / 25.9+)
On older ClickHouse versions (pre-24.12), there is no join reorder optimizer — the right table is always loaded into memory as the hash table, so putting a 1B-row table on the right causes an OOM. Since **24.12**, the planner auto-places the smaller table on the right for simple joins, and since **25.9**, global join reordering (with column statistics) extends this to multi-table joins across INNER/LEFT/RIGHT/ANTI/SEMI/FULL types.
**Fix:** On 25.9+, verify actual join order via `EXPLAIN` rather than assuming manual placement is required. On older versions, or as a safety net when statistics are stale/missing, still put the smaller table on the right and use Dictionaries for dimension tables.

### 3. FINAL Is Slow
`SELECT ... FINAL` disables parallel part reading — it serializes reads to merge duplicates in memory.
**Fix:** Use `argMax(col, version)` pattern instead of FINAL for most use cases.

### 4. ReplacingMergeTree Is Eventually Consistent
Duplicates persist until background merge. Queries without FINAL may return multiple versions of the same key.
**Fix:** Accept eventual consistency OR always use FINAL OR use the `argMax` workaround.

### 5. Materialized Views Don't Backfill
Creating a MV on an existing table with data leaves historical data out of the target.
**Fix:** After creating the MV, manually backfill: `INSERT INTO target SELECT ... FROM source`.

### 6. Materialized Views Fail Silently
By default, if the MV query fails (type error, etc.), the source INSERT still succeeds but data is lost from the MV target.
**Fix:** Monitor `system.query_log` for MV errors. Set alerting on insert exceptions.

### 7. ZooKeeper / Keeper Is a Single Point of Failure
If ZooKeeper is unavailable, replicated tables can't coordinate. Reads still work (from local data), but INSERTs to replicated tables fail.
**Fix:** Run a 3-node Keeper quorum. Monitor Keeper health separately from ClickHouse.

### 8. Mutations Are Slow and Non-Transactional
`ALTER TABLE ... UPDATE/DELETE` rewrites entire parts asynchronously. Large mutations can run for hours and consume significant I/O.
**Fix:** Use ReplacingMergeTree or CollapsingMergeTree for mutable data. Avoid mutations in hot paths.

### 9. LowCardinality Has a Limit
`LowCardinality(String)` is stored as a dictionary with UInt8/UInt16 keys. If distinct values exceed ~65K, it degrades to String automatically.
**Fix:** Only use `LowCardinality` for truly low-cardinality columns (country codes, status enums, event types).

### 10. Distributed Table Doesn't Guarantee Write Consistency
When inserting into a Distributed table, if a shard is down, rows for that shard are buffered locally and replayed later. This can cause duplicates if the buffer is also lost.
**Fix:** Insert directly to local replicated tables on each shard when possible. For Distributed inserts, enable `insert_distributed_sync = 1` (slower but safer).

### 11. Partition Pruning Requires Partition Function Match
```sql
-- Table partitioned by: PARTITION BY toYYYYMM(ts)
-- This query DOES prune partitions:
SELECT * FROM events WHERE toYYYYMM(ts) = 202401;
-- This query does NOT prune partitions (different function):
SELECT * FROM events WHERE ts BETWEEN '2024-01-01' AND '2024-01-31';
```
ClickHouse is getting smarter here but historically required exact function match.
**Fix:** Use the exact same function in WHERE as in PARTITION BY, or use `ts >= '2024-01-01' AND ts < '2024-02-01'` for timestamp range queries.

### 12. GROUP BY on Distributed Tables Requires Two-Phase Aggregation
If you `GROUP BY` a non-sharding-key column on a Distributed table, the initiator receives all partial groups from all shards and must merge them. For very high cardinality GROUP BY, this can OOM the initiator.
**Fix:** Shard by the GROUP BY key, or use `max_memory_usage` + `distributed_aggregation_memory_efficient = 1`.

### 13. ORDER BY Without LIMIT Is Dangerous
`SELECT * FROM events ORDER BY ts` on a billion-row table will sort everything in memory or spill to disk. There's no "fetch first N rows" short-circuit without `LIMIT`.
**Fix:** Always pair `ORDER BY` with `LIMIT`. Use `max_rows_to_sort` as a safety net.

---

## Quick Reference

### System Tables for Debugging

```sql
-- Active parts count per table (watch for "too many parts")
SELECT table, count() as parts FROM system.parts
WHERE active GROUP BY table ORDER BY parts DESC;

-- Running queries
SELECT query_id, elapsed, query FROM system.processes ORDER BY elapsed DESC;

-- Recent slow queries
SELECT query, query_duration_ms, read_rows, read_bytes
FROM system.query_log
WHERE type = 'QueryFinish' AND query_duration_ms > 5000
ORDER BY query_start_time DESC LIMIT 20;

-- Replication lag
SELECT database, table, replica_name, absolute_delay
FROM system.replicas
WHERE absolute_delay > 0
ORDER BY absolute_delay DESC;

-- Part merge activity
SELECT * FROM system.merges ORDER BY elapsed DESC;

-- Dictionary memory usage
SELECT name, status, bytes_allocated, element_count
FROM system.dictionaries;
```

### Useful EXPLAIN

```sql
-- Show query plan
EXPLAIN SELECT count() FROM events WHERE user_id = 1;

-- Show pipeline
EXPLAIN PIPELINE SELECT count() FROM events WHERE user_id = 1;

-- Show indexes used (which granules pruned)
EXPLAIN indexes = 1 SELECT count() FROM events WHERE user_id = 1;
```

### Compression Ratio Check

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
GROUP BY table
ORDER BY sum(data_compressed_bytes) DESC;
```
