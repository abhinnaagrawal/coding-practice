# Delta Lake

## 30-Second Intuition

Delta Lake is a **transaction log layer** on top of Parquet files. Every change to a table is a JSON entry appended to `_delta_log/`. To know what a table contains right now, replay those log entries in order. Because appending to a log is atomic (object stores guarantee this), you get ACID for free. Delta is Databricks-native but open-source; it's the de facto standard for Spark/Databricks workloads, and by 2026 its multi-engine story has matured significantly via **Delta Kernel** (a shared core library so non-Spark engines like Trino, Flink, DuckDB, and Polars don't need bespoke connectors) and **UniForm** (see below).

**Version note**: Delta Lake crossed a major version boundary with **4.0** (mid-2025, built on Apache Spark 4.0), followed by 4.1, 4.2, and **4.3** (June 2026) — each release covers Delta Spark, Kernel, UniForm, Sharing, and Flink together. Delta Kernel's Java read path has been production-ready since 3.2; write support (including deletion-vector-aware writes) is newer and still maturing. Delta Standalone is deprecated in favor of Kernel.

---

## The Delta Log: `_delta_log/`

Every Delta table is a directory with two types of content:

```
s3://warehouse/db/events/
├── _delta_log/
│   ├── 00000000000000000000.json    ← commit 0: CREATE TABLE
│   ├── 00000000000000000001.json    ← commit 1: INSERT
│   ├── 00000000000000000002.json    ← commit 2: UPDATE
│   ├── ...
│   ├── 00000000000000000009.json    ← commit 9
│   ├── 00000000000000000010.checkpoint.parquet  ← checkpoint at 10
│   ├── 00000000000000000010.json
│   ├── ...
│   └── _last_checkpoint             ← JSON: { "version": 10 }
└── part-00000-abc.snappy.parquet
    part-00001-def.snappy.parquet
    ...
```

### JSON Commit File Structure

Each commit file contains newline-delimited JSON **actions**:

```json
{"commitInfo": {"timestamp": 1715000000000, "operation": "WRITE", "operationParameters": {"mode": "Append"}, "isBlindAppend": true}}
{"protocol": {"minReaderVersion": 1, "minWriterVersion": 2}}
{"metaData": {"id": "abc-123", "format": {"provider": "parquet"}, "schemaString": "{...}", "partitionColumns": ["date"]}}
{"add": {"path": "date=2024-01-15/part-00000-abc.parquet", "partitionValues": {"date": "2024-01-15"}, "size": 134217728, "modificationTime": 1715000000000, "dataChange": true, "stats": "{\"numRecords\":150000,\"minValues\":{\"ts\":\"2024-01-15T00:00:00\"},\"maxValues\":{\"ts\":\"2024-01-15T23:59:59\"},\"nullCount\":{\"ts\":0}}"}}
{"add": {"path": "date=2024-01-15/part-00001-def.parquet", ...}}
```

**Action types**:
- `commitInfo`: metadata about the operation (auditing)
- `protocol`: reader/writer version requirements
- `metaData`: table schema, partition columns (written when changed)
- `add`: a data file added to the table
- `remove`: a data file logically removed (not physically deleted)
- `txn`: streaming transaction ID (for idempotent streaming writes)
- `cdc`: Change Data Feed file reference
- `domainMetadata`: table features metadata

### Checkpoint Files

Every 10 commits (configurable), Delta writes a **Parquet checkpoint** that consolidates all log entries into a single file representing the full table state. Reading the table then requires: read checkpoint + replay JSON commits after it. Without checkpoints, reading a table with 10,000 commits means opening 10,000 JSON files.

```
_last_checkpoint file:
{ "version": 10, "size": 42 }
```

The `size` field is the number of actions in the checkpoint.

**Multi-part checkpoints** (large tables): a single checkpoint can be split into multiple Parquet files for parallelism.

---

## ACID via Optimistic Concurrency

### How Two Writers Detect Conflicts

