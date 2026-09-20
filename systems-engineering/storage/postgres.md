# PostgreSQL

## 30-Second Intuition

PostgreSQL is a general-purpose, ACID-compliant relational database built around **MVCC implemented directly in the heap** — every row version, old and new, lives in the same table file, and a background process (VACUUM) is responsible for reclaiming the ones nobody can see anymore. This is a fundamentally different design choice from databases that keep only the current row in place and push old versions into a separate undo/rollback segment (Oracle, MySQL/InnoDB, SQL Server) — see the Signature Mechanism section for why that distinction is Postgres's single most operationally important fact. The one thing that matters most day to day: **a `UPDATE` in Postgres is really an INSERT-plus-mark-dead**, which means write-heavy tables bloat unless VACUUM keeps up, and "vacuum tuning" is a first-class, recurring operational concern here in a way it simply isn't for undo-log databases.

Current stable major version as of September 2026 is **PostgreSQL 18** (GA September 25, 2025, on 18.6 as of August 2026), which shipped a new asynchronous I/O subsystem (`io_method`, default `worker` on non-Linux, `io_uring` available on Linux), skip-scan for multicolumn B-tree indexes, virtual generated columns, `uuidv7()`, OAuth 2.0 auth, and a round of autovacuum throughput/threshold improvements (`autovacuum_vacuum_max_threshold`, runtime-adjustable `autovacuum_worker_slots`/`autovacuum_max_workers`). PostgreSQL 17 (2024) is the still-common prior baseline and is the release that stabilized logical replication failover slots (`failover` parameter, `sync_replication_slots`), which matters directly for CDC pipelines built on `pgoutput`/Debezium.

---

## Resource-Layer Map

| Layer | Role Postgres plays | What it's optimizing for |
|---|---|---|
| Disk | Two disk-resident structures dominate: the **heap** (table data, including every live and not-yet-vacuumed dead tuple version) and the **WAL** (append-only redo log). WAL fsync on commit is the single latency-critical write path in the entire engine — a transaction cannot return `COMMIT` to the client until its WAL record is durably on disk (unless `synchronous_commit = off`). Heap and index access for reads is random I/O by default (8KB pages scattered across the file), which is why sequential scans and WAL writes behave completely differently from point lookups | Durability of the *log* above all else on the write path (sequential, fsync-bound); minimizing random page faults on the read path via the buffer cache and index structure |
| Memory | `shared_buffers` is Postgres's own buffer cache, sized explicitly (commonly 25% of RAM) and living **underneath** the OS page cache, not instead of it — a page evicted from `shared_buffers` is very likely still resident in the OS page cache, so a "miss" in Postgres's own cache is often still a cheap read, not a disk seek. This double-buffering is a classic Postgres tuning question: too-small `shared_buffers` causes redundant copying between the two caches, too-large `shared_buffers` starves the OS cache and doesn't help because Postgres relies on the OS to actually flush dirty pages and the OS to cache read-mostly data efficiently | Keep hot pages in a cache Postgres directly controls (for locking/pinning semantics), while still letting the OS absorb the working set that doesn't fit in `shared_buffers` |
| CPU | Query planner (cost-based, statistics-driven) and executor (Volcano/iterator model, one tuple at a time per operator — no vectorization by default). For typical OLTP point-lookup/short-transaction workloads, CPU is rarely the bottleneck; the planner's job is precisely to avoid CPU-expensive plans (seq scans, sorts, hash builds) whenever an index can substitute cheaper random I/O for it | Choosing the plan that minimizes total estimated cost (I/O-weighted, not CPU-weighted first) — this is why `seq_page_cost`/`random_page_cost` dominate planner decisions more than CPU cost constants do |
| Network | Client/server protocol (simple or extended query protocol) over TCP; each connection is a dedicated OS process, not a lightweight thread or fiber — this is why connection *count*, not just query volume, is a first-class capacity constraint (see Gotchas) | Correctness/isolation per connection (a crashing backend can't corrupt another session's state) at the cost of per-connection memory and OS scheduling overhead |
| GPU | Not applicable — the core engine has no GPU execution path; some extensions (e.g. certain vector-index or ML extensions) may use GPU, but this is not part of core Postgres | N/A |

