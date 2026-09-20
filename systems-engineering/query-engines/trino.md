# Trino

## 30-Second Intuition

Trino (forked from PrestoSQL in 2020, renamed for trademark reasons — same project, same people) is a distributed SQL query engine that separates the *query engine* from the *storage layer* entirely: it does not own a storage format the way ClickHouse owns MergeTree parts or DuckDB owns its embedded file. Instead it plugs into dozens of heterogeneous sources — Iceberg tables on S3, a live Postgres OLTP database, a Kafka topic, a MongoDB collection — through a pluggable **connector** SPI, and lets you `JOIN` across all of them in one SQL statement as if they were tables in the same database. The one fact that matters operationally: Trino's execution engine is **fully in-memory and pipelined by default, with no durable checkpoint of intermediate query state** — if a worker dies mid-query, the whole query (or, if you've explicitly opted into fault-tolerant execution, the affected task) restarts from scratch, unlike Spark, which spills shuffle data to disk and recomputes lost partitions from lineage. Trino trades resilience for latency: it's built to answer an interactive query in seconds, not to survive an hours-long batch job by construction.

---

## Resource-Layer Map

| Layer | Role Trino plays | What it's optimizing for |
|---|---|---|
| CPU | Vectorized, columnar operators (largely Java, JIT-compiled) execute filter/project/aggregate directly on in-memory pages | Push as much work as possible into tight per-page loops on the coordinator-scheduled worker threads; CPU-bound once data is resident in memory and has already been read from the source |
| Memory | Every operator's intermediate state (hash tables for joins/aggregations, partial sort buffers) lives in JVM heap on the worker; `query.max-memory-per-node` / `query.max-memory` cap it hard | Keep the whole query's working set in memory for lowest latency; memory is the resource Trino is least willing to trade away — spilling to disk is opt-in and only for a few operator types, not the default posture |
| Disk | Not used for intermediate query state by default at all; only touched for the source scan (whatever the connector's underlying storage is) and, opt-in, for spill-to-disk sort/aggregation or for Fault-Tolerant Execution's exchange spooling | Disk is deliberately absent from the standard execution path — this is what "in-memory, no durability" means concretely: no shuffle files, no write-ahead log, no checkpoint directory unless you turn one on |
| Network | The **exchange** — the operator that redistributes rows between pipeline stages (map-side to reduce-side, build-side broadcast to probe workers) — moves data over the cluster network for every partitioned/broadcast join, `GROUP BY`, or `ORDER BY`; this is Trino's equivalent of Spark's shuffle | Minimize round-trips and bytes moved per stage boundary; because there's no disk buffer between stages by default, network throughput and worker availability during the exchange directly gate query completion — a network hiccup mid-exchange isn't recoverable without FTE |
| GPU | Not part of the core engine | Trino's execution model targets JVM-resident columnar pages processed by CPU threads; no native GPU execution path exists |

The sharpest contrast is with Spark (`compute/spark.md`): Spark's resource-layer table calls disk-backed shuffle-with-lineage-recompute "the fallback, not the default" but still a first-class, always-available mechanism — a lost Spark shuffle partition is *always* recoverable by recomputing from lineage, because the shuffle files exist on disk regardless of whether the job configured any special resiliency. Trino's default posture has no equivalent: intermediate exchange data lives only in the sending worker's memory/local buffer, and if that worker or the exchange itself fails, the query simply fails, unless you explicitly configure Fault-Tolerant Execution (see Deep Internals) to spool exchange data externally. This is a deliberate trade: Trino is built to be *fast* for interactive federated queries, not resilient by construction for long batch jobs — that resilience is available, but it's an opt-in mode change, not the baseline architecture.

---

## The Signature Mechanism: The Connector SPI (Query Federation)

Trino's defining architectural choice is that the query engine has **zero built-in knowledge of storage**. Every table Trino can see — an Iceberg table, a Postgres table, a Kafka topic, an in-memory `system` table — is exposed to the planner through the same `Connector` SPI: a Java interface a plugin implements to answer "what schemas/tables exist," "what are this table's statistics," "generate splits for this scan," and (optionally) "can you push this filter/aggregate/limit down to your own storage." The coordinator's planner treats every connector identically at the SQL layer — a `JOIN` between an Iceberg connector table and a Postgres connector table is planned exactly like a `JOIN` between two Iceberg tables, with the only difference being which pushdown capabilities each connector's `ConnectorMetadata` reports back.

This is fundamentally different from Spark or DuckDB, which assume a small set of native storage formats (Parquet/Delta/etc for Spark's DataFrame reader, DuckDB's own file + a handful of scan functions) with everything else bolted on as a data-source plugin around that native core. Trino was built federation-first: a `catalog` is just a named instantiation of a connector plus its connection config, and a query can reference tables across an arbitrary number of catalogs in one statement.

```sql
-- Two catalogs, two totally different systems, one SQL statement:
-- `iceberg` catalog -> S3 + Iceberg REST catalog
-- `pg`      catalog -> live Postgres OLTP database

SELECT o.order_id, o.total_amount, c.email, c.signup_country
FROM iceberg.sales.orders o
JOIN pg.public.customers c
  ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE '2026-01-01'
  AND c.signup_country = 'DE';
```

Configuring those two catalogs is two flat property files, not code:

```properties
# etc/catalog/iceberg.properties
connector.name=iceberg
iceberg.catalog.type=REST
iceberg.rest-catalog.uri=https://catalog.example.com/api
hive.s3.aws-access-key=...
hive.s3.aws-secret-key=...

# etc/catalog/pg.properties
connector.name=postgresql
connection-url=jdbc:postgresql://pg-primary.internal:5432/appdb
connection-user=trino_ro
connection-password=...
```

`SHOW CATALOGS` and `SHOW SCHEMAS FROM <catalog>` are how you discover what's actually plugged in at runtime:

```sql
SHOW CATALOGS;
--  Catalog
-- ---------
--  iceberg
--  pg
--  system

SHOW SCHEMAS FROM iceberg;
SHOW TABLES FROM pg.public;
```

Roughly 50 connectors ship with Trino as of the 483 release line (mid-2026), including Iceberg, Hive, Delta Lake, Hudi, and the new unified **Lakehouse** connector (auto-detects table format per-table across Iceberg/Delta/Hudi under one catalog) for lake formats; PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, Redshift, Snowflake, SingleStore, ClickHouse, DuckDB for JDBC-style relational sources; MongoDB, Cassandra for NoSQL; Kafka, Pinot, Druid for streaming/real-time OLAP; Elasticsearch, OpenSearch for search indices; BigQuery, plus a `system`, `memory`, `blackhole`, and `faker` connector for testing. Each connector independently decides how much of the SQL it can push down — this is the load-bearing asymmetry of the federation model (see Gotchas).

---

## High-to-Low Walkthrough: A Federated `SELECT` From CLI to Exchange Bytes

Trace this literal federated query — an Iceberg fact table joined to a live Postgres dimension table — from the CLI down to the exchange.

```bash
# Connect the Trino CLI to a running coordinator
trino --server https://trino-coordinator.internal:8443 --catalog iceberg --schema sales
```

```sql
SELECT c.signup_country, sum(o.total_amount) AS revenue
FROM iceberg.sales.orders o
JOIN pg.public.customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE '2026-01-01'
GROUP BY c.signup_country
ORDER BY revenue DESC;
```

```
Client (trino CLI / JDBC / ODBC driver)
      │  POST /v1/statement  (SQL text over HTTP)
      ▼
Coordinator: Parser -> AST
      │
      ▼
Coordinator: Analyzer         resolves iceberg.sales.orders and pg.public.customers
                               against each connector's ConnectorMetadata; checks
                               access control per catalog/schema/table/column
      │
      ▼
Coordinator: Logical planner   builds a connector-agnostic logical plan (Scan, Filter,
                               Join, Aggregate) then asks each connector's metadata:
                               "can you push this predicate/projection/aggregation
                               down into your own scan?" -- Iceberg says yes to the
                               orderdate filter (partition + file-stats pruning);
                               Postgres connector may or may not push the whole
                               table depending on configuration (see Gotchas)
      │
      ▼
Coordinator: Cost-based        uses per-connector table statistics (row counts,
optimizer                      NDVs, min/max) to choose join order and join
                               distribution type (broadcast vs. partitioned) --
                               see Deep Internals
      │
      ▼
Coordinator: Split generation   Iceberg connector enumerates splits from the table's
                                current snapshot's manifest files (one split per
                                data file or file range); Postgres connector
                                generates splits from its own row-range/partition
                                logic (often just "one split" for a small table)
      │
      ▼
Coordinator: Stage/Fragment     the plan is cut into FRAGMENTS at each exchange
assignment                      boundary; each fragment becomes a STAGE with one
                                 or more TASKS distributed across workers
      │
      ▼
Workers: Task execution         each worker pulls its assigned splits, and for the
(parallel, per stage)            Iceberg side reads Parquet/ORC row groups directly
                                 from S3 (see data-formats/apache-iceberg.md); for
                                 the Postgres side, workers open JDBC connections
                                 and stream rows over the network from the live
                                 database
      │
      ▼
EXCHANGE (network)               build-side (customers, filtered/joined smaller
                                 side) is redistributed -- broadcast to every
                                 probe-side worker if small, or hash-partitioned
                                 by join key if large -- entirely in memory,
                                 no disk write, unless FTE is on
      │
      ▼
Workers: Join + partial          each worker builds/probes a hash table locally,
aggregate                        computes a PARTIAL sum(total_amount) per
                                 signup_country on its slice of the data
      │
      ▼
EXCHANGE (network)               partial aggregates are hash-redistributed by
                                 signup_country to the final-aggregation stage
      │
      ▼
Coordinator: FINAL aggregate      merges partial sums into final per-country
+ ORDER BY                        totals, sorts by revenue DESC
      │
      ▼
Client receives result           streamed back over the same HTTP connection
                                  the CLI/driver opened
```

Ask Trino to show its actual plan before running anything, with `EXPLAIN`:

```sql
EXPLAIN
SELECT c.signup_country, sum(o.total_amount) AS revenue
FROM iceberg.sales.orders o
JOIN pg.public.customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE '2026-01-01'
GROUP BY c.signup_country
ORDER BY revenue DESC;
```

```
Fragment 0 [SINGLE]
    Output layout: [signup_country, revenue]
    Output partitioning: SINGLE []
    - Output[columnNames = [signup_country, revenue]]
        - Sort[orderBy = [revenue DESC]]
            - RemoteSource[sourceFragmentIds = [1]]

Fragment 1 [HASH]
    Output layout: [signup_country, sum]
    Output partitioning: SINGLE []
    - Aggregate[type = FINAL, keys = [signup_country]]
        - LocalExchange[partitioning = HASH]
            - RemoteSource[sourceFragmentIds = [2]]

Fragment 2 [HASH]
    Output layout: [signup_country, partial_sum]
    Output partitioning: HASH [signup_country]
    - Aggregate[type = PARTIAL, keys = [signup_country]]
        - InnerJoin[criteria = ("customer_id" = "customer_id"), distribution = PARTITIONED]
            - RemoteSource[sourceFragmentIds = [3]]     -- orders side (Iceberg)
            - LocalExchange[partitioning = HASH]
                - RemoteSource[sourceFragmentIds = [4]] -- customers side (Postgres)

Fragment 3 [SOURCE]
    - ScanFilterProject[table = iceberg:sales:orders,
        filterPredicate = ("order_date" >= DATE '2026-01-01')]

Fragment 4 [SOURCE]
    - TableScan[table = pg:public:customers]
```

Fragment 3 and Fragment 4 are the two connectors' scans — Iceberg's file-level `order_date` predicate is visibly pushed into the `ScanFilterProject` node (Trino requested it and Iceberg's connector honored it via partition/file pruning), while the Postgres side here shows a plain `TableScan` because this Postgres connector configuration didn't push the join filter down — the whole table streams over JDBC and gets filtered/joined on the Trino side instead. That asymmetry is visible directly in the plan, which is exactly the point of reading `EXPLAIN` for a federated query.

Now run it for real numbers with `EXPLAIN ANALYZE`, which adds live per-fragment timing and row counts:

```sql
EXPLAIN ANALYZE
SELECT c.signup_country, sum(o.total_amount) AS revenue
FROM iceberg.sales.orders o
JOIN pg.public.customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE '2026-01-01'
GROUP BY c.signup_country
ORDER BY revenue DESC;
```

```
Trino version: 483
Queued: 1.20ms, Analysis: 84.31ms, Planning: 142.05ms, Execution: 6.41s

Fragment 2 [HASH]
    CPU: 3.91s, Scheduled: 4.02s, Blocked 1.88s (Input: 1.80s, Output: 0.00ns)
    Input: 42,118,204 rows (1.61GB); per task avg.: 5,264,775.50, std.dev.: 4.10%
    Output: 187 rows (6.02kB)
    Aggregate[type = PARTIAL, keys = [signup_country]]
    └─ InnerJoin[criteria = ("customer_id" = "customer_id"), distribution = PARTITIONED]
       │   Left (build): 812,004 rows, Right (probe): 42,118,204 rows
       │   Dynamic filter collected on customer_id, applied to probe-side scan
       ├─ RemoteSource[sourceFragmentIds = [3]]        -- Iceberg orders, post-pushdown
       └─ RemoteSource[sourceFragmentIds = [4]]        -- Postgres customers

Fragment 3 [SOURCE]
    CPU: 2.65s, Scheduled: 2.71s, Blocked 0.00ns
    Input: 96,004,511 rows (3.4GB) scanned; 53,886,307 rows (1.9GB) after dynamic filter pushdown
    ScanFilterProject[table = iceberg:sales:orders,
        filterPredicate = ("order_date" >= DATE '2026-01-01')
                       AND ("customer_id" IN <dynamic filter from Fragment 2>)]

Fragment 4 [SOURCE]
    CPU: 410.22ms, Scheduled: 890.14ms, Blocked 0.00ns
    Input: 812,004 rows (28.4MB) via JDBC from pg:public:customers
```

Two things this real output makes visible that `EXPLAIN` alone doesn't: (1) `Blocked` time on Fragment 2's join — the probe side is waiting on network/exchange input, which is the network row of the resource-layer table made concrete; (2) the dynamic filter collected from the small Postgres `customers` build side (812K rows) got pushed all the way back into Fragment 3's Iceberg scan, cutting the Iceberg-side row count from 96M scanned down to 53.9M read post-filter — this is dynamic filtering in action (see Deep Internals).

---

## Deep Internals

### Cost-Based Optimizer and Table Statistics

Trino's CBO chooses join order and join distribution strategy (`BROADCAST` vs. `PARTITIONED`) using per-connector table statistics — row counts, NDV (number of distinct values) estimates, min/max, and null fractions. Statistics support is connector-specific; if a connector reports none, the CBO falls back to the `ELIMINATE_CROSS_JOINS` strategy rather than true cost-based reordering.

```sql
-- Inspect what statistics a connector actually has for a table
SHOW STATS FOR iceberg.sales.orders;
--  column_name  | data_size_bytes |  distinct_values_count |  nulls_fraction |  row_count
-- --------------+-----------------+-------------------------+-----------------+-----------
--  order_id     |            NULL |               41000000  |             0.0 |       NULL
--  customer_id  |            NULL |                 800000  |             0.0 |       NULL
--  total_amount |            NULL |                    NULL |             0.0 |       NULL
--  NULL         |            NULL |                    NULL |            NULL | 42000000

-- Force Iceberg to (re)compute/refresh column-level statistics used by the CBO
ANALYZE iceberg.sales.orders;

-- Control the join reordering strategy explicitly
SET SESSION join_reordering_strategy = 'AUTOMATIC';   -- default: use stats when available
SET SESSION join_distribution_type = 'AUTOMATIC';     -- let CBO pick broadcast vs partitioned
```

Broadcast is chosen when one side is small enough to fit in `join-max-broadcast-table-size` (default 100MB) after filtering — replicating the whole build side to every worker avoids a hash-partitioning exchange on the (larger) probe side entirely, at the cost of sending a full copy of the small side to every node:

```sql
-- Force a specific distribution type for testing/tuning
SET SESSION join_distribution_type = 'BROADCAST';
SET SESSION join_distribution_type = 'PARTITIONED';
```

Statistics quality varies sharply by connector: Iceberg/Hive can have real file-level and (if `ANALYZE`d) column-level stats; a Postgres connector's stats depend on whether the connector is configured to read the source database's own `pg_stats`; a Kafka connector generally has none. A federated query joining a well-analyzed Iceberg table to an un-analyzed JDBC table is exactly the case where the CBO can silently pick the wrong join order or distribution — always check `EXPLAIN` rather than assuming the CBO got it right.

### Fault-Tolerant Execution (FTE)

FTE is Trino's answer to "the resource-layer map says intermediate query state isn't durable — how do you survive a worker crash on a long query?" It is **disabled by default**; you opt in via `retry-policy`:

```properties
# etc/config.properties on the coordinator
retry-policy=TASK
exchange.compression-enabled=true

# Required exchange manager config (etc/exchange-manager.properties) --
# TASK policy needs external spooling for exchange data; QUERY policy
# can fall back to the coordinator's in-memory result buffer for smaller results
exchange-manager.name=filesystem
exchange.base-directories=s3://my-trino-exchange-bucket/spool
exchange.s3.region=us-east-1
```

Two retry policies, deliberately different granularity:

| Policy | Retries at | Requires exchange manager | Best for |
|---|---|---|---|
| `QUERY` | Whole query, from the start | No (in-memory buffer works, external storage recommended for large results) | Many small/medium interactive queries where a full restart is cheap |
| `TASK` | Individual failed tasks within a query | Yes — mandatory | Long-running batch queries where restarting from scratch would be far more expensive than retrying one task |

```sql
-- Per-session override of the cluster-wide default, for one query
SET SESSION retry_policy = 'TASK';

SELECT /* long batch aggregation over years of Iceberg data */
    date_trunc('month', order_date) AS month, sum(total_amount)
FROM iceberg.sales.orders
GROUP BY 1;
```

The precise nuance worth stating plainly: FTE does not contradict "Trino has no durability by default" — it *is* the explicit, opt-in exception. With `TASK` policy and a configured exchange manager, exchange data is spooled to external storage (S3, GCS, Azure Blob, HDFS, or local disk for dev/test) instead of living only in each worker's memory, so a failed downstream task can be retried by re-reading already-spooled upstream output rather than recomputing the whole upstream fragment. This is architecturally closer to what Spark does unconditionally (materialize shuffle output, retry from that materialized state) — Trino just makes it a deliberate, cluster-operator-configured trade-off (extra network + storage cost, higher latency per query) rather than the baseline. FTE support is also connector-dependent: not every connector supports resumable writes under task retry, so check per-connector FTE support before relying on it for writes, not just reads.

### The Connector SPI Model

A connector plugs into four main SPI surfaces:

- `ConnectorMetadata` — schema/table discovery, statistics, and pushdown capability negotiation (can this connector accept a predicate, a projection, a `LIMIT`, an aggregation, pushed into its own scan?).
- `ConnectorSplitManager` — turns a table (plus any pushed-down predicate) into a list of `ConnectorSplit`s, the unit of work handed to a single task.
- `ConnectorPageSourceProvider` / `ConnectorPageSink` — reads/writes actual `Page`s (Trino's in-memory columnar batch unit) for a given split.
- `ConnectorAccessControl` — per-connector authorization hooks.

```sql
-- See what pushdown/capabilities a specific catalog's connector actually reports
-- (visible indirectly via EXPLAIN's ScanFilterProject/TableScan node shape --
-- a bare TableScan vs. one with filterPredicate/projections baked in)
EXPLAIN SELECT customer_id FROM pg.public.customers WHERE signup_country = 'DE';
```

Any connector, however exotic the underlying source, only needs to implement enough of this SPI to answer "list my tables" and "give me pages of rows" to work in Trino at all — pushdown is an optimization layer on top, not a requirement, which is why some connectors (e.g. `faker`, generic JDBC connectors without cost-based extensions) are functionally correct but leave far more filtering work to Trino's own operators than a fully pushdown-capable connector like Iceberg's.

### Dynamic Filtering

Dynamic filtering is runtime, join-derived predicate pushdown: while a join's build side is being processed, Trino collects the actual set (or min/max range) of join-key values seen, and pushes that as a runtime filter into the probe side's table scan — before the probe side has even finished reading, for connectors that support split-level pruning. This is enabled by default:

```sql
-- Enabled by default; can be disabled per-session for comparison/debugging
SET SESSION enable_dynamic_filtering = false;
```

```sql
-- The exact pattern that benefits most: a small, highly selective dimension
-- table filtering a much larger fact table
SELECT o.order_id, o.total_amount
FROM iceberg.sales.orders o
JOIN pg.public.customers c ON o.customer_id = c.customer_id
WHERE c.signup_country = 'DE'    -- filters customers down to a small build side
  AND c.is_active = true;
```

For broadcast joins, the collected dynamic filter is applied locally on each worker doing the probe-side scan (same-node, no network cost to distribute it). For partitioned joins, the coordinator distributes the collected filter to worker nodes over the network once collection completes on the build side, which is itself an added coordination round-trip — dynamic filtering trades a small delay in starting the probe-side scan for potentially reading and transferring far fewer rows. This was the concrete mechanism visible in the walkthrough's `EXPLAIN ANALYZE` output above: the Iceberg scan's "53,886,307 rows after dynamic filter pushdown" vs. "96,004,511 rows scanned" is dynamic filtering doing exactly this.

---

## Comparative: vs. Spark and vs. DuckDB

**vs. Spark** (`compute/spark.md`): Spark's shuffle writes to local disk at every stage boundary and can recompute lost partitions from lineage — durability is unconditional, cost is unconditional too (every wide transformation pays a disk write). Trino's exchange is in-memory by default with no disk write and no recompute-from-lineage concept at all; a worker failure mid-exchange fails the query outright unless FTE is explicitly configured, in which case Trino approximates (but doesn't fully replicate) Spark's model by spooling exchange data externally. The practical rule: Trino for interactive, federated, sub-minute queries where a rare full-query retry is an acceptable cost; Spark (or Trino with FTE enabled) for long batch jobs where you need resilience against partial failures as a first-class guarantee, not an add-on.

**vs. DuckDB** (`query-engines/duckdb.md`): DuckDB has no exchange/shuffle concept at all, because it has no distributed workers to exchange data between — it's a single embedded process using shared-memory morsel-driven parallelism across threads. Trino's entire distributed architecture (coordinator, workers, fragments, exchanges) exists specifically because Trino assumes data and compute live on separate machines (often separate from the client entirely) and frequently in separate, heterogeneous *systems* — DuckDB assumes one process, one machine, and (mostly) one connector-like scan function reading files it can access directly. Trino's connector federation (join Postgres to Iceberg to Kafka in one query) has no DuckDB equivalent beyond DuckDB's own narrower set of scan extensions (`postgres_scanner`, `iceberg`, `delta`), each of which is a read/write integration into DuckDB's single-process engine, not a live distributed query participant the way a Trino connector is.

---

## Key Gotchas

- **Memory-per-query limits and the OOM killer**: `query.max-memory-per-node` caps user memory (hash tables, sort buffers) a single query can use on one worker; `query.max-memory` caps it cluster-wide. Hitting either kills the query with an `EXCEEDED_LOCAL_MEMORY_LIMIT`/`EXCEEDED_GLOBAL_MEMORY_LIMIT` error — this is Trino's own killer, not the OS OOM killer, and it exists specifically so one runaway query doesn't take down a worker's JVM. The docs are explicit that `query.max-memory-per-node` plus `memory.heap-headroom-per-node` must stay below the JVM's max heap — misconfigure that gap and you *do* risk the actual OS OOM killer terminating the JVM process instead of Trino's graceful query-kill path.
  ```properties
  query.max-memory-per-node=4GB
  memory.heap-headroom-per-node=2GB
  # JVM -Xmx must exceed the sum of the two above with real headroom
  ```
- **Connector-specific pushdown limitations**: a `JOIN`/`WHERE`/aggregation that gets fully pushed into an Iceberg or Hive scan (file/partition pruning, column pruning) may not get pushed at all into a generic JDBC connector's query, depending on how much of `ConnectorMetadata`'s pushdown surface that specific connector implements. Always check `EXPLAIN` for a bare `TableScan` (no pushdown happened, filtering happens in Trino) vs. a `ScanFilterProject`/pushdown-annotated node before assuming a `WHERE` clause is cheap on a federated join.
- **One slow connector stalls the whole federated query**: because a join's probe and build sides across two catalogs execute concurrently but the join itself can't complete until both sides deliver their data, a slow, unindexed, or overloaded source (a Postgres table without the right index, a Kafka topic under replication lag) becomes the query's critical path regardless of how fast the Iceberg/S3 side is. Federation doesn't parallelize away a genuinely slow source — it just makes the slow source's latency visible as "Trino is slow" rather than "Postgres is slow."
- **The coordinator is a real (if HA-capable) bottleneck**: a single coordinator does all parsing, analysis, planning, split generation, and stage/task scheduling for every query in the cluster; heavy planning load (many concurrent large or highly federated queries, especially with many small splits) or a busy coordinator JVM (GC pauses) directly delays every query's start, not just one. Trino supports coordinator HA/failover for availability, but that's redundancy for failover, not horizontal scale-out of the planning role itself — one coordinator is doing the planning work for the whole cluster at any given moment.
- **`ANALYZE` is not automatic**: unlike some warehouses that auto-refresh statistics, Trino connectors that support `ANALYZE` (Iceberg, Hive) only have accurate CBO stats if someone runs it after significant data changes; stale stats after a large backfill can cause the CBO to pick a bad join order or distribution type silently — verify with `SHOW STATS FOR <table>` and `EXPLAIN`, don't assume.
- **`FTE`'s `TASK` policy adds real latency for small/fast queries**: because it requires spooling exchange data externally (network + object storage round-trip) rather than passing it in-memory between workers, enabling `TASK` retry policy cluster-wide for a workload dominated by fast interactive queries will measurably slow every one of them down — it's meant for and should be scoped to long batch workloads, typically via per-session `retry_policy` overrides rather than a blanket cluster default.

---

*Grounded against trino.io official documentation (fault-tolerant execution, dynamic filtering, cost-based optimizations, EXPLAIN ANALYZE, connector list, resource-management properties) and the Trino 482/483 release blog posts as of September 2026. Current stable release: Trino 483 (July 2026); the "Lakehouse" connector (unified Iceberg/Delta/Hudi table-format auto-detection under one catalog) and PIVOT/OVERLAPS SQL support are recent (2026) additions. FTE has no formally announced "GA" milestone in current docs — it is a mature, documented, opt-in feature with per-connector support caveats rather than an experimental flag, but "which connectors fully support FTE for writes" is a fast-moving list worth re-checking against the current docs before relying on it in a specific deployment. EXPLAIN/EXPLAIN ANALYZE output in this doc follows the documented format and the official example's real structure; specific row counts/timings for the federated Iceberg+Postgres walkthrough are illustrative, not captured from one real run.*