```
Writer A                         Writer B
--------                         --------
Read table at version 5          Read table at version 5
Plan: add 3 files                Plan: UPDATE rows in part-00000.parquet
Write new parquet files          Write new parquet file (part-00000-new.parquet)
                                 Write remove(part-00000.parquet) + add(part-00000-new.parquet)
Write commit 6.json              Write commit 6.json  ← FAILS: 006.json already exists!
Commit succeeds                  Writer B reads 006.json, checks conflict:
                                   - A did blind append (isBlindAppend: true)
                                   - B modified part-00000, A didn't touch it
                                   → No actual conflict → Write as commit 7.json ✓
```

**Conflict detection rules**:
1. Try to write `N.json`. If it exists, someone else committed first.
2. Read their commit. Check if your operation conflicts:
   - Blind appends never conflict with anything.
   - UPDATE/DELETE conflict if the other writer `remove`d a file you also need to update.
   - Schema changes conflict with anything.
3. If no conflict: retry with version N+1.
4. If conflict: abort.

The **atomic write** of `N.json` is guaranteed by object store conditional PUT (S3, ADLS) or filesystem atomics (HDFS rename). Only one writer can create `N.json` — the loser sees HTTP 412 or equivalent.

---

## Transaction Log Replay

To reconstruct table state at version V:

```python
# Pseudocode
def read_table_at_version(V):
    checkpoint = find_latest_checkpoint_at_or_before(V)
    if checkpoint:
        state = load_checkpoint_parquet(checkpoint)  # full state
        start_version = checkpoint.version + 1
    else:
        state = empty_state()
        start_version = 0
    
    for version in range(start_version, V + 1):
        for action in parse_json(f"_delta_log/{version:020d}.json"):
            if action is Add:
                state.active_files.add(action.path)
            elif action is Remove:
                state.active_files.discard(action.path)
            elif action is MetaData:
                state.schema = action.schemaString
                state.partitionColumns = action.partitionColumns
    
    return state  # set of active parquet files + schema
```

**Time travel** is trivially: stop replay at version V instead of the latest.

---

## DML Operations: File-Level Mechanics

### UPDATE

```sql
UPDATE events SET status = 'processed' WHERE user_id = 42;
```

1. Scan table: find files containing `user_id = 42` (using min/max stats to skip files).
2. For each matching file:
   - Read full file.
   - Apply update to matching rows.
   - Write new parquet file with updated rows.
3. Commit: `remove(old_file)` + `add(new_file)` actions in one JSON commit.

Without deletion vectors (classic Delta): even if 1 row out of 10M needs updating, the whole file is rewritten.

### DELETE

Same as UPDATE, but instead of rewriting with changes, rows are simply omitted from the new file. The commit adds the new file and removes the old one.

### MERGE (UPSERT)

```sql
MERGE INTO events t
USING updates s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.status = s.status
WHEN NOT MATCHED THEN INSERT *;
```

1. Identify files that could contain matching rows (file pruning with join keys).
2. For each such file: read, apply merge logic row-by-row, write new file.
3. For non-matching source rows: write new files.
4. Commit: remove old files + add new files.

MERGE is the most expensive DML — it can rewrite most of the table if the join predicate doesn't prune well.

---

## Change Data Feed (CDF)

Tracks row-level changes (INSERT, UPDATE, DELETE) in a dedicated directory.

### Enable

```sql
ALTER TABLE events SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
-- or at CREATE:
CREATE TABLE events (...) TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
```

### What Gets Written

When CDF is enabled, DML operations additionally write **CDF data files**:

```
_delta_log/
  00000000000000000001.json:
    {"cdc": {"path": "_change_data/cdc-00000-abc.parquet", "partitionValues": {}, "size": 2048, "dataChange": false}}

_change_data/
  cdc-00000-abc.parquet   ← contains pre/post images of changed rows
```

The CDF parquet file includes all table columns plus:
```
_change_type: "insert" | "update_preimage" | "update_postimage" | "delete"
_commit_version: LONG
_commit_timestamp: TIMESTAMP
```

For an UPDATE: you get two rows per changed row — `update_preimage` (old) and `update_postimage` (new).

### Read CDF

```python
# Spark
df = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 5) \
    .option("endingVersion", 10) \
    .table("events")

# Or by timestamp
df = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingTimestamp", "2024-01-15 00:00:00") \
    .table("events")
```

