# Z-Ordering (Morton Curve)

## 30-Second Intuition

Z-ordering reorganizes rows in Parquet files so that rows with similar values across multiple columns end up in the same file. This makes Parquet's per-file min/max statistics useful for multi-column predicates — the engine can skip entire files instead of scanning them. Without Z-ordering, min/max stats for non-partition columns are nearly useless because rows are laid out in insertion order, spreading every value across every file.

---

## The Data Skipping Problem

### Why Row Groups Are Useless Without Layout Optimization

Parquet stores data in **row groups** (~128MB each). Each row group has column statistics:
```
Row group 0: user_id [min=1, max=9999999], event_type [min="click", max="view"]
Row group 1: user_id [min=1, max=9999999], event_type [min="click", max="view"]
```

If your table has 1000 files with natural insertion order (rows arrive from all users, all event types), the min/max range for every file is nearly the full domain. **Zero files can be skipped.**

Query: `WHERE user_id = 42 AND event_type = 'purchase'`
- File 1: user_id range [1, 9999999] → could contain 42 ✓ (read it)
- File 2: user_id range [1, 9999999] → could contain 42 ✓ (read it)
- ...all 1000 files read

With Z-ordering, rows with similar (user_id, event_type) combinations cluster together:
- File 1: user_id [1, 1000], event_type ["click", "click"] → no 42 skip
- File 5: user_id [40, 50], event_type ["purchase", "purchase"] → could have it ✓
- ...most files skipped

---

## What Z-Ordering (Morton Curve) Actually Is

### The Space-Filling Curve Intuition

Imagine a 2D grid. You want to visit every cell in an order that keeps nearby cells close together in the sequence. Naive approaches fail:

```
Row-major order:          Z-order (Morton):
1  2  3  4               1  2  5  6
5  6  7  8               3  4  7  8
9  10 11 12              9  10 13 14
13 14 15 16              11 12 15 16
```

Z-order visits cells in a Z-shaped pattern recursively. Cells that are neighbors in 2D space are **close together in the 1D Z-order sequence** — this is the key property.

```
Z-pattern on a 4x4 grid:
 1  2 | 5  6
 3  4 | 7  8
------+------
 9 10 |13 14
11 12 |15 16

The Z-shape: top-left quadrant → top-right → bottom-left → bottom-right, recursively.
```

When you sort data by Z-value and then write to files, rows that are spatially nearby (similar x AND y values) end up in the same file.

---

## How Z-Values Are Computed: Bit Interleaving

### Simplified 2D Example

Given two 4-bit values (columns A and B):
```
A = 5  = 0101 (binary)
B = 3  = 0011 (binary)
```

Interleave the bits (B's bit, A's bit, alternating):
```
A bits: 0  1  0  1
B bits: 0  0  1  1
        ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓
Z bits: B3 A3 B2 A2 B1 A1 B0 A0
      = 0  0  0  1  1  0  1  1
      = 00011011 binary
      = 27 decimal
```

More concretely:
```
(A=0, B=0) → Z=0
(A=1, B=0) → Z=1
(A=0, B=1) → Z=2
(A=1, B=1) → Z=3
(A=2, B=0) → Z=4   (bit pattern: 0100)
(A=3, B=0) → Z=5
(A=2, B=1) → Z=6
(A=3, B=1) → Z=7
...
```

This is the Morton code. Points that are close in 2D space (nearby A and B values) get Z-values that are also numerically close.

### For String/Categorical Columns

Strings are typically converted to integer ranks or truncated to fixed-width byte strings before Z-value computation. The transformation must preserve sort order to maintain the locality property.

### For N Dimensions

Bit interleaving extends to N columns: for 3 columns A, B, C, interleave A_bits, B_bits, C_bits in sequence. Z-values grow in bit width: k columns × n bits per value = k×n bit Z-value.

---

## Why Co-Locality Matters for Query Skipping

### The Query Pruning Chain

```
Query: WHERE col_a BETWEEN 40 AND 60 AND col_b = 'purchase'
  │
  ▼
File 1: col_a [min=1, max=10]      → pruned ✓ (40 not in [1,10])
File 2: col_a [min=35, max=65]     → maybe...
          col_b [min='purchase',   → both columns match range
                 max='purchase']   → READ
File 3: col_a [min=100, max=200]   → pruned ✓
```

**The insight**: Z-ordering makes it likely that `col_a` and `col_b` ranges are **tight** within each file. A file covering `col_a [35,65]` will also have a tight `col_b` range because rows were sorted to cluster by both columns simultaneously.

