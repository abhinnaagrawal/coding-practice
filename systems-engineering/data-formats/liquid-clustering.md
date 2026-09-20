# Liquid Clustering

## 30-Second Intuition

Liquid Clustering is Delta Lake's answer to "Z-order breaks the moment you append new data." Instead of a one-time full sort, Liquid Clustering is an **incremental, online re-clustering** scheme: new data files get tagged as "unclustered," and OPTIMIZE progressively merges and re-sorts them into the existing clustered layout. The table gradually self-organizes over time — hence "liquid" — without ever needing a big-bang full rewrite.

---

## Why Z-Order Breaks Down

### The Core Problem: It's a One-Time Sort

```
Day 1: 100 files, Z-ordered by (user_id, event_type)
        → perfect clustering, 90% file skip rate

Day 2: +20 new files appended (insertion order)
        → new files have full-domain stats, ruin pruning for new data

Day 3: +20 more files
        → degradation compounds

After 1 week: 240 files, only the original 100 are clustered
              effective skip rate: ~40% (new data drags it down)
```

To restore clustering: `OPTIMIZE events ZORDER BY (user_id, event_type)` — but this **rewrites all 240 files**. For a 10TB table this means reading and writing 10TB. Expensive, slow, wastes I/O budget.

### The Operational Pain

1. You must schedule OPTIMIZE to run regularly.
2. OPTIMIZE during business hours competes with query I/O.
3. Each run rewrites the entire table even if only 5% of data changed.
4. No fine-grained tracking of "which files are already well-clustered."

---

## Liquid Clustering Concept

### The Key Idea

Track clustering state **per file**. New files are born "unclustered." OPTIMIZE only works on unclustered files, merging them with each other (and sometimes with nearby clustered files) to produce clustered output. Already-clustered files are left alone.

```
Before OPTIMIZE:
  [clustered] file_001 (user_id 1-500)
  [clustered] file_002 (user_id 501-1000)
  [unclustered] file_100 (user_id 1-9999, insertion order)
  [unclustered] file_101 (user_id 1-9999, insertion order)
  [unclustered] file_102 (user_id 1-9999, insertion order)

After incremental OPTIMIZE:
  [clustered] file_001 (user_id 1-500)     ← untouched
  [clustered] file_002 (user_id 501-1000)  ← untouched
  [clustered] file_200 (user_id 1-100)     ← new, from merging 100+101+102
  [clustered] file_201 (user_id 101-200)   ← new
  ...
```

**Only unclustered files are read and rewritten.** Cost is proportional to new data volume, not total table size.

---

## How It Works Internally

### Hilbert Curve Instead of Z-Order

Liquid Clustering uses the **Hilbert space-filling curve** instead of the Z-order (Morton) curve.

**Why Hilbert is better**:
```
Z-order (Morton):        Hilbert:
1  2  5  6              1  2 15 16
3  4  7  8              4  3 14 13
9  10 13 14             5  8  9 12
11 12 15 16             6  7 10 11
```

The Hilbert curve has **better locality preservation**: points near each other in 2D space are more consistently near each other in the 1D Hilbert sequence. The Z-order has "jumps" (e.g., going from quadrant to quadrant), causing points that are geometrically close to be far apart in the 1D sequence. Hilbert eliminates most of these jumps.

**Practical impact**: slightly better file pruning rates for multi-column range queries.

### Clustering Keys and Micro-Partition Ranges

```sql
-- Define clustering keys at table creation
CREATE TABLE events (
  user_id   BIGINT,
  event_type STRING,
  ts        TIMESTAMP,
  payload   STRING
) CLUSTER BY (user_id, event_type);
```

Clustering keys define the **Hilbert space**. Each unique (user_id, event_type) pair maps to a point in this space. Rows are sorted by their Hilbert value, producing files where each file covers a tight contiguous region of the Hilbert curve.

**Micro-partition ranges**: each clustered file covers a small, tight range of Hilbert values. The file's min/max stats for `user_id` and `event_type` are therefore tight, enabling good pruning.

### Delta's Clustering State Tracking

Delta table properties track clustering metadata:
```json
{
  "delta.feature.clustering": "supported",
  "delta.clusteringColumns": "[{\"physicalName\":\"user_id\"},{\"physicalName\":\"event_type\"}]"
}
```