**Use case**: propagate changes downstream to Silver/Gold tables without full scans. CDC pipelines.

---

## Deletion Vectors

Introduced in Delta 2.x (table writer version 7, reader version 3). Avoids file rewrites for small deletes. As of 2026, deletion vectors are not yet a universal default: Delta Live Tables enables them by default for materialized views/streaming tables (since April 2025), and Databricks has been rolling out "enabled by default for all new tables" at the workspace-admin-setting level through 2026 — but plain open-source Delta Lake tables still require the opt-in `delta.enableDeletionVectors = true` table property.

### Old Way (CoW)

```
Delete 1 row from a 128MB file → rewrite 128MB
```

### Deletion Vectors

A **deletion vector** (DV) is a bitset/bitmap stored as a separate small file that marks which rows in a data file are deleted.

```
events/
  part-00000-abc.parquet          ← original data file (unchanged)
  deletion_vector_abc.bin         ← bitset: "row 42, row 1337 are deleted"
```

Commit actions:
```json
{"add": {"path": "part-00000-abc.parquet", "deletionVector": {
  "storageType": "u",
  "pathOrInlineDv": "deletion_vector_abc.bin",
  "offset": 1,
  "sizeInBytes": 36,
  "cardinality": 2
}}}
```

At read time, the reader applies the DV as a row filter. Reads are slightly slower (must check DV), but writes are dramatically cheaper for small deletes.

**Compaction** eventually rewrites files, removing DV-marked rows physically.

---

## OPTIMIZE and ZORDER

### OPTIMIZE (bin-packing)

Merges small files into target-size files (default 1GB).

```sql
OPTIMIZE events;
OPTIMIZE events WHERE date = '2024-01-15';  -- partition-specific
```

Commit: adds new larger files, removes small files. Only active files in the table; doesn't touch history.

### ZORDER BY

Data layout optimization within OPTIMIZE (covered in detail in z-ordering.md).

```sql
OPTIMIZE events ZORDER BY (user_id, event_type);
```

Rewrites files so that rows with similar (user_id, event_type) values are co-located. Enables file pruning via min/max stats for those columns.

**Key point**: ZORDER is applied during OPTIMIZE — it's not a table property, it's a one-time operation. New incoming data is not Z-ordered until the next OPTIMIZE run.

**2026 status**: Databricks now recommends **Liquid Clustering** over ZORDER/partitioning for all new tables, including streaming tables and materialized views — it self-maintains clustering incrementally instead of requiring repeated full-partition rewrites (see liquid-clustering.md). ZORDER still works and is fully supported, but treat it as the legacy approach for new table design in the Databricks ecosystem; it remains the standard technique in open-source Delta/Iceberg contexts without Liquid Clustering.

---

## Schema Enforcement vs Evolution

### Enforcement (default)

Writing data with extra columns or wrong types → runtime error. Schema is validated before any files are written.

```python
# This fails if new_col doesn't exist in table schema
df.write.format("delta").mode("append").saveAsTable("events")
# AnalysisException: A schema mismatch detected when writing to the Delta table
```

### Evolution (opt-in)

```python
df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .saveAsTable("events")
```

Or permanently:
```sql
ALTER TABLE events SET TBLPROPERTIES ('delta.schema.autoMerge.enabled' = 'true');
```

**What mergeSchema does**: adds new columns to the table schema, widens compatible types (ByteType → LongType). Existing rows get NULL for new columns. **Does not** allow type narrowing or column drops via merge.

---

## Vacuum

Physically deletes files from storage that are no longer referenced by any version within the retention window.

```sql
-- Default: 7 days retention
VACUUM events;

-- Custom retention
VACUUM events RETAIN 168 HOURS;

-- Dry run (shows what would be deleted)
VACUUM events DRY RUN;
```

**What Vacuum deletes**:
- Parquet files referenced only by `remove` actions older than the retention window.
- Orphan files (never referenced by any commit).

**What Vacuum does NOT delete**:
- `_delta_log/` JSON and checkpoint files (those have separate cleanup via `delta.logRetentionDuration`).
- Files referenced by recent versions (within retention window).

