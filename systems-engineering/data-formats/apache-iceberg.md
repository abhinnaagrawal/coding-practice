# Apache Iceberg

## 30-Second Intuition

Iceberg is a **table format** — a specification for how to organize data files and metadata on object storage so that you get ACID transactions, schema evolution, and time travel without a heavyweight metastore. Think of it as a filesystem-level protocol that any engine (Spark, Flink, Trino, DuckDB) can speak. The core trick: every write produces an immutable **snapshot** pointing to a tree of metadata files that together describe exactly which data files exist. Readers pin a snapshot; writers append new snapshots. No locks needed.

---

## The Problem Iceberg Solves

### Hive Table Format Pain Points

| Problem | Hive | Iceberg |
|---|---|---|
| ACID on object storage | No (need ORC transactions or HBase) | Yes, snapshot isolation |
| List all files in partition | `LIST s3://bucket/year=2024/month=01/` O(files) | Manifest lookup, O(log n) |
| Change partition scheme | Rewrite all data | Partition evolution, no rewrite |
| Rename column | Breaks all downstream readers | Safe, uses field IDs |
| Schema at read vs write | Reader must know schema by convention | Schema stored in metadata |
| Concurrent writers | Last write wins / silent corruption | Optimistic concurrency, conflict detection |
| Count rows | Full scan | Metadata statistics |

### The Full Partition Scan Problem

In Hive, adding a new partition column means:
```
s3://warehouse/events/year=2024/month=01/day=15/file1.parquet
```
Querying `WHERE event_type = 'click'` forces a full scan of every file — there's no index into non-partition columns. Iceberg solves this with per-file min/max statistics stored in manifests.

---

## Table Format Spec: The 3-Layer Tree

```
Table (logical)
│
├── catalog entry  →  points to current metadata.json
│
└── metadata.json          (1 file, ~KB)
    ├── table schema
    ├── partition specs
    ├── snapshot history
    └── current-snapshot-id ──────────────────────────────────┐
                                                               ▼
                                              manifest-list-<snap>.avro  (~KB)
                                              ┌────────────────────────────┐
                                              │ manifest_path              │
                                              │ partition_spec_id          │
                                              │ added_rows_count           │
                                              │ partition_stats (min/max)  │  × N
                                              └────────────┬───────────────┘
                                                           │
                                                           ▼
                                          manifest-<uuid>.avro  (~MB)
                                          ┌──────────────────────────────┐
                                          │ file_path                    │
                                          │ file_format (parquet/orc/avro│
                                          │ partition_values             │
                                          │ record_count                 │
                                          │ file_size_in_bytes           │
                                          │ column_sizes                 │
                                          │ value_counts                 │
                                          │ null_value_counts            │
                                          │ lower_bounds  ◄── per column │
                                          │ upper_bounds  ◄── per column │
                                          └──────────────────────────────┘
                                                           │
                                                           ▼
                                          data-<uuid>.parquet  (actual data)
```

**Layer 1 — Metadata file**: JSON, describes the table (schema, partition specs, list of snapshots). Small, always read first.

**Layer 2 — Manifest list**: Avro file, one entry per manifest. Stores partition-level stats so the engine can skip entire manifests. O(num_manifests) to scan for pruning.

**Layer 3 — Manifest file**: Avro file, one entry per data file. Stores column-level min/max stats. Used for file-level pruning.

**Data files**: Parquet (most common), ORC, or Avro. Immutable once written.

---

## Snapshot Isolation

Every write (INSERT, UPDATE, DELETE, compaction) creates a **new snapshot**:

```
metadata.json (before):
  current-snapshot: snap-001
  snapshots: [snap-001]

-- someone runs: INSERT INTO events VALUES (...)

metadata.json (after):
  current-snapshot: snap-002
  snapshots: [snap-001, snap-002]
```

`snap-002` points to a new manifest list. That manifest list may reuse manifests from `snap-001` for unchanged partitions (manifest reuse is key for write efficiency).