Each `add` action in the Delta log carries a **clustering score** (internal metric). Files with low clustering scores (recently written, insertion order) are prioritized by OPTIMIZE.

Delta tracks a **"domino"** structure internally: a set of file ranges that form the current clustered layout. Incoming files are compared against this structure to determine if they fit or need to be reorganized.

---

## The "Liquid" Metaphor

Think of clustered data as solid ice and new unclustered data as water being poured in.

```
Day 1: [ICE: clustered files spanning user_id 1–10M]
Day 2: [ICE: clustered] + [WATER: 20 new unclustered files]
Day 3: [ICE: clustered] + [MORE WATER: 40 unclustered files]

OPTIMIZE (incremental):
  Takes the WATER, freezes it into ICE, merges with adjacent ICE blocks
  → [ICE: all clustered]

The table "flows" into a good layout organically, without draining the whole lake.
```

The key property: at any point, some of the table is liquid (unclustered) and some is ice (clustered). Queries work on both; they just get less skipping on the liquid parts. Over time, OPTIMIZE "freezes" the liquid incrementally.

---

## CLUSTER BY vs ZORDER BY

| Aspect | `CLUSTER BY` (Liquid) | `ZORDER BY` (Classic) |
|---|---|---|
| **Declaration** | Table property (CREATE/ALTER) | Per-OPTIMIZE command argument |
| **Persistence** | Permanent — clustering keys stored in table metadata | One-time — must specify every OPTIMIZE run |
| **Scope of OPTIMIZE** | Only unclustered files | All files in affected partitions |
| **Algorithm** | Hilbert curve | Z-order (Morton) curve |
| **Incremental** | Yes | No |
| **New data handling** | New files auto-marked unclustered, next OPTIMIZE picks them up | New data unclustered until next full OPTIMIZE |
| **Compatibility** | Delta Lake OSS 3.1+ (preview), GA in 3.2+ | Open Delta Lake |
| **With partitioning** | Mutually exclusive on same table | Compatible |

### Syntax

```sql
-- Liquid Clustering: declare at table creation
CREATE TABLE events (user_id BIGINT, event_type STRING, ts TIMESTAMP)
CLUSTER BY (user_id, event_type);

-- Or add clustering to existing table
ALTER TABLE events CLUSTER BY (user_id, event_type);

-- OPTIMIZE: just run it, no column specification needed
OPTIMIZE events;
-- Delta knows the clustering keys from table properties
-- Delta knows which files are unclustered
-- Only those files are processed

-- Classic Z-order: must specify columns every time
OPTIMIZE events ZORDER BY (user_id, event_type);
-- This ALWAYS rewrites all files in the affected partition
```

---

## Incremental OPTIMIZE: How Delta Tracks Files

### The Clustering Score

When a file is written (INSERT, MERGE, etc.), Delta assigns it a **clustering score** based on how well its data aligns with the declared clustering keys. A freshly written file from streaming ingestion has a low score (data is insertion-ordered, not clustered).

During OPTIMIZE:
1. Read all `add` actions with clustering scores below a threshold.
2. Group low-score files by their Hilbert value ranges (so co-located files are processed together).
3. Merge groups: read files, sort by Hilbert value, write new clustered files.
4. Commit: remove old files, add new clustered files with high clustering scores.

Files already above the threshold: untouched.

### Practical Run Cadence

Because OPTIMIZE is incremental, you can run it **continuously or frequently** without worrying about rewriting clustered data:

```python
# Run after every batch ingest
df.write.format("delta").mode("append").saveAsTable("events")
spark.sql("OPTIMIZE events")  # fast: only processes new files
```

For streaming:
```python
# Or schedule OPTIMIZE every 30 minutes
# Each run takes ~seconds for typical incremental volumes
```

Databricks supports **Auto Optimize** (`delta.autoOptimize.autoCompact` + `delta.autoOptimize.optimizeWrite`) which triggers incremental OPTIMIZE automatically after writes.

---

## Choosing Clustering Keys

### Cardinality Guidelines

| Column Cardinality | Suitable for Clustering? | Notes |
|---|---|---|
| Very low (< 10 distinct values) | No | Partition instead |
| Low-medium (10–1000) | Maybe | OK if frequently filtered, but consider partition |
| Medium-high (1K–100M) | Yes | Ideal range for clustering |
| Very high (> 100M, e.g., UUID) | With caution | Works, but each file covers a tiny range |