**Critical gotcha**: If you `VACUUM` with a retention period shorter than your longest-running query, that query may fail mid-execution because its files were deleted. Default 7 days is safe for most workloads.

**Log retention** (separate from data retention):
```sql
ALTER TABLE events SET TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval 30 days',
  'delta.deletedFileRetentionDuration' = 'interval 7 days'
);
```

---

## Delta Sharing Protocol

Open protocol for sharing Delta tables with external parties **without copying data**.

```
Provider (has Delta table) → Delta Sharing Server → Recipient (any engine)
```

The server exposes:
- Table metadata (schema, version)
- Presigned URLs to data files (S3/ADLS/GCS) valid for a limited time

Recipients use the Delta Sharing open source connector (Spark, pandas, Power BI, etc.).

```python
# Recipient reads shared table
import delta_sharing
client = delta_sharing.SharingClient("config.share")  # contains credentials
df = delta_sharing.load_as_pandas(f"{profile}#schema.table")
```

The share file contains:
```json
{"shareCredentialsVersion": 1, "endpoint": "https://sharing.example.com/delta-sharing/", "bearerToken": "..."}
```

---

## Universal Format (UniForm)

UniForm lets a single copy of Parquet data be read as **Delta, Iceberg, or Hudi** by generating the other formats' metadata alongside the native Delta log — no data duplication or conversion job needed.

- **Iceberg compatibility**: GA since June 2024 (Databricks Runtime 14.3+). As of Delta 4.3 (June 2026), UniForm generates Iceberg metadata **atomically and incrementally** as part of the Delta commit itself, rather than a full-snapshot regeneration after the fact.
- **Hudi compatibility**: announced but still preview-level as of mid-2026 — don't treat it as GA without checking the current Delta release notes.

```sql
ALTER TABLE events SET TBLPROPERTIES (
  'delta.universalFormat.enabledFormats' = 'iceberg'
);
```

Any Iceberg-only reader (Trino, Snowflake, Athena) can then query the table without a separate Iceberg copy.

---

## Delta vs Iceberg: Concrete Differences

| Aspect | Delta Lake | Apache Iceberg |
|---|---|---|
| **Change tracking** | Transaction log (JSON append) | New snapshot + new manifest files |
| **Catalog requirement** | Path-based tables need none, but Databricks now defaults to Unity Catalog-managed tables | Requires external catalog (Hive, REST, Glue...); REST Catalog is now the de facto standard |
| **Multi-engine** | Much improved via Delta Kernel + UniForm — still slightly behind Iceberg in native engine breadth | First-class, any engine (broadest native support) |
| **Streaming** | Native (Structured Streaming) | Good (Flink native) |
| **Schema evolution** | Column names (no field IDs) | Field IDs — safer |
| **Partition evolution** | No true metadata-only partition evolution; Liquid Clustering covers some of the same use case differently | First-class feature (metadata-only, no rewrite) |
| **Hidden partitioning** | No | Yes (transform functions) |
| **CDF / Change feed** | Built-in (`enableChangeDataFeed`); newer "automatic CDF" computes changes at query time to cut write overhead | Requires CDC-capable writer |
| **Deletion vectors** | Delta 2.x+, increasingly default-on (DLT, Databricks-managed tables) but still opt-in in OSS | v3 spec (2025/2026) adds binary deletion vectors, replacing v2 position deletes |
| **Compaction** | OPTIMIZE command (manual); Optimized Writes/Auto Compaction in OSS since 3.1.0; Liquid Clustering auto-manages layout | `rewrite_data_files` procedure (manual/scheduled) |
| **Time travel syntax** | `VERSION AS OF N` | `VERSION AS OF N` (same) |
| **Log cleanup** | Vacuum | `expire_snapshots` + `remove_orphan_files` |
| **Ecosystem** | Databricks-centric, but UniForm/Kernel narrow this | Vendor-neutral |
| **File statistics** | In `add` action JSON (per-file) | In manifest Avro (per-file + per-manifest) |
| **Cross-format reads** | UniForm: read as Iceberg (GA) or Hudi (preview) from one copy | N/A (is the target format) |