**Readers always see a consistent view**: they resolve the snapshot at query start and follow its manifest chain. Writers do not invalidate in-progress readers.

**Optimistic concurrency**: If two writers both read `snap-001` and try to commit `snap-002`, one will fail the atomic catalog swap. The loser retries (if operations are compatible) or aborts (if they conflict — e.g., both modified the same partition).

### Snapshot Properties

```json
{
  "snapshot-id": 3519467992,
  "parent-snapshot-id": 3051729675,
  "timestamp-ms": 1715000000000,
  "manifest-list": "s3://wh/db/events/metadata/snap-3519467992.avro",
  "summary": {
    "operation": "append",
    "added-data-files": "3",
    "added-records": "150000",
    "total-records": "1500000"
  }
}
```

---

## Schema Evolution

Iceberg uses **field IDs** (integers), not column names. Column names are just aliases.

```sql
-- Original schema
CREATE TABLE events (
  id    BIGINT,   -- field-id: 1
  ts    TIMESTAMP,-- field-id: 2
  data  STRING    -- field-id: 3
);

-- Rename column — safe because readers use field-id: 3, not name
ALTER TABLE events RENAME COLUMN data TO payload;

-- Add column — old files just return NULL for new column
ALTER TABLE events ADD COLUMN user_id BIGINT; -- field-id: 4

-- Drop column — old files still have bytes for it, just ignored
ALTER TABLE events DROP COLUMN id;
```

Safe operations: **add, drop, rename, reorder, widen type** (int → long).

Unsafe (requires explicit schema overwrite): narrowing types.

Old data files are never rewritten — the schema in `metadata.json` tracks field ID → name mappings. Readers use field IDs to locate columns in Parquet column metadata.

---

## Partition Evolution

Hive: partition scheme is burned into the directory structure. Change it → full data rewrite.

Iceberg: partition spec is stored in metadata, separate from data file paths. Each manifest records which partition spec it was written under.

```sql
-- Original: partition by month
ALTER TABLE events ADD PARTITION FIELD months(ts);

-- After 2 years, switch to daily — zero data rewrite
ALTER TABLE events DROP PARTITION FIELD months(ts);
ALTER TABLE events ADD PARTITION FIELD days(ts);
```

Old manifests retain their original partition spec ID. New data is written under the new spec. Queries work across both because Iceberg normalizes partition values during planning.

---

## Hidden Partitioning

In Hive, you write partition values explicitly:
```sql
INSERT INTO events PARTITION (year=2024, month=1) SELECT ...
```

In Iceberg, you declare a **partition transform** and the engine handles it:
```sql
CREATE TABLE events (ts TIMESTAMP, user_id BIGINT, ...)
PARTITIONED BY (days(ts), bucket(16, user_id));
```

When you write a row with `ts = 2024-01-15 10:30:00`, Iceberg computes:
- `days(ts)` → `19737` (days since epoch) — used as partition value
- `bucket(16, user_id)` → `hash(user_id) % 16` — used as partition value

The **data file path** is something like:
```
s3://wh/db/events/data/ts_day=19737/user_id_bucket=7/00000-1-abc.parquet
```

Queries with `WHERE ts BETWEEN '2024-01-15' AND '2024-01-20'` automatically prune to days 19737–19742. No user-facing partition column needed. **The partition column does not appear in the table schema** — hence "hidden."

### Transforms Available

| Transform | Example | Use Case |
|---|---|---|
| `identity(col)` | exact value | low-cardinality strings |
| `bucket(N, col)` | hash mod N | high-cardinality IDs |
| `truncate(W, col)` | first W chars/bits | string prefix |
| `years(ts)` | year number | coarse time |
| `months(ts)` | months since epoch | medium time |
| `days(ts)` | days since epoch | fine time |
| `hours(ts)` | hours since epoch | very fine time |
| `void(col)` | always null | disable partition |