The clearest cross-system contrast: this table is the mechanical explanation for why `coordination/etcd.md` says "unlike Postgres" when describing its own space model — etcd is disk-fsync-bound on a *replicated Raft log* with no in-place reuse until explicit compaction, whereas Postgres is disk-fsync-bound on WAL for durability **and separately** manages in-place page reuse in the heap via VACUUM. Same "disk is the bottleneck" shape, structurally different second-order mechanism (see Comparative section).

---

## The Signature Mechanism: MVCC via In-Heap Tuple Versioning (No Separate Undo Log)

Every row in a Postgres heap file carries two hidden system columns, `xmin` and `xmax` — the transaction ID that created this tuple version and the transaction ID that deleted/superseded it (0/null if still current). An `UPDATE` does **not** modify a row in place: it writes a brand-new tuple version into the heap (usually the same page if room permits, HOT-updated; otherwise a new page), sets the old tuple's `xmax` to the updating transaction's ID, and sets the new tuple's `xmin` to that same ID. The old tuple is now a **dead tuple** — invisible to any transaction that starts after the updater commits, but still physically occupying heap space.

```
Heap page, before UPDATE:
┌────────────────────────────────────────────┐
│ tuple (xmin=100, xmax=NULL) balance=500     │  ← visible, current
└────────────────────────────────────────────┘

UPDATE accounts SET balance = 400 WHERE id = 1;  -- run by txn 105

Heap page, after UPDATE (same page, HOT update):
┌────────────────────────────────────────────┐
│ tuple (xmin=100, xmax=105) balance=500      │  ← now DEAD once txn 105 commits
│                                                  and no older txn can see it
│ tuple (xmin=105, xmax=NULL) balance=400     │  ← new current version
└────────────────────────────────────────────┘
```

This is the crux of why vacuum tuning is a first-class Postgres concern in a way it is not for Oracle/InnoDB-style engines: those systems keep exactly one physical copy of a row in the table and write the *old* value into a separate undo/rollback segment, which is reclaimed automatically and quickly once the transaction that needed it for rollback/MVCC-read finishes — the "hot" table storage itself never grows from updates. Postgres inverts this: the table **is** the version store, so every update inflates the heap until a separate, asynchronous process (autovacuum) walks the pages, identifies tuples with `xmax` older than the oldest transaction anyone could still need to see (`xmin` horizon), and marks that space free for reuse. If that process falls behind — long transactions holding an old snapshot, insufficient autovacuum workers, huge burst of updates — heap and index bloat accumulate directly and visibly as wasted disk and slower scans, which is a categorically different failure mode from an undo-log database (whose failure mode under similar pressure is usually "rollback segment too small" / "snapshot too old" errors, not creeping bloat of the primary data file).

```sql
-- See the hidden system columns directly
SELECT xmin, xmax, ctid, * FROM accounts WHERE id = 1;
```

```
 xmin | xmax |  ctid  | id | balance
------+------+--------+----+---------
  105 |    0 | (0,2)  |  1 |     400
```

---

## High-to-Low Walkthrough: One `UPDATE`, Client to Bytes

The literal statement, run in `psql`:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
COMMIT;
```

```
psql: UPDATE accounts SET balance = balance - 50 WHERE id = 1;
      │
      ▼
Parser/Analyzer/Planner        parse tree → rewritten query → planner picks
                                 access path (index scan on accounts_pkey to
                                 find id=1, since WHERE id = 1 is an equality
                                 on the primary key)
      │
      ▼
