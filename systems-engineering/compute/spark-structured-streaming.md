# Spark Structured Streaming

## 30-Second Intuition

Structured Streaming is not a separate engine. It is a scheduling loop wrapped around Spark's ordinary batch SQL engine. Each micro-batch is a real Spark job, planned and executed by the same Catalyst/Tungsten machinery covered in [`compute/spark.md`](/systems-engineering/compute/spark.md), triggered repeatedly against whatever new data has arrived since the last run. This micro-batch model is why Structured Streaming's latency floor has historically been seconds, not milliseconds, no matter how well-tuned the query is. Every trigger pays a real, if small, batch-job-launch tax that a streaming-native engine like Flink never pays. Spark 4.1 introduced a **Real-Time Mode** alongside `transformWithState` in 2026, moving that floor from "seconds" toward sub-second territory for the first time. Most existing comparison material still treats "Spark streaming = seconds, Flink = milliseconds" as a fixed law rather than a gap that is actively closing — the operational fact that matters is that this floor is no longer fixed.

---

## Resource-Layer Map

| Layer | Role Structured Streaming plays | What it's optimizing for |
|---|---|---|
| CPU | Each micro-batch is a full Catalyst-planned, whole-stage-codegen'd batch job (see [`compute/spark.md`](/systems-engineering/compute/spark.md)) — reused machinery, not a separate streaming code path | Amortize Spark's existing query-optimization investment across streaming and batch, at the cost of per-trigger planning/launch overhead that a purpose-built streaming engine avoids |
| Memory | Stateful operators (`groupBy`+aggregation, `mapGroupsWithState`) hold running state in memory by default, or in RocksDB (see [`data-structures/rocksdb.md`](/systems-engineering/data-structures/rocksdb.md)) for large state | The in-memory default is simplest but drives JVM heap/GC pressure as state grows; RocksDB moves state off-heap into native memory + local disk to avoid that GC cost at scale |
| Disk | Checkpoint location (HDFS-compatible path, commonly S3) stores offsets, state snapshots, and the write-ahead log for exactly-once recovery | Durability of "where did we leave off" across driver/executor restarts — the checkpoint is the source of truth for exactly-once semantics, not any in-memory bookkeeping |
| Network | Kafka source/sink reads and writes (see [`streaming/kafka.md`](/systems-engineering/streaming/kafka.md)) are the dominant network cost; shuffle for stateful `groupBy` aggregations behaves like any Spark shuffle (see `compute/spark.md`'s shuffle section) | Same shuffle-is-expensive story as batch Spark — a streaming `groupBy` still redistributes data by key across executors every micro-batch |
| GPU | Not applicable | No native GPU path; irrelevant to streaming state management or micro-batch scheduling |

The sharpest resource-layer contrast is with true streaming, covered in [`compute/flink.md`](/systems-engineering/compute/flink.md). Flink's checkpoint-barrier mechanism snapshots state while records keep flowing. Structured Streaming's checkpoint happens between discrete micro-batches — there is no in-flight-record concept to snapshot around, because there is no continuous flow in the classic micro-batch model.

---

## The Signature Mechanism: Micro-Batch as a Repeated Batch Query (and the 2026 Real-Time Mode Exception)

Structured Streaming's core trick, historically: treat an unbounded input as an infinite table that is incrementally appended to, and re-run the same query against the new rows each trigger. It is implemented so each micro-batch only processes the new increment, not the whole table from scratch. Concretely, each trigger:

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

### Trigger modes, in code

**Default trigger** — runs the next micro-batch as soon as the previous one finishes:

```python
query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .start("s3://bucket/gold/clicks_agg"))
# no .trigger(...) call → default processing-time trigger, back-to-back batches
```

**Fixed processing-time interval** — run once per interval instead of back-to-back:

```python
from pyspark.sql.streaming import Trigger

query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .trigger(processingTime="1 minute")
    .start("s3://bucket/gold/clicks_agg"))
```

**`Trigger.AvailableNow`** — snapshot whatever's currently available at startup, process it across as many internal micro-batches as needed, then terminate:

```python
query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .trigger(availableNow=True)
    .start("s3://bucket/gold/clicks_agg"))

query.awaitTermination()   # returns once all currently-available data is processed
```

This mode is for scheduled incremental *batch* processing that reuses streaming checkpoint/offset-tracking machinery. It is not for continuous 24/7 operation.

**Continuous Processing** — an experimental, low-latency, non-micro-batch mode introduced in Spark 2.3:

```python
query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .trigger(continuous="1 second")   # experimental; not recommended by Databricks for production
    .start("s3://bucket/gold/clicks_agg"))
```

Still experimental as of Spark 3.5.x. Databricks does not recommend it for production. It is worth knowing this mode exists; do not design a production path around it.

**Real-Time Mode (Spark 4.1, 2026)** — the newer, more credible attempt at sub-second latency, paired with `transformWithState`:

```python
spark.conf.set("spark.sql.streaming.realTimeMode.enabled", "true")

query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .start("s3://bucket/gold/clicks_agg"))
```

This is new territory as of this doc's writing. Treat it as the answer to "is Structured Streaming catching up to Flink's latency floor," not yet as a fully load-bearing, battle-tested default the way the micro-batch model is.

---

## High-to-Low Walkthrough: A Kafka-Sourced Windowed Aggregation, One Trigger

```python
from pyspark.sql.functions import window, col

stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "clicks")
    .option("startingOffsets", "latest")
    .option("maxOffsetsPerTrigger", 100000)
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

The same aggregation expressed in SQL, registered against a temp view of the streaming DataFrame:

```sql
CREATE OR REPLACE TEMPORARY VIEW clicks_stream AS
SELECT * FROM STREAM(clicks_kafka_source);

SELECT
    window(event_time, '5 minutes') AS win,
    user_id,
    count(*) AS click_count
FROM clicks_stream
WHERE event_time > current_timestamp() - INTERVAL 10 MINUTES  -- illustrative; real watermark set via withWatermark
GROUP BY window(event_time, '5 minutes'), user_id
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
                                         to the checkpoint path — this is
                                         what makes exactly-once recovery
                                         possible after a driver restart
```

`outputMode` changes what the sink receives. Non-aggregation streams typically use `append` instead of `update`:

```python
raw_query = (stream.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-raw")
    .outputMode("append")   # no aggregation state; every new row is appended once
    .start("s3://bucket/bronze/clicks_raw"))
```

Custom sink logic that needs per-batch control (upserts, multi-table writes, non-Spark-native sinks) uses `foreachBatch`, which hands each micro-batch's result to a plain batch DataFrame:

```python
def upsert_to_delta(batch_df, batch_id):
    (batch_df.write
        .format("delta")
        .mode("append")
        .save("s3://bucket/gold/clicks_agg"))

query = (agg.writeStream
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .foreachBatch(upsert_to_delta)
    .start())
```

---

## Deep Internals

### Watermarks: How Late Is Too Late

The watermark tracks `max(event_time seen so far) - threshold`. It is a one-way, forward-only ratchet that never decreases. Processing time (wall-clock) plays no role in the calculation. A window closes once the watermark passes the window's end boundary: it stops accepting new data, emits its final result, and frees its state.

```python
from pyspark.sql.functions import window, col

windowed = (stream
    .withWatermark("event_time", "10 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("user_id"))
    .count())
```

Worked numeric trace, with round timestamps, for a `10 minute` watermark and `5 minute` tumbling windows:

```
t=12:07  event arrives, event_time = 12:07
         watermark = max(event_time) - 10min = 12:07 - 0:10 = 11:57
         → any window with end_time <= 11:57 is now CLOSED, state freed
         → windows [11:50-11:55) and [11:55-12:00) are both closed and emitted

t=12:08  a late event arrives, event_time = 11:50
         window for 11:50 is [11:50-11:55), end_time 11:55
         watermark is 11:57, and 11:55 <= 11:57
         → this window already closed one tick ago
         → the event is DROPPED, not aggregated, no error raised

t=12:08  a second late event arrives, event_time = 11:59
         window for 11:59 is [11:55-12:00), end_time 12:00
         watermark is still 11:57, and 12:00 > 11:57
         → this window is still OPEN
         → the event IS aggregated normally
```

The same trace as a runnable check against `query.lastProgress` after each batch, using a `MemoryStream` for a self-contained test:

```python
from pyspark.sql.streaming import StreamingQuery
from pyspark.testing.streamingutils import MemoryStream  # test-only utility, illustrative
import time

input_stream = MemoryStream(spark, schema="event_time timestamp, user_id string")

windowed = (input_stream.toDF()
    .withWatermark("event_time", "10 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("user_id"))
    .count())

q = windowed.writeStream.format("memory").queryName("watermark_trace").outputMode("update").start()

input_stream.addData(("2026-01-01 12:07:00", "u1"))
q.processAllAvailable()
input_stream.addData(("2026-01-01 11:50:00", "u1"))   # dropped: window already closed
input_stream.addData(("2026-01-01 11:59:00", "u1"))   # accepted: window still open
q.processAllAvailable()

spark.sql("SELECT * FROM watermark_trace").show(truncate=False)
```

Set the threshold too tight and late but real data gets dropped silently. Set it too loose and state for old windows lingers, growing memory/RocksDB usage for windows that are effectively done but not yet closed.

### RocksDB State Store

Spark 3.2 introduced RocksDB as a state store option to address JVM GC pressure from large in-memory streaming state. It runs one RocksDB instance per Spark partition, per executor, storing state in native (off-heap) memory plus local disk rather than JVM-managed heap objects. This is the same fundamental tradeoff [`data-structures/rocksdb.md`](/systems-engineering/data-structures/rocksdb.md) documents generically — LSM-tree write path, off-heap block cache — applied here to solve large streaming aggregation state causing GC pauses that hurt trigger latency. Databricks recommends RocksDB for production stateful streaming workloads for this reason.

```python
spark.conf.set(
    "spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
)
```

Additional tuning knobs that control how much of the LSM-tree write path is exposed:

```python
# Write state changes to a changelog instead of re-uploading full snapshots each checkpoint
spark.conf.set("spark.sql.streaming.stateStore.rocksdb.changelogCheckpointing.enabled", "true")

# Compact the RocksDB instance on every commit instead of relying on background compaction
spark.conf.set("spark.sql.streaming.stateStore.rocksdb.compactOnCommit", "false")

# Cap the in-memory block cache RocksDB uses per instance
spark.conf.set("spark.sql.streaming.stateStore.rocksdb.blockCacheSizeMB", "128")
```

RocksDB state-store health is visible per-batch through `StreamingQuery.lastProgress`:

```python
progress = query.lastProgress
for op in progress["stateOperators"]:
    print(op["operatorName"], op["customMetrics"].get("rocksdbBytesUsed"))
```

### Checkpointing and Exactly-Once

Periodically, not necessarily every trigger, Spark snapshots the entire state across all executors to the checkpoint location. Combined with tracked source offsets and idempotent/transactional sink writes, this is what makes end-to-end exactly-once possible. The checkpoint location must be a durable, HDFS-compatible path — S3 in most modern deployments. Losing it means losing the ability to resume correctly, not just losing some convenience metadata.

The checkpoint directory layout, inspectable directly:

```bash
aws s3 ls s3://bucket/checkpoints/clicks-agg/ --recursive
# offsets/       — per-batch committed source offsets
# commits/       — marks a batch as fully committed (offsets + state + sink write)
# state/         — versioned state-store snapshots (in-memory or RocksDB backing files)
# sources/       — source-specific metadata (e.g. Kafka partition assignment)
# metadata       — query-level metadata, including the query id
```

Recovering after a driver restart or a code deploy is just restarting the same query against the same checkpoint location — Spark resumes from the last committed offset and state version automatically:

```python
# Same code, same checkpointLocation — this IS the recovery mechanism, no special "resume" API
query = (agg.writeStream
    .format("delta")
    .option("checkpointLocation", "s3://bucket/checkpoints/clicks-agg")
    .outputMode("update")
    .start("s3://bucket/gold/clicks_agg"))
```

```bash
# Equivalent recovery via spark-submit after a driver crash — pointing at the
# unchanged checkpoint location is the entire "how do I resume" answer
spark-submit \
  --class com.example.ClicksAggJob \
  --conf spark.sql.streaming.checkpointLocation=s3://bucket/checkpoints/clicks-agg \
  clicks-agg-job.jar
```

### `transformWithState` and Real-Time Mode (Spark 4.1)

`transformWithState` pairs with Real-Time Mode to target lower latency and more flexible state management than the older `mapGroupsWithState`/`flatMapGroupsWithState` APIs. This is Spark's most direct 2026 answer to "why is Flink lower-latency," and it moves Structured Streaming's architecture toward the true-streaming end of the spectrum instead of treating micro-batch as a permanent ceiling.

The older API, for comparison — `flatMapGroupsWithState` with an explicit timeout:

```python
from pyspark.sql.streaming.state import GroupStateTimeout

def update_session(user_id, events, state):
    if state.hasTimedOut:
        state.remove()
        return []
    count = state.getOption[0] if state.exists else 0
    count += len(list(events))
    state.update((count,))
    state.setTimeoutDuration("30 minutes")
    return [(user_id, count)]

sessions = (stream.groupByKey(lambda row: row.user_id)
    .flatMapGroupsWithState(
        outputMode="update",
        timeoutConf=GroupStateTimeout.ProcessingTimeTimeout,
        func=update_session))
```

The newer API — a `StatefulProcessor` used with `transformWithState`:

```python
from pyspark.sql.streaming import StatefulProcessor, StatefulProcessorHandle

class SessionCounter(StatefulProcessor):
    def init(self, handle: StatefulProcessorHandle):
        self.count_state = handle.getValueState("count", "long")

    def handleInputRows(self, key, rows, timer_values):
        current = self.count_state.get() or 0
        current += sum(1 for _ in rows)
        self.count_state.update(current)
        yield (key, current)

    def close(self):
        pass

sessions = stream.groupByKey(lambda row: row.user_id).transformWithState(
    SessionCounter(),
    outputMode="update",
)
```

---

## Comparative

### vs. Apache Flink

The historical framing, covered in [`compute/flink.md`](/systems-engineering/compute/flink.md): Flink is streaming-native — per-record or small-buffer processing, checkpoint barriers flowing alongside data without stopping anything. Structured Streaming is micro-batch — discrete, repeated batch jobs. This gave Flink a structural latency advantage for years.

2026 status: the market has consolidated into two standard stacks rather than one winner.

- Kafka + Flink for event-driven, real-time architectures.
- Spark + Delta Lake for analytics-oriented streaming ETL.
- Most large organizations run both, not one or the other.

For new streaming-first projects, Flink SQL is the more commonly recommended default in 2026 industry commentary. Structured Streaming's advantage is that it is the same engine and API surface as an existing batch Spark/lakehouse investment — see [`data-architecture/data-platform-architectures.md`](/systems-engineering/data-architecture/data-platform-architectures.md)'s Kappa/hybrid-streaming discussion. That matters most when a team and its pipelines are already Spark-centric.

### vs. Kafka Streams

Kafka Streams uses a different deployment model. It is a library embedded directly in an application, with no separate cluster or execution engine to operate. It is used for in-app stateful processing, fraud scoring, materialized views, and event-driven microservices. It is still active in 2026, with new KIPs landing including its own rebalance protocol and native DLQ/error-handling support. This is a different shape of tool, embedded library vs. standalone streaming engine, solving a more application-embedded class of problem than Structured Streaming or Flink target.

### vs. ksqlDB

ksqlDB's status is different from Kafka Streams'. ksqlDB is feature-complete but no longer actively evolving — effectively maintenance-mode. Confluent is steering new SQL-on-streams investment toward Flink SQL instead: Confluent acquired Immerok, a Flink company, in January 2023, and has gone all-in on managed Flink since. The Kafka Streams engine underneath ksqlDB is still alive and maintained. ksqlDB, the higher-level SQL product built on top of it, is not where new investment is going.

---

## Key Gotchas

- **Micro-batch latency is a real architectural floor, not a tuning knob.** For years, no amount of trigger-interval tuning got Structured Streaming below roughly one-to-a-few-second latency, because every trigger is a real batch-job launch. Only Real-Time Mode (Spark 4.1+) meaningfully changes this, and it is new enough to validate carefully before betting a latency-sensitive production path on it.
- **Watermark threshold is a business decision with a data-loss consequence, not a performance knob.** Too tight silently drops late data that is otherwise real. Nothing errors — the data just never appears in the aggregate, as shown in the worked trace above.
- **The in-memory state store is the default, and it will eventually cause GC-driven latency spikes as state grows.** If a streaming query has any unbounded or slowly-growing stateful aggregation — long session windows, `mapGroupsWithState` without a timeout — switch to the RocksDB state store provider before it becomes an incident, not after.
- **`Trigger.AvailableNow` and Continuous Processing solve different problems and are not interchangeable.** `AvailableNow` is for scheduled incremental batch jobs reusing streaming's checkpoint/offset machinery. Continuous Processing is, still as of 3.5.x, an experimental attempt at sub-second latency via the old architecture, and Databricks does not recommend it for production. Reach for Real-Time Mode instead if running Spark 4.1+.
- **The checkpoint location is load-bearing, not incidental.** Deleting or corrupting it breaks exactly-once recovery guarantees entirely, not just some state. Treat it with the same operational care as a database's transaction log.
- **"Use Kafka Streams / ksqlDB instead" needs the actual distinction understood first.**
  - Kafka Streams is an embedded library for application-level stream processing, not a competing standalone engine for the ETL/analytics workloads Structured Streaming and Flink target.
  - ksqlDB is no longer where new investment goes, even though the Kafka Streams engine beneath it is fine.

---

*Grounded against spark.apache.org's Structured Streaming Programming Guide (3.5.x), Databricks' production-streaming and RocksDB state store documentation, and 2026 industry writeups (Kai Waehner's "Data Streaming Landscape 2026," Confluent/RisingWave/Streamkap comparisons) as of September 2026. Spark 4.1's Real-Time Mode and `transformWithState` are newly released relative to most existing comparison material — re-verify current production maturity/adoption before treating Real-Time Mode as a settled default. ksqlDB's maintenance-mode status and Kafka Streams' continued active development are both confirmed as distinct facts, not to be conflated.*