---

## Row-Level Deletes

Iceberg supports two strategies. Note: as of the **v3 table format spec** (ratified 2025, shipping broadly in engines through 2026 — e.g. Snowflake GA May 2026, Databricks/Unity Catalog managed Iceberg v3 GA), v3 adds **binary deletion vectors** (roaring bitmaps stored in Puffin files) as the new default row-delete mechanism, superseding position-delete files for v3 tables. This closes the long-standing gap with Delta Lake's deletion vectors. The two classic strategies below (CoW and position/equality delete files) remain how v2 tables — and most production Iceberg tables today — work.

### Copy-on-Write (CoW)

When a row is deleted or updated:
1. Read the affected data file.
2. Write a new data file with the row removed/changed.
3. New snapshot points to new file; old file is no longer referenced.

**Pros**: Reads are fast (no merge needed).
**Cons**: Every small delete rewrites entire files. Bad for frequent updates.

### Merge-on-Read (MoR)

Delete operations write **delete files** that are applied at read time.

**Position delete files**: "Row at position 42 in file `data-abc.parquet` is deleted."
```
Avro schema: { file_path: string, pos: long }
```

**Equality delete files**: "All rows where `user_id = 999` are deleted."
```
Avro schema: { user_id: long }  -- subset of table schema
```

At read time, the engine:
1. Reads data files.
2. Applies position deletes (fast — positional join).
3. Applies equality deletes (hash join on delete columns).

**Pros**: Writes are fast (append delete file, no rewrite).
**Cons**: Read amplification grows with number of unapplied deletes. Need periodic compaction.

### In Practice (Flink CDC → Iceberg)

Streaming ingestion from CDC typically uses MoR (equality deletes for UPDATEs). Batch Spark compaction periodically rewrites to CoW files, clearing accumulated delete files.

---

## Metadata Tables

Every Iceberg table exposes virtual metadata tables:

```sql
-- History of all snapshots
SELECT * FROM events.history;
-- snapshot_id, parent_id, is_current_ancestor, made_current_at

-- All snapshots with summaries
SELECT * FROM events.snapshots;
-- committed_at, snapshot_id, parent_id, operation, manifest_list, summary

-- All manifests in current snapshot
SELECT * FROM events.manifests;
-- path, length, partition_spec_id, added_snapshot_id, added_data_files_count, ...

-- All data files in current snapshot
SELECT * FROM events.files;
-- content (DATA/POSITION_DELETES/EQUALITY_DELETES), file_path, file_format,
-- record_count, file_size_in_bytes, column_sizes, value_counts,
-- null_value_counts, lower_bounds, upper_bounds

-- All delete files
SELECT * FROM events.delete_files;

-- Partitions and their stats
SELECT * FROM events.partitions;
-- partition, spec_id, record_count, file_count, ...

-- References (branches and tags)
SELECT * FROM events.refs;
```

These are invaluable for understanding table health, debugging slow queries, and capacity planning.

---

## Catalog Types

A **catalog** is the service that maps `database.table_name → current metadata.json path`. That's its core job. Everything else is in the metadata files.

| Catalog | Storage | Atomic Swap Mechanism |
|---|---|---|
| **Hive Metastore** | MySQL/Postgres (via HMS) | HMS table property update |
| **REST Catalog** | Any backend (server decides) | HTTP PUT with precondition |
| **JDBC** | Any JDBC-compatible DB | SELECT FOR UPDATE + UPDATE |
| **AWS Glue** | Glue Data Catalog | Glue `UpdateTable` API |
| **Nessie** | JGit-based | Git-like branching + merge |
| **DynamoDB** | DynamoDB table | Conditional writes |

**What a catalog stores** (for each table):
```
table_name → {
  metadata_location: "s3://wh/db/events/metadata/v42.json",
  previous_metadata_location: "s3://wh/db/events/metadata/v41.json"
}
```