### Query Pattern Analysis

1. Identify the 3-5 most common WHERE clause columns across your workload.
2. Count how often each column appears in filters.
3. Pick top 2-3 as clustering keys.

```sql
-- Example: analyze query history (Databricks)
SELECT filter_columns, count(*) as query_count
FROM query_history
WHERE table_name = 'events'
GROUP BY filter_columns
ORDER BY 2 DESC;
```

### Key Selection Rules

- **Choose columns that appear together in WHERE clauses**: clustering on (user_id, event_type) helps queries filtering both; it also helps queries filtering just user_id (high-order dimension benefit).
- **Avoid columns that are always equality-filtered with tiny result sets**: bloom filters may be better.
- **Avoid timestamp as a clustering key if you already partition by time**: within a partition, ts is already sorted naturally (or close to it). Use the other column.
- **Maximum 4 clustering columns recommended**: beyond 4, Hilbert value space becomes so large that locality degrades.

---

## Comparison Table

| Feature | Hive Partitioning | Z-Order (Classic) | Liquid Clustering |
|---|---|---|---|
| **Mechanism** | Directory-based file grouping | One-time full sort by Morton value | Incremental sort by Hilbert value |
| **Query pruning** | Exact partition match | Min/max stats skipping | Min/max stats skipping |
| **Multi-column benefit** | No (one dimension per partition level) | Yes | Yes |
| **Incremental writes** | Works naturally | Degrades clustering | Maintained incrementally |
| **Rewrite cost** | None (just directory routing) | Full partition rewrite each time | Only unclustered files |
| **Column cardinality** | Low (< 1000 distinct values recommended) | Any | Medium-high |
| **Partition evolution** | Hard (Hive), easy (Iceberg) | N/A | N/A |
| **Data skipping granularity** | Partition level | File level | File level |
| **Operational complexity** | Low (auto) | High (schedule OPTIMIZE) | Low (set-and-forget + periodic OPTIMIZE) |
| **Open source** | Yes | Yes (Delta open source) | Databricks-specific (2024) |

### When to Use Each

**Hive-style partitioning**: always filter by this column, low cardinality (date, region, status), column values naturally distribute data evenly.

**Z-order**: you're on open-source Delta (no Liquid Clustering), historical data that won't change, batch workload that can afford periodic full OPTIMIZE.

**Liquid Clustering**: Databricks environment, active tables receiving continuous writes, multi-column query patterns, want minimal operational overhead.

---

## Limitations

### Open Source Status (updated, current as of Delta Lake 3.3)

Liquid Clustering **is open source** — this reverses the 2024 assumption that it was Databricks-only. Timeline:

- **Delta Lake 3.1**: liquid clustering shipped in OSS as a *preview*, gated behind `spark.databricks.delta.clusteredTable.enableClusteringTablePreview`. Preview build lacked ZCube-based incremental clustering, `ALTER TABLE ... CLUSTER BY` (to change clustering columns), and `DESCRIBE DETAIL` clustering introspection.
- **Delta Lake 3.2**: preview flag removed, all of the above features supported. This is the first fully-GA OSS release.
- **Delta Lake 3.3**: adds support for enabling clustering on an *existing unpartitioned* table (previously only at table-creation time or via full rewrite).