**Key architectural difference**: Delta's log is self-contained (no catalog needed for path-based access), but the ecosystem has shifted toward catalog-managed tables on both sides — Databricks pushes Unity Catalog-managed Delta/Iceberg tables, while Iceberg's REST Catalog spec has become a portable, multi-vendor standard (Polaris, Nessie, Unity Catalog, Glue, Snowflake Open Catalog all implement it). The "no catalog vs. catalog required" framing is less of a real-world differentiator in 2026 than it was — most production deployments of either format use a catalog for multi-engine discovery.

---

## UPDATE Flow: End-to-End Worked Example

**Table**: `events`, 3 parquet files, no partitions, version 5.

**Operation**: `UPDATE events SET status = 'done' WHERE user_id = 42`

**Step 1 — Plan**
Read `_delta_log/` up to version 5 → active files: `[part-00000.parquet, part-00001.parquet, part-00002.parquet]`.

Read per-file stats from commit actions:
```
part-00000: user_id min=1,    max=100
part-00001: user_id min=1,    max=500   ← user_id 42 could be here
part-00002: user_id min=501,  max=9999  ← skip: 42 not in [501,9999]
```
Only `part-00000` and `part-00001` need to be read.

**Step 2 — Execute**

For `part-00000.parquet` (128MB):
- Read all rows.
- Find rows where `user_id = 42` → 3 rows found.
- Write `part-00000-new-xyz.parquet` with those 3 rows having `status = 'done'`.

For `part-00001.parquet` (128MB):
- Read all rows.
- Find rows where `user_id = 42` → 0 rows.
- No rewrite needed (optimization: if no rows matched, skip).

Actually with deletion vectors (Delta 2.x+):
- `part-00000.parquet` not rewritten.
- Write deletion vector for old rows + new file for updated rows.

**Step 3 — Commit**

Write `_delta_log/00000000000000000006.json`:
```json
{"commitInfo": {"operation": "UPDATE", "operationParameters": {"predicate": "(user_id = 42)"}}}
{"remove": {"path": "part-00000.parquet", "deletionTimestamp": 1715000000000, "dataChange": true}}
{"add": {"path": "part-00000-new-xyz.parquet", "size": 134217728, "dataChange": true, "stats": "..."}}
```

**Result**: Version 6 active files: `[part-00000-new-xyz.parquet, part-00001.parquet, part-00002.parquet]`.

`part-00000.parquet` is still physically on storage — removed only after Vacuum runs past the retention window.

---

## Key Gotchas

- **`_delta_log/` performance at scale**: 10,000+ JSON files → slow table opens. Checkpoint interval tuning is important (default 10 commits). For high-frequency writes (streaming), consider `logRetentionDuration` and checkpoint tuning.
- **UPDATE rewrites entire files**: Without deletion vectors, any UPDATE/DELETE touching a large file rewrites it completely. Use deletion vectors for point deletes; use MERGE carefully.
- **OPTIMIZE is not automatic by default**: You must still run OPTIMIZE explicitly for on-demand compaction. However, Optimized Writes and Auto Compaction are no longer Databricks-proprietary — both have shipped in open-source Delta Lake since 3.1.0. Small file accumulation still degrades read performance silently if none of these are enabled.
- **Vacuum removes time travel data**: Setting retention too short removes files needed for time travel. Setting too long wastes storage. 7 days is the safe minimum.
- **CDF doubles storage on UPDATE**: With CDF enabled, every UPDATE writes both the changed data file AND a CDF file. Monitor storage growth.
- **Schema evolution does NOT support column drops**: `mergeSchema` only adds/widens. To drop a column, you need `overwriteSchema = true`, which requires a full table rewrite.
- **Concurrent streaming writers**: Multiple Structured Streaming jobs writing to the same Delta table — Delta's txn action (`{"txn": {"appId": "...", "version": N}}`) provides exactly-once semantics per streaming app.
- **Partition column != regular column**: Partition columns in Delta are stored in file paths, not in Parquet row data. Reading partition columns is free (from path); filtering on them prunes files. Don't add high-cardinality columns as partitions — too many directories kills listing performance.