The atomic catalog update (`v41.json` → `v42.json`) is the **commit point**. If two writers race, the catalog's atomic-write guarantee (HMS lock, DynamoDB conditional write, etc.) determines the winner.

---

## Time Travel and Rollback

```sql
-- Query at a specific snapshot
SELECT * FROM events VERSION AS OF 3519467992;

-- Query at a specific timestamp
SELECT * FROM events TIMESTAMP AS OF '2024-01-15 00:00:00';

-- Rollback table to previous snapshot (Spark procedure)
CALL catalog.system.rollback_to_snapshot('db.events', 3519467992);

-- Rollback to timestamp
CALL catalog.system.rollback_to_timestamp('db.events', TIMESTAMP '2024-01-14');
```

Rollback creates a **new snapshot** whose manifest list equals that of the target snapshot. It does not delete data files — those are still there until `expire_snapshots` + `vacuum`.

---

## Compaction and Maintenance

### expire_snapshots

Removes old snapshot references from `metadata.json`. Does NOT delete data files.
```sql
CALL catalog.system.expire_snapshots(
  table => 'db.events',
  older_than => TIMESTAMP '2024-01-01',
  retain_last => 10
);
```

### remove_orphan_files

Deletes data files on storage that are not referenced by any snapshot. Run after failed writes or external manipulation.
```sql
CALL catalog.system.remove_orphan_files(table => 'db.events');
```

### rewrite_data_files (compaction)

Merges small files, rewrites MoR delete files into CoW files, re-sorts data.
```sql
CALL catalog.system.rewrite_data_files(
  table => 'db.events',
  strategy => 'sort',
  sort_order => 'zorder(user_id, event_type)',
  options => map('target-file-size-bytes', '134217728')  -- 128MB
);
```

### rewrite_manifests

Rewrites manifests to reflect current partition spec, consolidate small manifests.
```sql
CALL catalog.system.rewrite_manifests(table => 'db.events');
```

---

## Write Flow: End-to-End Worked Example

**Setup**: `events` table on S3, partitioned by `days(ts)`, REST catalog.

**Operation**: `INSERT INTO events SELECT * FROM staging WHERE dt = '2024-01-15'`

**Step 1 — Engine plans write**
- Reads current `metadata.json` from catalog → gets current snapshot, partition spec.
- Determines output partition: `days('2024-01-15') = 19737`.

**Step 2 — Write data files**
```
s3://wh/db/events/data/ts_day=19737/
  00000-1-a3f4b2c1.parquet   (128MB)
  00000-2-b5e6d7f8.parquet   (128MB)
  00000-3-c9a1b2d3.parquet   (64MB)
```
These are written first — they exist but are not yet "in" the table.

**Step 3 — Write manifest file**
```
s3://wh/db/events/metadata/
  a1b2c3d4-manifest.avro
```
Contains 3 entries, one per data file, with min/max stats per column.

**Step 4 — Write manifest list (snapshot)**
```
s3://wh/db/events/metadata/
  snap-3519467992.avro
```
Contains entries for all manifests in the new snapshot. Includes the new manifest from step 3 plus all manifests from the previous snapshot that cover other partitions (manifests are reused).

**Step 5 — Write new metadata.json**
```
s3://wh/db/events/metadata/
  v43.json
```
Points to `snap-3519467992.avro` as current snapshot.

**Step 6 — Atomic catalog update**
```
REST PUT /v1/namespaces/db/tables/events
  { "requirements": [{ "type": "assert-current-schema-id", ... }],
    "updates": [{ "action": "set-current-schema", ... },
                { "action": "add-snapshot", ... },
                { "action": "set-snapshot-ref", ... }] }
```
Server swaps `metadata_location` from `v42.json` to `v43.json` atomically. If another writer committed first, this fails → retry.

**Files created in this operation**:
- 3 data parquet files
- 1 manifest avro file
- 1 manifest list avro file
- 1 metadata json file