Without Z-order, a file covering `col_a [35,65]` was written by insertion time — it contains all possible `col_b` values, so the `col_b` filter gives no pruning power.

---

## Parquet Min/Max Statistics: The Mechanism

Parquet stores statistics per **row group** (typically ~128MB) and per **column chunk**:

```
Row Group 0:
  column "user_id": {min: 42, max: 42, null_count: 0, distinct_count: 1}
  column "event_type": {min: "purchase", max: "purchase", null_count: 0}
  column "ts": {min: 2024-01-15T10:00, max: 2024-01-15T11:00}
```

Delta/Iceberg read these statistics at planning time (they're in manifest/log entries, so you don't even need to open the Parquet files) and prune files entirely before the tasks are scheduled.

**Two-level pruning**:
1. **File-level**: skip entire Parquet files using per-file stats (from manifest/log).
2. **Row-group-level**: skip row groups within a file using Parquet internal stats.

Z-ordering improves **file-level** pruning. Within a file, row groups are naturally tighter too.

---

## Z-Order vs Range Sort: When Each Wins

### Range Sort (single column)

```sql
OPTIMIZE events ZORDER BY (ts);
-- vs
-- sort by single column ts
```

If you only ever filter on `ts`, a **simple sort** is better than Z-order. Range sort produces:
- File 1: ts [Jan 1 – Jan 5] → perfect range coverage
- File 2: ts [Jan 6 – Jan 10]

A single-column Z-order degenerates to a range sort anyway (no interleaving needed for 1 column).

**Range sort wins when**: queries have a dominant single-column filter (e.g., always filter by date, date is already the partition column, and you want sub-partition skipping).

### Z-Order Wins When

Queries filter on **two or more correlated columns simultaneously**:

```sql
WHERE user_id = 42 AND event_type = 'purchase'
WHERE region = 'us-west' AND product_id BETWEEN 100 AND 200
```

Z-order keeps both dimensions packed, so both column ranges are tight per file.

### Z-Order Loses When

1. **Low cardinality columns**: `event_type` has 5 possible values. Z-ordering on it just colocates rows by those 5 values — same as a simple sort but with worse file boundaries. Use partitioning instead.

2. **One column dominates all queries**: Just range-sort on that column.

3. **More than ~4 Z-order columns**: Each additional column dilutes the locality of others. With 8 columns, the Z-value is 64+ bits; nearby values in any single dimension are no longer close in Z-order.

4. **Monotonic timestamps as the sole filter**: Partition by days/hours and range-sort by timestamp within partitions. Z-ordering adds no value.

---

## The Cost: Full Data Rewrite

Z-ordering via OPTIMIZE is a **full file rewrite** for the affected partitions:

```sql
OPTIMIZE events WHERE date = '2024-01-15' ZORDER BY (user_id, event_type);
```

- Reads all files in the `date=2024-01-15` partition.
- Computes Z-values for (user_id, event_type) per row.
- Sorts all rows by Z-value.
- Writes new compacted files.
- Old files marked as removed (physically deleted after Vacuum).

**This is not incremental**. If you append 1M new rows to `date=2024-01-15` tomorrow, the newly written files are in insertion order. The ordering degraded. You must re-run OPTIMIZE to restore Z-ordering.

**Practical implication**: For active partitions receiving daily appends, Z-ordering must be re-run daily. For historical partitions (no more writes), Z-order persists.

This is the core motivation for **Liquid Clustering** (see liquid-clustering.md). As of 2026, Databricks recommends Liquid Clustering over Z-order/partitioning for all new tables — including streaming tables, materialized views, and (as of Databricks Runtime 18.0+) managed Apache Iceberg v3 tables in Unity Catalog. Z-order is not deprecated and remains the standard approach for open-source Delta/Iceberg without Liquid Clustering, or for existing tables not yet migrated, but it is no longer the recommended default for new Databricks tables.

---

## Bloom Filters as a Complementary Technique

Bloom filters are **probabilistic membership indexes** stored in Parquet files.

```sql
-- Spark: create Parquet files with bloom filters
df.write.option("parquet.bloom.filter.enabled#user_id", "true") \
        .option("parquet.bloom.filter.expected.ndv#user_id", "1000000") \
        .parquet("s3://...")
```

**How it works**: For each row group, a bloom filter is stored per configured column. For a query `WHERE user_id = 42`:
1. Check bloom filter: "Is 42 possibly in this row group?"
2. If NO (bloom filter returns false) → skip row group. Zero false negatives guaranteed.
3. If YES (bloom filter returns true) → read row group (may still be a false positive).