Executor: fetch target tuple    index scan: B-tree traversal on accounts_pkey
                                 returns a (page, offset) pointer — a CTID —
                                 into the heap; heap page fetched into
                                 shared_buffers (buffer hit or a random-I/O
                                 read if not cached)
      │
      ▼
WAL record built                a WAL record describing "insert new tuple
(in-memory, in WAL buffers)      version, update xmax of old tuple, update
                                 index if needed" is constructed
      │
      ▼
WAL append (sequential write)   WAL record appended to the current WAL
                                 segment file — sequential I/O, cheap
      │
      ▼
COMMIT → WAL fsync              *this* is the latency-critical step: the
(disk, synchronous)             backend blocks until the WAL fsync (or
                                 group-commit-batched fsync) confirms the
                                 commit record is durable. Nothing about
                                 the heap page itself needs to be flushed
                                 yet — WAL durability alone makes the
                                 transaction durable (WAL = "write-ahead":
                                 log first, data pages can follow later,
                                 driven by the background writer/checkpoint)
      │
      ▼
Heap tuple versions              old tuple's xmax set to this txn's ID
(in shared_buffers, written        (now dead once no snapshot needs it);
 back to disk on its own            new tuple written with xmin = this
 schedule, not synchronously)       txn's ID, in the same page if a HOT
                                     update is possible (no indexed column
                                     changed, room on page) — HOT updates
                                     skip re-inserting into every index
      │
      ▼
