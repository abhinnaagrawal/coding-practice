# Apache Spark (Core Engine)

## 30-Second Intuition

Spark is a distributed compute engine built around one idea: describe your computation as a lazy DAG of transformations, let an optimizer (Catalyst) rewrite that DAG into an efficient physical plan, then execute it with as much in-memory reuse as possible instead of materializing every intermediate stage to disk the way MapReduce did (see `storage/hadoop.md` for that comparison). The one fact that matters operationally: **nothing actually runs until an action is called** — `df.filter(...).groupBy(...).agg(...)` builds a plan, and only `.collect()`/`.write()`/etc. triggers Catalyst to optimize the *whole* chain at once and execute it. This laziness is what lets Catalyst do things a row-at-a-time imperative engine can't: reorder filters before joins, push projections down to the scan, and fuse multiple logical steps into one physical stage — none of which is possible if each line executes immediately in isolation.

---

## Resource-Layer Map

| Layer | Role Spark plays | What it's optimizing for |
|---|---|---|
| CPU | Whole-stage code generation compiles a chain of operators into one JVM bytecode function per stage | Eliminate virtual-function-call overhead between operators (the classic "Volcano iterator model" cost) by fusing operators at compile time instead of interpreting a plan tree row-by-row at runtime |
| Memory | Tungsten manages a large fraction of execution memory off-heap, in a binary row format, bypassing JVM object overhead and GC | Avoid JVM object header/boxing overhead and GC pause time for the actual data being processed — Spark treats memory as a resource it manages directly, not something it hands entirely to the JVM garbage collector |
| Disk | Shuffle files are the main disk write Spark does deliberately; otherwise Spark keeps intermediate results in memory across stages when it fits | Disk is the fallback, not the default — this is the direct architectural response to MapReduce's mandatory disk write after every map and every reduce stage |
| Network | Shuffle is the dominant network cost: every wide transformation (`groupBy`, `join`, `repartition`) redistributes data across the cluster | Adaptive Query Execution (AQE) exists specifically to reduce unnecessary shuffle-network cost at runtime, once real data statistics are known instead of only the query-planning-time estimates |
| GPU | Not part of the core engine's design, though pluggable RAPIDS-style GPU acceleration exists as an add-on | Core Spark's execution model (JVM-based row/columnar batches) isn't GPU-native; GPU acceleration is bolted on for specific operators, not a first-class execution target the way CPU/memory are |

The sharpest resource-layer contrast is with Trino/DuckDB (see `query-engines/duckdb.md`): those engines assume the working set mostly fits (or streams through) memory for a single query and don't durably materialize shuffle state to disk the way Spark's shuffle does — Spark's willingness to spill shuffle data to disk (and recompute lost partitions from lineage instead of re-running the whole job) is what makes it viable for jobs whose intermediate data is far larger than cluster memory, at the cost of disk I/O that a memory-resident query engine doesn't pay.

---

## The Signature Mechanism: Lazy DAG + Catalyst Optimizer + Whole-Stage Codegen

Three things work together, but the one that makes the other two possible is **laziness**: every DataFrame transformation (`filter`, `select`, `groupBy`, `join`) just appends a node to a logical plan tree — no data moves. Only an action (`collect`, `write`, `count`) triggers execution, and at that point Catalyst has the *entire* chain of transformations visible at once, not just one step in isolation.

Literal, runnable version of that chain:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum as _sum

spark = SparkSession.builder.appName("catalyst-demo").getOrCreate()

orders = spark.read.parquet("s3://bucket/orders/")   # user_id, status, amount, region
users  = spark.read.parquet("s3://bucket/users/")    # user_id, name, signup_date, ...

result = (
    orders.filter(col("status") == "active")
          .join(users, "user_id")
          .groupBy("region")
          .agg(_sum("amount").alias("total_amount"))
)
# Nothing has executed yet -- `result` is still just a logical plan (an
# unresolved tree of Filter -> Join -> Aggregate nodes). No file has been
# read, no join has happened.