So: plain `delta-spark` (no Databricks Runtime) on Delta Lake ≥3.2 gets full `CLUSTER BY` support, `OPTIMIZE`-driven incremental clustering, and changeable clustering columns. Source: [docs.delta.io/delta-clustering](https://docs.delta.io/delta-clustering/).

**Engine support caveat**: liquid clustering is a Delta *writer feature* (table uses writer version 7 / `clusteringtable` feature). Any writer must understand the feature to write compliant clustered layout — Trino/Presto and Flink Delta connectors that predate this feature will write unclustered files (data is still valid, just untagged); the next Spark-driven `OPTIMIZE` picks them up. Always check your engine's Delta connector changelog before assuming native support.

### Column Limits (from official docs)

- Clustering columns must have **statistics collected** — by default only the **first 32 columns** of a table have stats collected (`delta.dataSkippingNumIndexedCols`).
- **Maximum 4 clustering columns** per table.

### Other Limitations

- **Mutually exclusive with Hive-style partitioning and `ZORDER` on the same table.** The official docs are explicit: *"Clustering is not compatible with partitioning or ZORDER."* This corrects an earlier common assumption (and a claim in some third-party blog posts) that you can do `PARTITIONED BY (date) CLUSTER BY (user_id)` on one table — you cannot. Pick one. In practice: coarse time-based partitioning gives way to **using the date/time column itself as one of the up-to-4 clustering keys** instead of a partition column, since clustering subsumes what partitioning did for pruning purposes while staying incremental.
- **Table protocol upgrade is one-way**: enabling clustering sets Delta writer version 7 / reader version 1 with the `clustering` and `domainMetadata` table features. You cannot downgrade the protocol afterward.
- **Changing clustering keys does NOT auto-rewrite anything — and plain `OPTIMIZE` won't fix it either**: `ALTER TABLE events CLUSTER BY (new_col)` (supported OSS-wide since 3.2) only updates table metadata. Existing files keep their old physical layout. New writes after the ALTER get clustered by the *new* keys (eager clustering, if under ~512GB/write), but old files are left alone — they are not tagged "unclustered" just because the key changed. A subsequent plain `OPTIMIZE table_name` only picks up genuinely new/unclustered files; it does **not** detect "these files were clustered under a stale key set" and will leave them exactly as they are. The table then has old data clustered on key A and new data clustered on key B, indefinitely, unless you act. To force reclustering of pre-existing data under the new keys, you must explicitly run `OPTIMIZE table_name FULL` (Delta Lake 3.3+) — this can take hours on large tables since it's effectively a full rewrite. Plan clustering keys carefully upfront to avoid this.
- **Eager clustering threshold**: newly ingested data is auto-clustered on write only under ~512GB per write; beyond that, rely on scheduled `OPTIMIZE`.
- **Vacuum still required**: clustering creates new files and marks old ones removed. Physical deletion still needs Vacuum.
- **Not compatible with HiveSerde tables**: requires Delta format.
- **Streaming microbatch latency**: if you're writing microbatches every 1 minute, each batch creates unclustered files. OPTIMIZE must run frequently. For very-high-frequency streaming, the per-OPTIMIZE overhead can be non-trivial.

---

## Key Gotchas

- **`CLUSTER BY` + partitioning is NOT allowed on the same table** — they're mutually exclusive per the official spec. If you need a time-based dimension, add the timestamp column as one of your (max 4) clustering keys instead of partitioning by it.
- **Liquid clustering is open source since Delta Lake 3.2 (GA)** — don't assume Databricks-only. Verify your Spark's `delta-spark` version and your Trino/Flink connector's feature support before relying on it in a non-Databricks stack.
- **Bloom filter indexes are not part of the open Delta Lake spec.** [delta-io/delta#1347](https://github.com/delta-io/delta/issues/1347) is still an *open* enhancement request (as of this writing) — no bloom filter table feature exists in OSS Delta. Databricks *did* ship a proprietary Bloom filter index in DBR, but Databricks has since **deprecated** it in favor of predictive I/O and liquid clustering — so it's fading even in the platform that had it. See `apache-iceberg.md` for a contrast: Iceberg's Puffin/NDV stats and engine-side bloom filters (e.g., Parquet bloom filters written by Spark) are a separate, format-level mechanism unrelated to this deprecated Databricks feature.
- **First OPTIMIZE after enabling clustering on an existing table is expensive**: all existing files are unclustered → essentially a full rewrite. Schedule it during off-peak hours.
- **OPTIMIZE is idempotent but not free**: running OPTIMIZE on a fully-clustered table is fast (nothing to do), but there's still overhead to check clustering scores. Fine to run frequently.
- **Auto Optimize does not replace scheduled OPTIMIZE**: Databricks Auto Optimize runs a light compaction pass. Full clustering OPTIMIZE should still run periodically for large tables.
- **Liquid Clustering does not improve write performance**: writes are still insertion-order. The improvement is entirely on reads (via file pruning at query time).
- **Cannot use Liquid Clustering to replace partitioning for very large tables**: a 100TB table with 1M files needs partitioning to limit the number of files OPTIMIZE must evaluate, even if Liquid Clustering only processes unclustered ones.