Index update (if not HOT)        if balance were indexed (it isn't here),
                                   or if the page had no room, a new
                                   index entry is inserted pointing at the
                                   new tuple's CTID — the old index entry
                                   is left for VACUUM to clean up later
      │
      ▼
Background writer / checkpoint   dirty shared_buffers pages (heap + index)
                                   are flushed to the actual heap/index
                                   files on disk on their own cadence,
                                   decoupled from the client's COMMIT
```

**Confirming the plan and actual buffer activity** with `EXPLAIN (ANALYZE, BUFFERS)`:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
```

```
Update on accounts  (cost=0.15..8.17 rows=1 width=14) (actual time=0.041..0.042 rows=0 loops=1)
  Buffers: shared hit=4 dirtied=2
  ->  Index Scan using accounts_pkey on accounts  (cost=0.15..8.17 rows=1 width=14)
        (actual time=0.018..0.019 rows=1 loops=1)
        Index Cond: (id = 1)
        Buffers: shared hit=3
Planning:
  Buffers: shared hit=6
Planning Time: 0.089 ms
Execution Time: 0.067 ms
```

`shared hit=4` means all touched pages were already resident in `shared_buffers` (no disk read needed for this update); `dirtied=2` is the heap page (new tuple version + old tuple's `xmax`) and the index page/entry needing eventual write-back. A cold-cache version of the same query would show `read=N` buffers instead of `hit`.

**Checking dead-tuple accumulation and autovacuum activity** after a burst of updates:

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

```
 relname  | n_live_tup | n_dead_tup |      last_autovacuum      | autovacuum_count
----------+------------+------------+----------------------------+-------------------
 accounts |     50000  |       812  | 2026-09-13 14:02:11.100+00 |                 3
```

```sql
-- Is autovacuum currently running against this table, right now?
SELECT pid, datname, relid::regclass, phase, heap_blks_scanned, heap_blks_total
FROM pg_stat_progress_vacuum;
```

```
  pid  | datname |  relid   |          phase           | heap_blks_scanned | heap_blks_total
-------+---------+----------+---------------------------+-------------------+-----------------
 41822 | prod    | accounts | scanning heap             |             8420  |           12000
```

```sql
-- All active backends and what they're waiting on (useful when vacuum seems stuck)
SELECT pid, state, wait_event_type, wait_event, query, xact_start
FROM pg_stat_activity
WHERE state != 'idle';
```

---

## Deep Internals

### xmin/xmax Visibility — A Concrete Two-Transaction Example

```sql
-- Session A
BEGIN;                                         -- txn 200 starts
SELECT * FROM accounts WHERE id = 1;           -- sees balance=500 (xmin=100)

-- Session B, concurrently
BEGIN;                                         -- txn 201 starts
UPDATE accounts SET balance = 400 WHERE id = 1;
COMMIT;                                        -- txn 201 commits: old tuple
                                                -- xmax=201, new tuple xmin=201

-- Back in Session A (still inside its original transaction, REPEATABLE READ or
-- the default READ COMMITTED re-reading mid-transaction shows the difference):
SELECT * FROM accounts WHERE id = 1;
```

Under **READ COMMITTED** (Postgres's default), Session A's second `SELECT` sees `balance=400` — each statement gets a fresh snapshot, so it now sees txn 201's committed change. Under **REPEATABLE READ**, Session A's second `SELECT` still sees `balance=500` — the snapshot taken at `BEGIN` fixes visibility for the whole transaction, so the tuple with `xmin=201` is invisible (201 wasn't yet committed, or didn't exist, when A's snapshot was taken), and the tuple with `xmin=100, xmax=201` is still visible to A specifically because A's snapshot predates txn 201's commit. This is the entire visibility rule in miniature: **a tuple is visible to your snapshot if its `xmin` committed before your snapshot was taken and either `xmax` is null or `xmax`'s deleting transaction had not committed before your snapshot** — no separate undo log is consulted; the heap tuple itself, plus the requesting transaction's snapshot, is enough.

```sql
-- Confirm current isolation level and see snapshot-dependent xmin/xmax directly
SHOW default_transaction_isolation;
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT xmin, xmax, balance FROM accounts WHERE id = 1;
COMMIT;
```

Both dead tuple (`xmin=100,xmax=201`) and live tuple (`xmin=201`) sit in the same heap page simultaneously — this is literally what "dead tuple" means before vacuum reclaims it: correct-but-invisible-to-new-snapshots, not yet garbage-collectible until the *oldest* active snapshot anywhere in the cluster no longer needs it.

```sql
-- The oldest snapshot horizon vacuum must respect — a long-running txn here
-- is exactly what blocks cleanup (see Gotchas)
SELECT pid, xact_start, state, backend_xid, backend_xmin
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY xact_start;
```

### B-Tree Index Structure: Pointing Into the Heap, Not Clustering It

A Postgres B-tree index leaf entry stores an indexed key value plus a **CTID** (`(page, tuple offset)`) pointing into the heap — it never stores the row itself. This means Postgres's default table organization is a **heap table** (unordered, insert-order-ish, definitely not sorted by any index) with **secondary, non-clustering indexes only** — every index scan is, in the general case, index lookup → CTID → separate heap page fetch, i.e. two random I/Os instead of one.

```sql
CREATE INDEX accounts_balance_idx ON accounts (balance);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM accounts WHERE balance > 10000;
```

```
Index Scan using accounts_balance_idx on accounts
  (cost=0.29..245.16 rows=612 width=14) (actual time=0.031..1.203 rows=598 loops=1)
  Index Cond: (balance > 10000)
  Buffers: shared hit=601 read=14
```

Each of those 601+14 buffer touches includes both the index leaf page *and* the heap page the CTID points to — this is the concrete cost of a non-clustering index, and it's exactly what `CLUSTER` (a one-time, non-maintained physical reorder of the heap to match an index's order) and covering indexes (`INCLUDE`) exist to reduce:

```sql
-- One-time physical reorder of the heap to match the index's key order —
-- NOT maintained automatically on subsequent inserts/updates
CLUSTER accounts USING accounts_balance_idx;

-- Covering index: avoids the heap fetch entirely for queries that only need
-- the included columns (an index-only scan), IF the visibility map says the
-- page is all-visible (no need to double-check heap tuple visibility)
CREATE INDEX accounts_balance_covering_idx ON accounts (balance) INCLUDE (id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM accounts WHERE balance > 10000;
-- Index Only Scan using accounts_balance_covering_idx ... Heap Fetches: 0
```

`Heap Fetches: 0` in that last plan is the tell — no heap page was touched at all, because the visibility map (a bitmap tracking which heap pages have no dead tuples) confirmed every matching tuple is visible without checking `xmin`/`xmax` in the heap directly. `VACUUM` is what keeps the visibility map current, which is another reason vacuum lag directly degrades index-only-scan performance, not just heap bloat.

### Skip Scan (New in Postgres 18)

Multicolumn B-tree indexes historically required an equality/range predicate on the *leftmost* column to be useful (same leftmost-prefix rule as any B-tree/sparse-index structure — see `query-engines/clickhouse.md`'s sparse-index section for the analogous constraint). Postgres 18 added **skip scan**, letting the planner use a multicolumn index even when the query omits a predicate on a prefix column, by having the executor internally iterate distinct leading-column values:

```sql
CREATE INDEX idx_multi ON accounts (region, balance);

-- Pre-18: this ignored idx_multi's benefit for `region`, near-full index scan
-- 18+: skip scan iterates each distinct `region` value internally, still
-- using the index for the `balance` predicate within each region
EXPLAIN (ANALYZE)
SELECT * FROM accounts WHERE balance > 10000;
```

### WAL and Checkpointing

Every change is first described as a WAL record and fsynced before the transaction is durable; the actual heap/index pages are written back later, on a schedule controlled by the background writer and periodic **checkpoints**. A checkpoint flushes all dirty buffers as of a point in time and lets Postgres discard WAL segments older than that checkpoint (since replaying that WAL is no longer necessary for crash recovery past the checkpoint).

```sql
-- Key checkpoint tuning knobs
SHOW checkpoint_timeout;    -- default 5min: max time between checkpoints
SHOW max_wal_size;          -- default 1GB: soft cap that triggers an early checkpoint
SHOW checkpoint_completion_target;  -- default 0.9: spread checkpoint I/O over this
                                    -- fraction of checkpoint_timeout, to avoid I/O spikes

-- Force one manually (useful before a planned failover/backup)
CHECKPOINT;

-- See checkpoint activity and whether checkpoints are being forced too often
-- by max_wal_size (a forced/urgent checkpoint is a red flag — I/O spike risk)
SELECT * FROM pg_stat_checkpointer;
```

**The tradeoff:** a larger `max_wal_size` (or longer `checkpoint_timeout`) means fewer, less frequent checkpoints — less average I/O overhead — but a longer crash-recovery replay window (more WAL to replay before the database is consistent again) and a bigger WAL directory to retain. A smaller `max_wal_size` bounds recovery time and disk usage but risks frequent forced checkpoints (each one causing an I/O burst as dirty pages flush) if the write rate is high. `checkpoint_completion_target` close to 1.0 spreads that I/O burst thinner across the checkpoint interval, trading peak I/O for a longer sustained-but-lower rate.

### Autovacuum Tuning and the Bloat Failure Mode

```sql
-- Global defaults (per-table overridable via ALTER TABLE ... SET)
SHOW autovacuum_vacuum_scale_factor;   -- default 0.2: vacuum when dead tuples
                                        -- exceed 20% of table size (plus a base
                                        -- threshold, autovacuum_vacuum_threshold=50)
SHOW autovacuum_vacuum_cost_delay;     -- default 2ms: throttling between cost-
                                        -- limited work batches, to avoid autovacuum
                                        -- itself saturating I/O
SHOW autovacuum_max_workers;           -- default 3: concurrent autovacuum workers
                                        -- cluster-wide, runtime-adjustable as of PG18
                                        -- via autovacuum_worker_slots

-- PG18: absolute cap alongside the scale-factor percentage, so huge tables
-- don't wait for 20% of a billion rows to go dead before vacuum kicks in
SHOW autovacuum_vacuum_max_threshold;

-- Per-table override for a hot, high-churn table
ALTER TABLE accounts SET (autovacuum_vacuum_scale_factor = 0.02,
                           autovacuum_vacuum_cost_delay = 0);

-- Manual vacuum (blocking, foreground) when autovacuum has fallen behind
VACUUM (VERBOSE, ANALYZE) accounts;

-- The aggressive fix: also compacts and returns space to the OS, but takes
-- an ACCESS EXCLUSIVE lock for the duration — blocks reads and writes
VACUUM FULL accounts;
```

**The failure mode:** if the delete/update rate on a table sustainably exceeds what autovacuum can reclaim (too few workers, cost-delay throttling too conservative, or a scale factor too high for the table's size), `n_dead_tup` grows monotonically, the table and its indexes physically bloat (more pages than live data justifies), sequential scans and index scans both get slower (more pages to read for the same logical row count), and eventually the visibility map degrades too, hurting index-only scans as well. This is the practical, day-to-day consequence of "MVCC lives in the heap, not a separate undo log": the primary data file itself is where the operational risk accumulates.

```sql
-- Find the worst offenders: tables where dead tuples are a large fraction of live rows
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup::numeric / GREATEST(n_live_tup, 1), 3) AS dead_ratio,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC
LIMIT 10;
```

### Query Planner: Cost-Based, Seq Scan vs. Index Scan

The planner estimates cost in **arbitrary units anchored to disk I/O**, not wall-clock time: `seq_page_cost = 1.0` (baseline), `random_page_cost = 4.0` (default — random I/O costs 4× a sequential page fetch, an assumption tuned for spinning disks historically, often lowered to `1.1`–`2.0` on SSD-backed systems), plus `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost` for per-row/per-operator CPU work.

```sql
SHOW random_page_cost;      -- default 4.0
SHOW seq_page_cost;         -- default 1.0
SHOW cpu_tuple_cost;        -- default 0.01

-- A concrete crossover: same table, two selectivities
EXPLAIN SELECT * FROM accounts WHERE balance = 400;      -- highly selective
EXPLAIN SELECT * FROM accounts WHERE balance > 0;         -- matches ~everything
```

```
-- Highly selective predicate: planner picks the index
Index Scan using accounts_balance_idx on accounts
  (cost=0.29..8.31 rows=1 width=14)

-- Low-selectivity predicate on the same index: planner switches to seq scan
Seq Scan on accounts  (cost=0.00..1834.00 rows=99998 width=14)
  Filter: (balance > 0)
```

With `n=100,000` rows, `random_page_cost=4.0`, and roughly 100 rows/page: an index scan matching only 1 row costs approximately `random_page_cost` (fetch the one heap page) plus a few units for the B-tree traversal — cost `~8.3`. A seq scan matching ~99,998 of 100,000 rows must read every one of the table's ~1,000 pages sequentially (`1000 × seq_page_cost = 1000`) plus per-row CPU filter cost (`100,000 × cpu_tuple_cost(0.01) = 1000`) — cost `~1834`, still *cheaper per matched row* than doing ~99,998 individual random-I/O index+heap lookups at `4.0` each (`~400,000`+). This is the numeric shape of "the planner picks seq scan once selectivity crosses roughly a few percent of the table" — not a hardcoded threshold, but the natural crossover point where `rows_matched × (random_page_cost-equivalent per row)` exceeds `total_pages × seq_page_cost`.

**Stale statistics breaking this entirely:**

```sql
-- Simulate stats going stale: bulk-load then query before ANALYZE runs
INSERT INTO accounts SELECT generate_series(100001, 200000), 'checking', 100;
EXPLAIN SELECT * FROM accounts WHERE balance = 100;
-- planner still assumes the OLD row-count/histogram → may pick a bad plan,
-- e.g. an index scan the planner thinks matches 1 row but which actually
-- matches 100,000 newly-inserted ones

ANALYZE accounts;   -- refresh pg_statistic: row counts, MCVs, histograms
EXPLAIN SELECT * FROM accounts WHERE balance = 100;
-- now correctly estimates ~100,000 rows and switches to a seq scan
```

```sql
-- Confirm when stats were last refreshed and how stale they might be
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

---

## Comparative

### vs. LSM-Tree Storage Engines (see `data-structures/lsm-trees.md`)

Postgres's heap-plus-vacuum model and an LSM tree solve the same underlying problem — how to handle updates/deletes without constantly rewriting data in place — with structurally opposite tradeoffs. Postgres updates **in place, in the same file**, immediately reusable once vacuumed; an LSM tree never updates in place at all, appending new SSTables and relying on background **compaction** to merge away superseded versions (see that doc's write/read/space-amplification trilemma). The practical consequence: Postgres's write path is a single random-ish page write plus a WAL append (bounded write amplification, since a HOT update touches one page and doesn't necessarily touch every index), while an LSM's write path is purely sequential but pays compaction-driven write amplification later, off the critical path. Conversely, Postgres's *read* path is a single, direct heap/index lookup (once vacuumed and not bloated) with no multi-level merge; an LSM read must potentially check the memtable and multiple SSTable levels (mitigated by bloom filters), i.e. LSM optimizes the write side more aggressively and the read/space side less aggressively than Postgres's in-place model does. Both systems have an identical-shaped operational failure mode with different names: Postgres's "vacuum can't keep up → bloat" is functionally the same category of problem as an LSM's "compaction can't keep up → L0 file pileup and write stalls" — background reclamation debt, just against a heap-and-visibility-map instead of sorted immutable files.

### vs. etcd's MVCC (see `coordination/etcd.md`)

Both Postgres and etcd are MVCC systems that never let a reader see a torn/partial write, but they answer "what happens to old versions?" in opposite ways. etcd's MVCC layer (built on boltdb) **never overwrites a revision in place** — every `Put` creates a new keyspace revision, and old revisions remain fully queryable (`etcdctl get --rev=N`) until an operator explicitly runs `compact`, at which point history before that revision becomes permanently unavailable; the file-level space is only returned to the OS after a *separate* `defrag` step. Postgres's MVCC, by contrast, has no notion of "the history is deliberately kept until you say otherwise" — a dead tuple is a byproduct of an update or delete, not a retained feature, and it is reclaimed automatically and continuously by autovacuum the moment no active snapshot needs it, with the express goal of returning that heap space to the *same table* for reuse (not to the OS as free disk space, and not manually triggered the way etcd's `compact`+`defrag` is). In short: etcd treats old MVCC versions as a retained, queryable audit log with an explicit lifecycle you control (compact, then defrag); Postgres treats old MVCC versions as incidental garbage it wants gone as fast as safely possible, with vacuum as the automatic, continuous mechanism rather than an operator-invoked step. This is exactly the asymmetry `coordination/etcd.md` gestures at when it says etcd is "the exact inverse of Redis/Memcached's profile" while separately noting its MVCC differs from Postgres's — same acronym (MVCC), same underlying goal (snapshot isolation without locking readers out of writers), genuinely different space-reclamation contracts.

---

## Key Gotchas

- **Transaction ID wraparound**: Postgres transaction IDs (`xid`) are a 32-bit counter; once the gap between the oldest unfrozen `xid` in the cluster and the current counter approaches ~2 billion, Postgres would no longer be able to tell "past" from "future" transactions — it refuses new writes cluster-wide to prevent silent data-visibility corruption before this happens. Vacuum's `FREEZE` mechanism (marking old tuples as permanently visible, independent of any specific `xid`) is what prevents this in normal operation; it becomes a crisis specifically when regular vacuuming has been disabled or has been silently failing for a long time.
  ```sql
  -- Age in transactions since the oldest unfrozen xid — the early-warning metric
  SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age(datfrozenxid) DESC;
  -- default autovacuum_freeze_max_age = 200,000,000; approaching 2,000,000,000
  -- ("wraparound emergency" territory) means Postgres is close to refusing writes
  ```
- **A long-running transaction blocks vacuum from reclaiming anything newer than its snapshot** — even an idle-in-transaction session holding an old snapshot open (`BEGIN;` and forgetting to `COMMIT`/`ROLLBACK`, or a long analytical query) prevents autovacuum from removing tuples that became dead *after* that snapshot started, since that transaction could still theoretically read them. This is a very common real-world cause of sudden bloat on an otherwise well-vacuumed table.
  ```sql
  -- Find long-running/idle-in-transaction sessions holding back the vacuum horizon
  SELECT pid, state, now() - xact_start AS txn_age, query
  FROM pg_stat_activity
  WHERE xact_start IS NOT NULL
  ORDER BY xact_start ASC
  LIMIT 10;
  ```
- **Connection count limits and why PgBouncer matters**: each Postgres connection is a full OS process (`fork()`ed from postmaster), not a lightweight thread or coroutine — each one carries real memory overhead (`work_mem`-scale buffers per connection, process-level bookkeeping) and OS scheduling cost, so `max_connections` in the hundreds, not thousands, is typical. An application layer opening thousands of short-lived connections directly against Postgres will exhaust `max_connections` or force the OS to context-switch across far too many processes. PgBouncer (or similar poolers) sits in front and multiplexes many client connections onto a much smaller pool of actual Postgres backend processes.
  ```sql
  SHOW max_connections;                 -- typically 100-300 by default/tuned
  SELECT count(*) FROM pg_stat_activity;  -- current usage against that ceiling
  SELECT count(*), state FROM pg_stat_activity GROUP BY state;  -- idle vs active split
  ```
- **Index bloat requiring `REINDEX`**: B-tree indexes accumulate dead entries the same way heap pages do (an updated/deleted row's old index entry isn't removed until vacuum processes it), and unlike the heap, index page splits from bloat don't always get fully reclaimed even after vacuum runs, especially under high churn. Persistent index bloat shows up as index scans reading far more pages than the live row count justifies.
  ```sql
  -- Rebuild without a long exclusive lock (Postgres 12+)
  REINDEX INDEX CONCURRENTLY accounts_balance_idx;

  -- Rough bloat signal: compare index size to a fresh estimate for the same data
  SELECT relname, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
  FROM pg_stat_user_indexes
  JOIN pg_index USING (indexrelid)
  WHERE relname = 'accounts_balance_idx';
  ```
- **Stale statistics silently producing bad plans**: bulk loads, large deletes, or any workload that changes data distribution faster than `autovacuum_analyze_scale_factor` (default 0.1 — 10% of rows changed) triggers autoanalyze will leave the planner working off outdated row-count/histogram estimates, leading to a seq-scan-vs-index-scan misjudgment like the one shown in Deep Internals above. This is especially sharp right after a large `COPY`/bulk `INSERT` into an empty or small table, since the *relative* change is enormous.
  ```sql
  ANALYZE accounts;                 -- manual refresh, cheap relative to VACUUM
  SELECT last_analyze, last_autoanalyze, n_mod_since_analyze
  FROM pg_stat_user_tables WHERE relname = 'accounts';
  ```

---

*Grounded against postgresql.org release notes and community sources (Xata, CYBERTEC, pganalyze, Better Stack) as of September 2026. PostgreSQL 18 GA'd 2025-09-25, current minor as of this writing is 18.6 (2026-08-13); the AIO subsystem, autovacuum threshold/worker-count changes, skip scan, and `uuidv7()` are 18-specific and worth re-confirming against the exact deployed version before relying on them operationally. `random_page_cost` default (4.0) is a spinning-disk-era assumption many SSD-backed production clusters override to 1.1-2.0 — re-verify the configured value rather than assuming the default when reading a real EXPLAIN plan.*