**Total new files**: 6. Old files untouched.

---

## Table Format Spec Versions: v2 vs v3

Everything above describes the **v2** spec, which is still what most production tables run today. The **v3** spec (finalized 2025, shipping in engines through 2026 — Apache Iceberg library 1.11.0, May 2026) adds:

| v3 Feature | What it does |
|---|---|
| Deletion vectors | Binary (roaring bitmap) row-delete files in Puffin format, replacing position deletes |
| Variant type | Native semi-structured/JSON column type, with "shredding" for efficient storage/query |
| Row lineage | `_row_id` / `_last_updated_sequence_number` fields track row history across snapshots |
| Geospatial types | Native `geometry`/`geography` types (WKB-encoded) with bounding-box stats for pruning |
| Table encryption | Client-side encryption of table/metadata with snapshot-level access control |
| Multi-argument transforms | Partition/sort transforms spanning multiple columns |
| Default values | Column default values for schema evolution |

Engine support for v3 features is still uneven as of mid-2026 (e.g., geometry-based file pruning works in some engines but not others) — check your engine's release notes before relying on a v3-only feature.

---

## Comparison with Hive Table Format

| Aspect | Hive | Iceberg |
|---|---|---|
| File discovery | `LIST` directory recursively | Read manifest files |
| ACID | External (LLAP, ORC AcidV2) | Native snapshot isolation |
| Schema stored | In metastore only | In metadata.json per snapshot |
| Partition changes | Rewrite data | Evolve spec, no rewrite |
| Column rename | Breaks readers | Safe (field IDs) |
| File statistics | None (must open files) | Stored in manifests |
| Time travel | Not native | Built-in via snapshots |
| Multi-engine | Mostly Hive | Any engine |
| Concurrent writes | Metastore locks | Optimistic concurrency |

---

## Key Gotchas

- **Catalog is the critical dependency**: Lose the catalog → lose the pointer to metadata → "lose" the table (data still on S3, but unaddressable without the metadata path). Always back up catalog.
- **Small files are expensive**: Each manifest entry is cheap to read, but a table with millions of tiny files accumulates huge manifests. Run compaction regularly.
- **Equality deletes scale poorly**: Accumulating equality delete files means every read must do a hash join. Compact MoR tables frequently.
- **Manifest reuse is not free**: Adding a single row to every partition creates a new snapshot with a new manifest list but reuses all old manifests — efficient. Adding rows to one partition creates one new manifest + reuses the rest.
- **`expire_snapshots` does NOT free storage**: You must also `remove_orphan_files` (or have the catalog track orphans). Many teams forget this and storage grows unboundedly.
- **REST catalog is preferred for multi-engine**: HMS has locking quirks with non-Hive engines. REST catalog is engine-agnostic by design, and by 2026 it's the de facto standard — implemented by Apache Polaris, Project Nessie, Unity Catalog, AWS Glue (adapter), and Snowflake Open Catalog. Note the spec still doesn't mandate performance/latency guarantees, so implementations vary in practice.
- **Field ID assignment**: Field IDs are assigned by the catalog, not the user. When merging schemas from two sources, ID conflicts are possible — always use the same catalog for schema management.

## When to Use / When NOT to Use

**Use Iceberg when**:
- Multiple compute engines need to read/write the same tables.
- You need schema evolution without ETL rewrites.
- CDC/streaming ingestion with batch corrections (MoR + compaction).
- Time travel or audit history is required.
- Tables exceed 100GB — file statistics pay off at scale.

**Don't use (or consider alternatives) when**:
- Small tables < 1GB — overhead of manifest files is disproportionate.
- Single-engine (e.g., pure Databricks) — Delta Lake has tighter integration.
- You need row-level updates at very high frequency without compaction — MoR delete files accumulate fast.
- Your catalog infrastructure is not mature — Iceberg without a reliable catalog is a footgun.