**Bloom filters vs Z-order**:
- Z-order: range queries (`BETWEEN`, `>=`, `<=`). Works on min/max stats.
- Bloom filters: point lookups (`= value`). No false negatives; some false positives.
- They **complement each other**: Z-order for range skipping, bloom filters for exact match skipping.
- Bloom filters add space overhead per column (~1-2% of column data for 1% false positive rate).

---

## Concrete Example: Events Table

**Table**: `events(user_id BIGINT, event_type STRING, ts TIMESTAMP, payload STRING)`
- 1 billion rows, 1000 files × 1M rows each
- Inserted in timestamp order

### Without Z-Order

Query: `WHERE user_id = 42 AND event_type = 'purchase'`

```
File stats (insertion order):
  File 1:   user_id [1, 9999999], event_type ['add_to_cart', 'view']
  File 2:   user_id [1, 9999999], event_type ['click', 'purchase']
  ...
  File 1000: user_id [1, 9999999], event_type ['add_to_cart', 'view']
```

Files skipped: ~0 (because user_id and event_type span full domain in every file)
Files read: 1000 files = 128GB scanned

### With Z-Order on (user_id, event_type)

After `OPTIMIZE events ZORDER BY (user_id, event_type)`:

Z-values group rows: small user_id + early event_type → small Z → early files.

```
File 1:   user_id [1, 1000],       event_type ['add_to_cart', 'click']    → skip (user_id 42 in range, but event_type no purchase)
File 2:   user_id [1, 1000],       event_type ['purchase', 'purchase']    → READ (42 in range, purchase matches)
File 3:   user_id [1001, 2000],    event_type ['add_to_cart', 'click']    → skip
...
File 997: user_id [9990000, ...],  event_type ['view', 'view']            → skip
```

Files skipped: ~990 of 1000
Files read: ~10 files = ~1.3GB scanned ← 100x improvement

### Partition Before Z-Order

Best practice: partition by a high-level dimension first, then Z-order within partitions.

```sql
-- Partition by date (eliminates cross-day scans)
-- Z-order by (user_id, event_type) within each day
-- Each day has ~10 files; Z-order within that

CREATE TABLE events (...) PARTITIONED BY (date DATE);
OPTIMIZE events WHERE date = '2024-01-15' ZORDER BY (user_id, event_type);
```

Query `WHERE date = '2024-01-15' AND user_id = 42 AND event_type = 'purchase'`:
- Partition pruning: look at only Jan 15 partition (10 files)
- Z-order pruning: skip 9 of 10 files
- Scan: 1 file × 128MB = 128MB

---

## When NOT to Use Z-Order

| Situation | Why Z-order is Wrong | Better Alternative |
|---|---|---|
| Only one filter column | Z-order = range sort for 1D | Simple sort + optional partition |
| Low-cardinality column (`status ∈ {active, inactive}`) | 2 values can't create locality | Partition by status |
| Monotonically increasing timestamp as sole filter | Already ordered naturally | Partition by days(ts), range sort by ts |
| > 4 Z-order columns | Too many dims, locality degrades | Pick top 2-3 columns by query frequency |
| Append-only table with no historical queries | No point organizing history | Just ensure file size is reasonable |
| Very small tables (< 10 files) | Not enough files to skip | No need, overhead > benefit |
| Write-heavy, rare reads | Rewrite cost exceeds read savings | Accept insertion order until data is "cold" |

---

## Key Gotchas

- **Z-order is not maintained incrementally**: New data arrives in insertion order, degrading clustering. Must re-OPTIMIZE periodically (see liquid-clustering.md for the solution).
- **Column order matters weakly**: The first column in the Z-order tuple has slightly better locality. Put the most-queried column first.
- **Z-order ≠ partition**: Don't Z-order by a column you should be partitioning by. If you always filter by `region` and have 5 regions, partition by `region`; Z-ordering by it wastes effort.
- **Stats must be collected**: File-level stats are only useful if they're written. Parquet always writes them. But for Iceberg: stats in manifests are only written if the write engine supports it. Verify your engine version.
- **Row group size affects internal pruning**: Large row groups (512MB) mean coarser internal statistics. Smaller row groups (32MB) give finer pruning but more metadata overhead.
- **ZORDER with data skew**: If 80% of queries target user_id < 1000, Z-order will pack those rows into the first few files. Those few files become hot spots. Consider bucketing instead for load distribution.
