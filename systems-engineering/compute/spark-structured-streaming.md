# Spark Structured Streaming

## 30-Second Intuition

Structured Streaming is not a separate engine — it's a scheduling loop wrapped around Spark's ordinary batch SQL engine: each micro-batch is a real Spark job, planned and executed by the exact same Catalyst/Tungsten machinery covered in [`compute/spark.md`](spark.md), just triggered repeatedly against whatever new data has arrived since the last run. The one fact that matters operationally: this micro-batch model is *why* Structured Streaming's latency floor has historically been seconds, not milliseconds, no matter how well-tuned — every trigger pays a real (if small) batch-job-launch tax that a truly streaming engine like Flink never pays. **This assumption changed in 2026**: Spark 4.1 introduced a genuine **Real-Time Mode** alongside `transformWithState`, moving Structured Streaming's floor from "seconds" toward sub-second territory for the first time — worth flagging explicitly, since most existing material (including comparisons written before this release) still treats "Spark streaming = seconds, Flink = milliseconds" as a fixed law rather than a gap that's actively closing.

---

## Resource-Layer Map

| Layer | Role Structured Streaming plays | What it's optimizing for |
|---|---|---|
| CPU | Each micro-batch is a full Catalyst-planned, whole-stage-codegen'd batch job (see [`compute/spark.md`](spark.md)) — reused machinery, not a separate streaming code path | Amortize Spark's existing query-optimization investment across streaming and batch, at the cost of per-trigger planning/launch overhead that a purpose-built streaming engine avoids |
| Memory | Stateful operators (`groupBy`+aggregation, `mapGroupsWithState`) hold running state in memory by default, or in RocksDB (see [`data-structures/rocksdb.md`](../data-structures/rocksdb.md)) for large state | The in-memory default is simplest but drives JVM heap/GC pressure as state grows; RocksDB moves state off-heap into native memory + local disk specifically to avoid that GC cost at scale |
| Disk | Checkpoint location (HDFS-compatible path, commonly S3) stores offsets, state snapshots, and the write-ahead log for exactly-once recovery | Durability of "where did we leave off" across driver/executor restarts — this checkpoint is the actual source of truth for exactly-once semantics, not any in-memory bookkeeping |
| Network | Kafka source/sink reads and writes (see [`streaming/kafka.md`](../streaming/kafka.md)) are the dominant network cost; shuffle for stateful `groupBy` aggregations behaves like any Spark shuffle (see `compute/spark.md`'s shuffle section) | Same shuffle-is-expensive story as batch Spark — a streaming `groupBy` still redistributes data by key across executors every micro-batch |
| GPU | Not applicable | No native GPU path; irrelevant to streaming state management or micro-batch scheduling |

The sharpest resource-layer contrast with true streaming ([`compute/flink.md`](flink.md)): Flink's checkpoint-barrier mechanism snapshots state *while records keep flowing*, whereas Structured Streaming's checkpoint happens *between* discrete micro-batches — there's no in-flight-record concept to snapshot around, because there's no continuous flow to begin with in the classic micro-batch model.

---

## The Signature Mechanism: Micro-Batch as a Repeated Batch Query (and the 2026 Real-Time Mode Exception)

Structured Streaming's core trick, historically: treat an unbounded input as an infinite table that's incrementally appended to, and re-run (conceptually) "the same query" against the new rows each trigger — but implemented efficiently, so each micro-batch only processes the new data, not the whole table from scratch. Concretely, each trigger:

```
Trigger N fires
      │
      ▼
Determine new data available since the last committed offset
(e.g. new Kafka offsets since the last successful micro-batch)
      │
      ▼
Catalyst plans a REAL Spark batch job for just this increment  ← same
                                                                    planner as
                                                                    compute/spark.md,
                                                                    no separate
                                                                    "streaming" planner
      │
      ▼
Job executes (whole-stage codegen, shuffle if needed, etc.)
      │
      ▼
Results appended/updated in the sink; offsets + state checkpointed
      │
      ▼
Trigger N+1 waits for the configured interval, or fires immediately
if using the default (as-fast-as-possible) trigger
```

**Trigger modes, concretely**:
- **Default (fixed/processing-time trigger)**: run the next micro-batch as soon as the previous one finishes, or after a configured interval — the historical default, latency bound by micro-batch launch + execution time (typically seconds).
- **`Trigger.AvailableNow`**: snapshot whatever's currently available at startup (e.g. current Kafka offsets), process all of it across as many internal micro-batches as needed, then terminate cleanly — designed for scheduled incremental *batch* processing that reuses streaming checkpoint/offset-tracking machinery, not for continuous 24/7 operation.
- **Continuous Processing**: an experimental, low-latency, non-micro-batch mode introduced in Spark 2.3 — explicitly still experimental as of Spark 3.5.x, and explicitly *not* recommended by Databricks for production. Worth knowing it exists, but don't design around it.
- **Real-Time Mode (Spark 4.1, 2026)**: the newer, more credible attempt at sub-second latency — paired with the `transformWithState` API for more flexible/efficient stateful processing. This is genuinely new territory as of this doc's writing; treat it as the answer to "is Structured Streaming catching up to Flink's latency floor," not yet as a fully load-bearing, battle-tested default the way the micro-batch model is.

---

## High-to-Low Walkthrough: A Kafka-Sourced Windowed Aggregation, One Trigger

```python
from pyspark.sql.functions import window, col

stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "clicks")
    .load())

agg = (stream
    .withWatermark("event_time", "10 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("user_id"))
    .count())

query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .start("s3://bucket/gold/clicks_agg"))
```

```
readStream.format("kafka")            reads new records since the last
                                        committed Kafka offset (tracked in
                                        the checkpoint location, not in
                                        Kafka consumer-group offsets)
      │
      ▼
withWatermark("event_time", "10min")  establishes how late data can arrive
                                        before it's dropped from the window
                                        aggregation — see Watermarks below
      │
      ▼
groupBy(window(...), user_id).count() STATEFUL operator — running counts per
                                        (window, user_id) key persist across
                                        triggers, in memory or RocksDB
      │
      ▼
Catalyst plans this trigger's batch    same planner/optimizer as any Spark
job (predicate pushdown, codegen)       batch query — nothing streaming-
                                         specific about this step
      │
      ▼
writeStream.format("delta")            output written to the Delta sink
outputMode("update")                    (see data-formats/delta-lake.md);
                                         "update" mode emits only rows whose
                                         aggregate changed this trigger
      │
      ▼
Checkpoint written                     new Kafka offsets + updated
(checkpointLocation)                    (window, user_id) state snapshotted
                                         to the checkpoint path — THIS is
                                         what makes exactly-once recovery
                                         possible after a driver restart
```

---

## Deep Internals

### Watermarks: How Late Is Too Late

The watermark tracks `max(event_time seen so far) - threshold` — a **one-way, forward-only ratchet that never decreases**, and critically, **processing time (wall-clock) is irrelevant to the calculation**. A window closes (stops accepting new data, emits its final result, and its state is freed) once the watermark passes the window's end boundary.

```
Watermark = "10 minutes" late-tolerance, window = 5-minute tumbling windows

Event arrives with event_time = 12:07, watermark becomes 11:57
  (max seen so far = 12:07, minus 10 min threshold)
  → any window ending at or before 11:57 is now CLOSED, state freed
  → a late-arriving event with event_time = 11:50 arriving AFTER this point
    is DROPPED — it's older than the watermark, its window already closed
```

Set the threshold too tight and genuinely late (but real) data gets silently dropped; set it too loose and state for old windows lingers far longer than necessary, growing memory/RocksDB usage for windows that are effectively done but not yet closed.

### RocksDB State Store

Introduced in Spark 3.2 specifically to address JVM GC pressure from large in-memory streaming state: one RocksDB instance per Spark partition, per executor, storing state in native (off-heap) memory plus local disk rather than JVM-managed heap objects. This is the exact same fundamental tradeoff [`data-structures/rocksdb.md`](../data-structures/rocksdb.md) documents generically (LSM-tree write path, off-heap block cache) — applied here specifically to solve "streaming aggregation state got large enough that GC pauses were hurting trigger latency," a problem the in-memory default state store doesn't have an answer to at scale. Databricks recommends RocksDB for production stateful streaming workloads for exactly this reason.

```python
spark.conf.set(
    "spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
)
```

### Checkpointing and Exactly-Once

Periodically (not necessarily every trigger), Spark snapshots the *entire* state across all executors to the checkpoint location — this, combined with tracked source offsets and idempotent/transactional sink writes, is what makes end-to-end exactly-once possible. The checkpoint location must be a durable, HDFS-compatible path (S3 in most modern deployments) — losing it means losing the ability to resume correctly, not just losing some convenience metadata.

### `transformWithState` and Real-Time Mode (Spark 4.1)

The newer stateful-processing API pairs with Real-Time Mode to target lower latency and more flexible state management than the older `mapGroupsWithState`/`flatMapGroupsWithState` APIs — this is Spark's most direct 2026 answer to the "why is Flink lower-latency" question, and represents Structured Streaming actively moving its architecture toward the true-streaming end of the spectrum rather than accepting micro-batch as a permanent ceiling.

---

## Comparative

**vs. Apache Flink** ([`compute/flink.md`](flink.md)) — the historical framing: Flink is truly streaming (per-record or small-buffer processing, checkpoint barriers flowing alongside data without stopping anything), Structured Streaming is micro-batch (discrete, repeated batch jobs). This gave Flink a genuine, structural latency advantage for years. **2026 status, grounded**: the market has consolidated into two standard stacks rather than one winner — **Kafka + Flink for event-driven real-time architectures**, and **Spark + Delta Lake for analytics-oriented streaming ETL** — with most large organizations running both, not picking one. For genuinely new streaming-first projects, Flink SQL is the more commonly recommended default in 2026 industry commentary; Structured Streaming's advantage is that it's the same engine and API surface as your existing batch Spark/lakehouse investment (see [`data-architecture/data-platform-architectures.md`](../data-architecture/data-platform-architectures.md)'s Kappa/hybrid-streaming discussion), which matters enormously if your team and pipelines are already Spark-centric.

**vs. Kafka Streams** — a fundamentally different deployment model: Kafka Streams is a *library* embedded directly in your application (no separate cluster/execution engine to operate), used for in-app stateful processing, fraud scoring, materialized views, and event-driven microservices. It's explicitly still active in 2026 (new KIPs landing, including its own rebalance protocol and native DLQ/error-handling support) — not a legacy technology, just a different shape of tool (embedded library vs. standalone streaming engine) solving a more application-embedded class of problem than Structured Streaming or Flink target.

**vs. ksqlDB** — worth knowing precisely, since its status is genuinely different from Kafka Streams': ksqlDB is feature-complete but **no longer actively evolving** — effectively maintenance-mode, with Confluent explicitly steering new SQL-on-streams investment toward Flink SQL instead (Confluent acquired Immerok, a Flink company, in January 2023, and has gone all-in on managed Flink since). The Kafka Streams engine underneath ksqlDB is still alive and maintained; ksqlDB the higher-level SQL product built on top of it is not where new investment is going.

---

## Key Gotchas

- **Micro-batch latency is a real architectural floor, not just a tuning knob** — for years, no amount of trigger-interval tuning got Structured Streaming below roughly one-to-a-few-second latency, because every trigger is a real batch-job launch; only Real-Time Mode (Spark 4.1+) meaningfully changes this, and it's new enough to validate carefully before betting a latency-sensitive production path on it.
- **Watermark threshold is a real business decision with a data-loss consequence, not a performance knob**: too tight silently drops genuinely late data; the "silently" is the dangerous part — nothing errors, the data just never appears in the aggregate.
- **In-memory state store is the default, and it will eventually cause GC-driven latency spikes as state grows** — if your streaming query has any unbounded or slowly-growing stateful aggregation (long session windows, `mapGroupsWithState` without a timeout), plan to switch to the RocksDB state store provider before it becomes an incident, not after.
- **`Trigger.AvailableNow` and Continuous Processing solve different problems and aren't interchangeable**: `AvailableNow` is for scheduled incremental batch jobs reusing streaming's checkpoint/offset machinery; Continuous Processing is (still, as of 3.5.x) an experimental attempt at sub-second latency via the OLD architecture, explicitly not production-recommended by Databricks — don't reach for Continuous Processing expecting it to be "the low-latency option," reach for Real-Time Mode instead if you're on 4.1+.
- **The checkpoint location is load-bearing, not incidental** — deleting or corrupting it doesn't just lose "some state," it breaks exactly-once recovery guarantees entirely; treat it with the same operational care as a database's transaction log.
- **"We should just use Kafka Streams / ksqlDB instead" needs the actual distinction understood first**: Kafka Streams is an embedded library for application-level stream processing, not a competing standalone engine for the same class of ETL/analytics workloads Structured Streaming and Flink target — and ksqlDB specifically is no longer where new investment goes, even though the Kafka Streams engine beneath it is fine.

---

*Grounded against spark.apache.org's Structured Streaming Programming Guide (3.5.x), Databricks' production-streaming and RocksDB state store documentation, and 2026 industry writeups (Kai Waehner's "Data Streaming Landscape 2026," Confluent/RisingWave/Streamkap comparisons) as of September 2026. Spark 4.1's Real-Time Mode and `transformWithState` are newly released relative to most existing comparison material — re-verify current production maturity/adoption before treating Real-Time Mode as a settled, battle-tested default rather than an emerging option. ksqlDB's maintenance-mode status and Kafka Streams' continued active development are both explicitly confirmed as distinct, not-to-be-conflated facts.*