result.write.mode("overwrite").parquet("s3://bucket/region_totals/")  # <- ACTION, execution starts here
```

Catalyst sees the WHOLE chain above the moment the action fires and can:
1. **Predicate pushdown**: push `filter(status == "active")` down to before the join (and even into the Parquet scan itself, via file-level stats — see `data-formats/parquet.md`) — filtering 1,000 rows before a join is much cheaper than joining then filtering 1,000,000.
2. **Reorder** join vs. groupBy if it changes nothing semantically but reduces the amount of data flowing through the expensive join.
3. **Column pruning**: if only `region` and `amount` are ever read downstream, never materialize other columns from the scan at all — notice in the plan below that the `users` side is pruned down to just `user_id`, the join key, even though the source file has many more columns.

Calling `result.explain(True)` (or `result.explain("extended")` — same output) prints all four Catalyst phases for that exact chain:

```
>>> result.explain(True)
== Parsed Logical Plan ==
'Aggregate ['region], ['region, 'sum('amount) AS total_amount#12]
+- 'Join Inner, ('user_id = 'user_id)
   :- 'Filter ('status = active)
   :  +- 'UnresolvedRelation [orders]
   +- 'UnresolvedRelation [users]

== Analyzed Logical Plan ==
region: string, total_amount: double
Aggregate [region#3], [region#3, sum(amount#5) AS total_amount#12]
+- Join Inner, (user_id#1 = user_id#7)
   :- Filter (status#4 = active)
   :  +- Relation [user_id#1,status#4,amount#5,region#3] parquet
   +- Relation [user_id#7,name#8,signup_date#9] parquet

== Optimized Logical Plan ==
Aggregate [region#3], [region#3, sum(amount#5) AS total_amount#12]
+- Project [region#3, amount#5, user_id#1]
   +- Join Inner, (user_id#1 = user_id#7)
      :- Project [user_id#1, region#3, amount#5]                     -- column pruning: status dropped after the filter consumes it
      :  +- Filter (isnotnull(status#4) AND (status#4 = active))     -- pushed below the join
      :     +- Relation [user_id#1,status#4,amount#5,region#3] parquet
      +- Project [user_id#7]                                          -- users side pruned to ONLY the join key
         +- Relation [user_id#7,name#8,signup_date#9] parquet

== Physical Plan ==
*(3) HashAggregate(keys=[region#3], functions=[sum(amount#5)], output=[region#3, total_amount#12])
+- Exchange hashpartitioning(region#3, 200), ENSURE_REQUIREMENTS, [id=#42]
   +- *(2) HashAggregate(keys=[region#3], functions=[partial_sum(amount#5)], output=[region#3, sum#20])   -- MAP-SIDE partial aggregate, see High-to-Low Walkthrough
      +- *(2) Project [region#3, amount#5]
         +- *(2) BroadcastHashJoin [user_id#1], [user_id#7], Inner, BuildRight    -- users side small enough to broadcast, no shuffle for the join itself
            :- *(2) Filter (isnotnull(status#4) AND (status#4 = active))
            :  +- *(2) ColumnarToRow
            :     +- FileScan parquet orders[user_id#1,status#4,amount#5,region#3] Batched: true, PushedFilters: [IsNotNull(status), EqualTo(status,active)]
            +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false])), [id=#38]
               +- *(1) ColumnarToRow
                  +- FileScan parquet users[user_id#7] Batched: true
```

The `*(N)` markers are **whole-stage codegen**: each `*(N)` groups a chain of operators (Filter → Project → partial HashAggregate, for instance) that got fused into one generated JVM function tagged with codegen stage `N` — no per-row virtual-function calls between them. `Exchange` and `BroadcastExchange` are the two node types that break a whole-stage-codegen region, because they're the shuffle/broadcast boundaries described in the Resource-Layer Map's Network row.

For a more compact, IDE-friendly variant of the same physical plan, `explain("formatted")` numbers each node and prints operator details separately:

```
>>> result.explain("formatted")
== Physical Plan ==
* HashAggregate (7)
+- Exchange (6)
   +- * HashAggregate (5)
      +- * Project (4)
         +- * BroadcastHashJoin Inner BuildRight (3)
            :- * Filter (2)
            :  +- * ColumnarToRow (1)
            +- BroadcastExchange (0)

(1) ColumnarToRow
Input: [user_id#1, status#4, amount#5, region#3]

(2) Filter [codegen id : 1]
Input : [user_id#1, status#4, amount#5, region#3]
Condition : (isnotnull(status#4) AND (status#4 = active))

(3) BroadcastHashJoin [codegen id : 2]
Left keys: [user_id#1]
Right keys: [user_id#7]
Join type: Inner
Join condition: None
...
```

Once Catalyst produces a physical plan, **whole-stage code generation** takes each stage (a chain of operators with no shuffle boundary in between) and compiles it into a single generated JVM function, rather than executing it as a tree of iterator objects calling each other (the "Volcano model," where every single row triggers a virtual function call up through several stacked operators). Fusing `filter → project → aggregate` into one generated loop over a batch of rows removes that per-row call overhead — this is the direct payoff for the CPU row in the resource-layer table above: Tungsten's off-heap memory format is precisely what whole-stage codegen operates on directly, without JVM object deserialization at each step.

**2026 development worth flagging**: the JVM-based Catalyst+Tungsten pipeline itself is now being partially bypassed by native (C++/Rust) execution engines that plug in at the physical-execution layer while Catalyst still does the logical planning — Databricks' proprietary Photon, and open alternatives Apache Gluten (Apache Top-Level Project as of March 2026, Velox/ClickHouse backends) and Apache DataFusion Comet (1.0 stable, August 2026, Rust-based). These replace whole-stage-codegen's JVM bytecode with vectorized native code operating on columnar batches, reporting 2-5x speedups on analytical workloads specifically because native columnar execution avoids JVM overhead that even whole-stage codegen can't fully eliminate. This is a genuinely active area — verify which (if any) of these your specific Spark deployment has wired in before assuming vanilla Tungsten codegen is what's actually executing a given job.

---

## High-to-Low Walkthrough: `df.groupBy().agg()` to Shuffle Bytes

Starting from the literal call:

```python
totals = df.groupBy("region").agg(sum("amount"))
totals.write.mode("overwrite").parquet("s3://bucket/region_totals/")   # ACTION
```

```
df.groupBy("region").agg(sum("amount"))
      │
      ▼
Logical plan node appended        no execution yet — this just adds an
(lazy)                             Aggregate node to the DAG; Spark doesn't
                                    know the data yet, only the shape of the
                                    computation
      │
      ▼
ACTION triggers Catalyst          Catalyst runs its 4 phases: analysis
                                    (resolve column names/types) → logical
                                    optimization (predicate pushdown, etc.,
                                    described above) → physical planning
                                    (choose actual join/aggregate algorithms,
                                    e.g. hash aggregate vs sort aggregate) →
                                    (in Spark 3+) whole-stage codegen compiles
                                    the chosen physical plan
      │
      ▼
Stage boundary: the shuffle       groupBy requires rows with the same key
                                    ("region") to end up on the same executor
                                    — this is a WIDE transformation, and wide
                                    transformations are exactly what create a
                                    shuffle (a stage boundary): the current
                                    stage ends, a new stage begins on the
                                    other side of the shuffle
      │
      ▼
Map side (per source partition)   each task computes a PARTIAL aggregate for
                                    its local partition (partial sums per
                                    region, computed locally before any
                                    network transfer — this partial
                                    aggregation is what makes groupBy cheaper
                                    than a naive "shuffle everything then
                                    aggregate"), then writes shuffle output
                                    to local disk, bucketed by target
                                    partition (which reduce task will need it)
      │
      ▼
Shuffle read (reduce side)        each reduce-side task fetches the shuffle
(network + disk)                   blocks addressed to it from every map
                                    task's local shuffle output — this fetch
                                    is the actual network-bound step; the
                                    reduce task then combines the partial
                                    aggregates it received into the final
                                    per-region sums
      │
      ▼
AQE may re-plan mid-execution     once actual shuffle output sizes are known
                                    (not just the pre-execution estimate),
                                    Adaptive Query Execution can coalesce
                                    many small shuffle partitions into fewer
                                    larger ones, or switch a planned
                                    sort-merge join to a broadcast join if
                                    the actual data turned out small enough
                                    — this is a genuinely runtime decision,
                                    not something the original physical plan
                                    committed to upfront
      │
      ▼
Final result materialized          collected to driver, or written out
```

What the Spark UI's **SQL tab → query detail** actually reports for the map-side and reduce-side stages of that shuffle (representative numbers for a ~50 GB input aggregated down to a handful of regions):

```
Stage 4 (map side — WholeStageCodegen fuses Filter+Project+partial HashAggregate)
  Tasks: 200 succeeded / 200 total
  Input Size: 50.1 GB
  Shuffle Write: 3.2 GB           -- partial sums per (task, region) bucket, not raw rows
  Shuffle Write Records: 812,004,113 -> collapses to a few hundred partial-sum rows per task
  Duration: 58 s

Exchange hashpartitioning(region#3, 200)   -- the stage boundary itself, shown as a node in the SQL DAG

Stage 5 (reduce side — final HashAggregate)
  Tasks: 200 succeeded / 200 total
  Shuffle Read: 3.2 GB / 812,004,113 records
  Duration: 41 s
```

The gap between "Input Size: 50.1 GB" and "Shuffle Write: 3.2 GB" is the partial (map-side) aggregation paying for itself — most of the reduction already happened before a single byte crossed the network.

---

## Deep Internals

### Shuffle: Sort-Based by Default, Push-Based as an Optimization

Spark's shuffle implementation evolved from an early hash-based shuffle (each map task writes one file per reduce task — file-count blows up as `map_tasks × reduce_tasks` grows) to the current sort-based shuffle (each map task writes one sorted, indexed output file per task, which reduce tasks read specific byte ranges from) — this reduced file-handle and small-file overhead dramatically at scale. A further optimization, **push-based shuffle** (Magnet, contributed by LinkedIn), has map tasks proactively push their shuffle blocks to remote shuffle services which merge blocks from multiple map tasks by target partition ahead of time — this converts what would otherwise be many small random reads (one per map task, per reduce task) on the reduce side into fewer large sequential reads, at the cost of an extra write hop. Whether push-based shuffle is active depends on your cluster's shuffle service configuration — it is not universally on by default across all deployments.

Shuffle partition count is a config, not something Spark infers per-query — check and set it directly:

```python
spark.conf.get("spark.sql.shuffle.partitions")   # '200' -- Spark's long-standing default post-shuffle width

spark.conf.set("spark.sql.shuffle.partitions", 200)   # explicit; the number that matters for every wide transformation below
```

Partition count before a shuffle comes from the input layout; after a shuffle it's driven entirely by `spark.sql.shuffle.partitions` (unless AQE coalesces it — see below):

```python
df.rdd.getNumPartitions()                       # e.g. 87 -- inherited from input file/block layout

wide = df.groupBy("region").agg(sum("amount"))
wide.rdd.getNumPartitions()                     # 200 -- now driven by spark.sql.shuffle.partitions, not the input
```

`repartition()` vs `coalesce()` — literal calls, not interchangeable:

```python
# repartition(): triggers a FULL shuffle; can increase partition count; can target a
# column so all rows for a key land in the same partition (useful before a join on that key)
df.repartition(200, "region")

# coalesce(): merges adjacent existing partitions WITHOUT a full shuffle; cheaper, but
# can only reduce partition count and cannot fix skew (it just glues existing partitions together)
df.coalesce(50)
```

Annotated Spark UI **stage detail** page for that shuffle's map and reduce sides (representative numbers, 200 tasks per side):

```
Stage 7: Exchange hashpartitioning(region#3, 200)
  Tasks: 200 succeeded / 200 total
  Shuffle Write: 4.1 GB   (~20 MB/task average -- flag any task far above this, that's skew)
  Shuffle Write Records: 812,004,113

Stage 8: HashAggregate (reduce side, reads the shuffle output)
  Tasks: 200 succeeded / 200 total
  Shuffle Read: 4.1 GB / 812,004,113 records
  Task Time (GC Time): 6.4 min (0.3 min)   -- low GC time relative to task time is Tungsten's
                                              off-heap row format at work: little JVM-managed
                                              object data for the collector to trace
```

Enabling push-based (Magnet) shuffle, where the cluster's external shuffle service supports it:

```python
spark.conf.set("spark.shuffle.push.enabled", "true")   # off by default; requires a compatible external shuffle service on the cluster
```

### Adaptive Query Execution (AQE)

AQE re-optimizes the physical plan using runtime statistics gathered *between* stages (specifically, after a shuffle materializes and Spark can see the actual size of the data, not just the catalog/cost-based estimate used at initial planning time). Concretely, AQE can: coalesce many small post-shuffle partitions into fewer right-sized ones (avoiding the "too many tiny tasks" overhead of a bad initial partition-count guess), switch a sort-merge join to a broadcast join if one side turned out small enough to broadcast, and handle data skew by splitting an oversized partition into multiple tasks instead of one task processing a disproportionate share of the data. This closes a real gap that pure query-planning-time optimization can't: cardinality/size estimates before execution are frequently wrong, especially after several chained transformations, and AQE is Spark's answer to correcting course once ground truth is available.

The relevant config keys, literally:

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")                       # default true since Spark 3.2
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")    # merge small post-shuffle partitions (default true)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")              # split oversized/skewed partitions (default true)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10 * 1024 * 1024)   # 10 MB default ceiling for broadcast-join promotion
```

**Worked example — a join Spark plans as sort-merge, that AQE demotes to broadcast at runtime.** The optimizer doesn't know `lookup` will be filtered down to something tiny until the filter actually runs, so at planning time both sides look too large to broadcast:

```python
events = spark.read.parquet("s3://bucket/events/")        # large fact table, size known from file stats
lookup = spark.read.parquet("s3://bucket/lookup_raw/")     # large-looking source table in the catalog

# Only rows with is_current == True are relevant -- but Catalyst can't know how
# selective this filter is until it actually executes it.
current_lookup = lookup.filter(col("is_current") == True)

joined = events.join(current_lookup, "key")
```

The **initial** physical plan (chosen before any shuffle has run, so both sides are still assumed too big to broadcast):

```
== Physical Plan (initial) ==
*(5) SortMergeJoin [key#10], [key#22], Inner
:- *(2) Sort [key#10 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(key#10, 200)
:     +- *(1) FileScan parquet events[key#10,...] Batched: true
+- *(4) Sort [key#22 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(key#22, 200)
      +- *(3) Filter (isnotnull(is_current#23) AND (is_current#23 = true))
         +- FileScan parquet lookup_raw[key#22,is_current#23,...] Batched: true
```

Once the `Exchange` feeding the `lookup` side of the join actually materializes, AQE sees the filtered result is only ~8 MB — under the 10 MB `spark.sql.autoBroadcastJoinThreshold` — and re-optimizes the *remaining, not-yet-executed* part of the plan:

```
== Physical Plan (final, after AQE re-optimization) ==
*(3) BroadcastHashJoin [key#10], [key#22], Inner, BuildRight
:- *(3) FileScan parquet events[key#10,...] Batched: true
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false])), [id=#77]
   +- *(2) Filter (isnotnull(is_current#23) AND (is_current#23 = true))
      +- FileScan parquet lookup_raw[key#22,is_current#23,...] Batched: true
-- both Sort nodes and one Exchange are gone entirely: a broadcast join needs
-- neither side sorted nor both sides shuffle-partitioned on the join key
```

Calling `.explain()` on the finished job shows both plans in one place — this `AdaptiveSparkPlan isFinalPlan=true` wrapper, with `== Final Plan ==` and `== Initial Plan ==` sections, is exactly how Spark exposes the fact that AQE changed its mind mid-execution:

```
>>> joined.explain()
AdaptiveSparkPlan isFinalPlan=true
+- == Final Plan ==
   *(3) BroadcastHashJoin [key#10], [key#22], Inner, BuildRight
   :- *(3) FileScan parquet events...
   +- BroadcastExchange HashedRelationBroadcastMode(...), [id=#77]
      +- *(2) Filter (isnotnull(is_current#23) AND (is_current#23 = true))
         +- FileScan parquet lookup_raw...
+- == Initial Plan ==
   SortMergeJoin [key#10], [key#22], Inner
   :- Sort [key#10 ASC NULLS FIRST], false, 0
   :  +- Exchange hashpartitioning(key#10, 200)
   :     +- FileScan parquet events...
   +- Sort [key#22 ASC NULLS FIRST], false, 0
      +- Exchange hashpartitioning(key#22, 200)
         +- Filter (isnotnull(is_current#23) AND (is_current#23 = true))
            +- FileScan parquet lookup_raw...
```

### Tungsten: Off-Heap Binary Format

Tungsten represents rows in a compact binary format (not JVM objects) stored largely off-heap, which avoids two costs simultaneously: JVM object overhead (each Java object carries a header, and boxed primitives like `Integer` cost far more than the 4 bytes an `int` needs), and GC pressure (data the GC doesn't need to trace because it's off-heap raw memory, not JVM-managed objects, can't contribute to GC pause time). This is the direct mechanism behind the memory row of the resource-layer table: Spark chose to take on the complexity of managing its own binary memory layout specifically to sidestep JVM memory management overhead for the bulk of the data it processes.

Off-heap memory is one of three pillars Project Tungsten shipped together (Databricks, [2015](https://www.databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)) — the other two are whole-stage code generation, covered above as the signature mechanism, and **cache-aware computation**: designing Tungsten's internal data structures and algorithms (sorting, hashing, aggregation) around the CPU cache hierarchy instead of treating memory as flat and uniform. Concretely, this means favoring sequential, array-style access over pointer-chasing through scattered JVM objects, because a pointer chase can land anywhere in RAM and miss L1/L2/L3 cache on every hop, while a sequential scan over Tungsten's packed binary rows stays cache-resident far longer. This is the same cache-locality crux behind columnar engines generally — see [`performance/resource-layer-crux-guide.md`](/systems-engineering/performance/resource-layer-crux-guide.md)'s Layer 3 for the general CPU-cache-miss cost and why row-at-a-time, pointer-heavy processing loses to it.

Off-heap execution memory is a config, and it's separate from the on-heap JVM pool:

```python
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "4g")   # size of the off-heap Tungsten pool; separate from executor JVM heap sizing
```

To see the actual generated Java source a whole-stage-codegen region compiles down to (useful when tracking down why a specific query isn't fusing the way you expect):

```python
>>> result.queryExecution.debug.codegen()
Found 3 WholeStageCodegen subtrees.
== Subtree 1 / 3 (maxMethodCodeSize:187) ==
*(1) Filter (isnotnull(status#4) AND (status#4 = active))
+- *(1) ColumnarToRow
   +- FileScan parquet orders...

Generated code:
/* 001 */ public Object generate(Object[] references) {
/* 002 */   return new GeneratedIteratorForCodegenStage1(references);
/* 003 */ }
/* 004 */ ...
```

### Partitioning: The Unit of Parallelism

A DataFrame is split into partitions, and partition count is the ceiling on task parallelism for a given stage — too few partitions underutilizes a large cluster (some executors idle while a handful of oversized partitions are still processing); too many creates per-task scheduling overhead that can dominate actual work for small partitions. `repartition()` triggers a full shuffle to redistribute into a new partition count/scheme; `coalesce()` merges existing partitions without a full shuffle (cheaper, but can only reduce partition count, and can't fix skew the way a full shuffle-based repartition can).

Partitioning also determines the physical directory layout on write — `partitionBy` creates one subdirectory per distinct value combination, which is what lets a downstream reader prune whole directories instead of scanning files:

```python
df.write.partitionBy("region", "event_date").mode("overwrite").parquet("s3://bucket/out/")
# s3://bucket/out/region=US/event_date=2026-09-01/part-*.parquet
# s3://bucket/out/region=EU/event_date=2026-09-01/part-*.parquet
# ...a downstream spark.read.parquet(...).filter(col("region")=="US") can skip
# every region= directory except one, without opening a single file in them
```

---

## Comparative

**vs. Trino/DuckDB** (see `query-engines/duckdb.md`): both Trino and DuckDB assume interactive, largely memory-resident query execution and don't have Spark's shuffle-to-disk-with-lineage-based-recompute durability model — Spark's willingness to spill and recompute from lineage is what lets it run jobs whose total intermediate data is many times larger than available cluster memory, which is a genuinely different operating regime than "run one query that should finish in seconds against data that mostly fits in memory."

**vs. Hadoop MapReduce** (see `storage/hadoop.md`): MapReduce materializes every map output and every reduce output to disk as a hard architectural requirement — Spark's core innovation was making that materialization optional, keeping intermediate RDD/DataFrame results in memory across stages whenever they fit, and only writing to disk at true shuffle boundaries (and even then, recomputing from lineage on failure rather than requiring the write to have already fully succeeded and be re-read). This is precisely why Spark iterative workloads (multiple passes over the same dataset, common in ML training loops) vastly outperform the equivalent chained MapReduce jobs — MapReduce pays a full write-then-read-from-disk tax between every single stage, unconditionally.

---

## Key Gotchas

- **Shuffle is the thing to design around, not an incidental cost**: any `groupBy`, `join` (non-broadcast), `distinct`, or `repartition` is a wide transformation and triggers a shuffle — chaining several of these back-to-back without considering partition counts/skew is the single most common source of Spark job slowness.
- **Skewed keys defeat partition-count tuning**: if one key (e.g. `region = 'unknown'` holding 40% of rows) dominates a `groupBy`, that one partition's task becomes the job's critical path no matter how many total partitions you configure — AQE's skew-handling can split it automatically in modern versions (`spark.sql.adaptive.skewJoin.enabled`), but only if enabled and only within its detection thresholds; don't assume it always catches every skew case. A manual escape hatch when AQE doesn't catch it is key salting:
  ```python
  from pyspark.sql.functions import concat, lit, rand, floor

  salted = df.withColumn("salted_key", concat(col("region"), lit("_"), floor(rand() * 10)))
  # groupBy("salted_key") spreads the dominant region across 10 partitions instead of 1,
  # then a second aggregation stage combines the 10 partial results per region
  ```
- **`repartition()` vs `coalesce()` are not interchangeable**: `coalesce()` is cheap but can't increase partition count or fix skew (it can only merge adjacent existing partitions); reaching for `coalesce()` to fix a skewed job when a real shuffle-based `repartition()` was needed just moves the problem, it doesn't solve it.
- **Off-heap Tungsten memory and on-heap JVM memory are separately configured and separately exhaustible**: an OOM in a Spark executor can come from either pool, and the fix (adjusting `spark.memory.offHeap.size` vs. general executor memory/heap settings) depends on which pool is actually exhausted — check which before tuning blindly.
- **Broadcast joins have a size ceiling for a reason**: broadcasting a "small" table means shipping a full copy to every executor — past a certain size (`spark.sql.autoBroadcastJoinThreshold`, 10 MB by default), this either OOMs executors or simply costs more network/memory than the sort-merge join it was meant to avoid. AQE can auto-promote a join to broadcast at runtime (worked example above), but a manually-forced broadcast hint (`df.hint("broadcast")`) on a table that grows over time is a common source of a job that used to be fast suddenly failing.
- **Lineage-based fault tolerance assumes recompute is acceptable, not free**: losing a partition's cached/shuffled data mid-job means Spark recomputes it from the lineage graph — for a job with many chained expensive transformations upstream of the lost partition, this can be a very expensive retry, not a cheap one; this is why checkpointing (`df.checkpoint()`, breaking lineage explicitly) matters for long iterative jobs.
- **Native execution engines (Photon/Gluten/Comet) change operator support surface, not just speed**: these accelerate a *subset* of operators/expressions natively and fall back to JVM execution for the rest — a job using unsupported UDFs or expressions may silently get little to no speedup, or (depending on the integration) behave subtly differently on edge cases (null handling, floating-point rounding) versus vanilla Catalyst/Tungsten execution. Verify operator coverage for your specific workload before assuming a native-engine plugin is accelerating the whole job.

---

*Grounded against spark.apache.org release notes and current (2026) community writeups on AQE, shuffle evolution, and native execution engines. Current release line: Spark 4.1.x (4.1.2, May 2026) is the latest 4.1 patch, with the Spark 4.0 line still receiving maintenance releases (4.0.3) in parallel. Native execution engines (Photon proprietary to Databricks; Gluten reached Apache Top-Level Project status March 2026; DataFusion Comet reached 1.0 in August 2026) are an active area — re-verify which, if any, are active in a specific deployment before assuming vanilla JVM Catalyst/Tungsten execution. `explain()`/`.explain("formatted")` output, Spark UI stage numbers, and byte/record counts shown throughout this doc are representative illustrations of the mechanisms described, not captured output from one specific run — shapes and relative magnitudes are accurate, exact IDs/bytes will differ per job.*
