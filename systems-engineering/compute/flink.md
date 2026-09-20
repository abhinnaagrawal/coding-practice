# Apache Flink

## 30-Second Intuition

Flink is a distributed stream-processing engine built around one architectural bet that Spark Structured Streaming did not make: **records flow through the dataflow graph one at a time (or in small network buffers), continuously, with no artificial batch boundary** — there is no "trigger a micro-batch every N seconds" scheduling loop sitting between the source and the operators. Fault tolerance and exactly-once state consistency come from a distinct mechanism layered on top of that continuous flow: periodic **checkpoint barriers** injected into the record stream that force a distributed, asynchronous snapshot of all operator state without ever stopping the pipeline. The one fact that matters operationally: Flink's latency floor is bound by per-record/per-buffer processing and network flush intervals (single-digit milliseconds is normal), not by a batch-interval scheduling constant — the tradeoff is a checkpointing protocol that has to solve distributed-snapshot alignment instead of just "commit the finished micro-batch," which is where most of Flink's operational complexity (barrier alignment, backpressure interaction, watermark tuning) actually lives.

---

## Resource-Layer Map

| Layer | Role Flink plays | What it's optimizing for |
|---|---|---|
| CPU | Continuous per-record (or per-buffer) operator execution across all TaskManager slots, all the time — no idle-then-burst batch cadence | Keep pipeline latency low by never waiting to accumulate a batch; CPU is spent constantly at whatever the sustained input rate demands, not in scheduled bursts |
| Memory | JVM heap for operator logic + a managed off-heap memory segment pool for network buffers and (with `HashMapStateBackend`) in-memory keyed state | Buffer just enough in-flight data (network buffers, in-flight checkpoint barriers) to keep the streaming pipeline flowing without stalling on backpressure |
| Disk | `EmbeddedRocksDBStateBackend` for keyed state that exceeds memory (see `data-structures/rocksdb.md` for the LSM internals) — checkpoints persist snapshots to durable storage (S3/HDFS) as they complete | State that doesn't fit in memory spills to a local RocksDB instance per TaskManager; checkpoint completion writes durable copies out to remote storage, decoupled from the local state store |
| Network | Checkpoint barriers flow inline with records, and completing a checkpoint under barrier alignment is a distributed coordination cost, not just a data-transfer cost | Barriers must reach every operator instance and (with aligned checkpoints) all input channels' barriers must arrive together before that operator snapshots — under backpressure, slow channels stall barrier alignment, which is the direct cost this system pays for exactly-once semantics without stopping data flow |
| GPU | Not part of the core engine's execution model; GPU-accelerated ML inference is invoked from Flink jobs (e.g. Flink ML, PyFlink UDFs calling out to GPU-backed models) rather than being a first-class execution target for the dataflow runtime itself | Not applicable to Flink's own record-processing path |

The sharpest resource-layer contrast is with Spark Structured Streaming (`compute/spark.md` covers core Spark's batch engine underneath it): Structured Streaming reuses Spark's batch engine and executes each micro-batch as a short-lived batch job — CPU and network activity happen in a scheduled cadence (every `trigger` interval), and exactly-once semantics come from committing a whole batch's output atomically. Flink's CPU/network activity is continuous, and exactly-once semantics come from the checkpoint-barrier protocol below, which is a genuinely different mechanism from "commit this finished batch."

---

## The Signature Mechanism: Chandy-Lamport Distributed Snapshots via Checkpoint Barriers

Flink needs a way to answer "what is a globally consistent snapshot of every operator's state, at a single logical point in the unbounded stream" — without pausing the stream to take it. The mechanism is a direct, practical adaptation of the **Chandy-Lamport distributed snapshot algorithm**: instead of stopping the world, inject a marker into the data flow itself and let the marker's propagation order define the snapshot boundary.

Concretely:

1. The **JobManager's CheckpointCoordinator** periodically (`checkpoint.interval`, e.g. every 10s or 60s) tells every source task to inject a **checkpoint barrier** — a special record-like marker tagged with a checkpoint ID — into its output stream, interleaved with normal records.
2. Barriers flow downstream **in-band, in the same order as the records around them** — a barrier for checkpoint N is never overtaken by data records that were emitted before it, and never overtakes records emitted after it. This ordering guarantee is what makes the barrier a meaningful cut point between "state as of before checkpoint N" and "state as of after."
3. An operator with **multiple input channels** (e.g. a downstream operator reading from several upstream parallel instances) must wait until it has received the checkpoint-N barrier **from every input channel** before it snapshots its own state — this is **barrier alignment**. Channels whose barrier has already arrived have their subsequent records buffered (not processed) until the last channel's barrier shows up, so the operator's state reflects the same consistent instant across the union of its inputs.
4. Once aligned, the operator **asynchronously snapshots its state** (write to `HashMapStateBackend`'s heap objects or fetch RocksDB's SSTable state, per `data-structures/rocksdb.md`'s incremental-checkpoint mechanics) to durable storage, then forwards the barrier downstream and resumes processing buffered + new records.
5. The checkpoint is **complete** once every operator in the graph has acknowledged its snapshot to the JobManager. Only then does state for checkpoint N count as durable and recoverable.

```
Source (barrier injected)
   │  ...records... [Barrier N] ...records...
   ▼
Operator A (1 input channel: trivial alignment)
   │  snapshots state as soon as its one barrier arrives
   ▼
   │  [Barrier N]
   ├──────────────┐
   ▼              ▼
Operator B     Operator C     (both feed into Operator D)
   │ [Barrier N]   │ [Barrier N, arrives LATER]
   └──────┬────────┘
          ▼
     Operator D (2 input channels)
       - barrier from B arrives first: buffer B's post-barrier records,
         keep processing C's pre-barrier records
       - barrier from C arrives: NOW aligned — snapshot D's state,
         then release B's buffered records and forward barrier N
```

This is exactly why the resource-layer table's Network row calls out barrier alignment cost: a slow or backpressured input channel doesn't just delay its own throughput, it delays every downstream operator's checkpoint, because alignment requires waiting for the *last* channel — this is elaborated in Deep Internals and Gotchas below (and is the reason **unaligned checkpoints** exist as an escape hatch).

Trigger a checkpoint manually and inspect the coordinator's view of it via the CLI/REST API — this is the literal operational surface for the mechanism above:

```bash
# Trigger a checkpoint on demand (does not require waiting for the next scheduled interval)
curl -X POST http://jobmanager:8081/jobs/<job-id>/checkpoints
# {"request-id":"c112233"}

# Poll for that checkpoint's status/details
curl http://jobmanager:8081/jobs/<job-id>/checkpoints/details/c112233
```

```json
{
  "id": 42,
  "status": "COMPLETED",
  "checkpoint_type": "CHECKPOINT",
  "trigger_timestamp": 1757750000000,
  "latest_ack_timestamp": 1757750000842,
  "state_size": 1073741824,
  "duration": 842,
  "alignment_buffered": 3145728,
  "num_subtasks": 24,
  "num_acknowledged_subtasks": 24
}
```

`duration: 842` (ms) is end-to-end checkpoint time; `alignment_buffered: 3145728` bytes is data buffered while waiting on slower input channels — a nonzero, growing value here across checkpoints is the direct fingerprint of barrier-alignment stalling under backpressure.

---

## High-to-Low Walkthrough: Submitting a Windowed Aggregation Over Kafka

The job: read clickstream events from Kafka, tumbling-window count per `user_id` every 1 minute of event time, write results back to Kafka.

### 1. Submit via the CLI

```bash
flink run \
  --target yarn-per-job \
  --class com.example.ClickCounterJob \
  --parallelism 24 \
  ./click-counter-1.0.jar \
  --checkpoint-interval 60000
```

```
Job has been submitted with JobID 3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c
```

### 2. Or the same job as a Flink SQL Client session

```sql
-- flink-sql-client, connecting to a session cluster
CREATE TABLE clicks (
  user_id    STRING,
  event_time TIMESTAMP(3),
  page       STRING,
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'clickstream',
  'properties.bootstrap.servers' = 'kafka-broker:9092',
  'properties.group.id' = 'click-counter',
  'scan.startup.mode' = 'latest-offset',
  'format' = 'json'
);

CREATE TABLE click_counts (
  window_start TIMESTAMP(3),
  window_end   TIMESTAMP(3),
  user_id      STRING,
  click_count  BIGINT
) WITH (
  'connector' = 'kafka',
  'topic' = 'click-counts',
  'properties.bootstrap.servers' = 'kafka-broker:9092',
  'format' = 'json'
);

INSERT INTO click_counts
SELECT window_start, window_end, user_id, COUNT(*) AS click_count
FROM TABLE(
  TUMBLE(TABLE clicks, DESCRIPTOR(event_time), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, user_id;
```

```
[INFO] Submitting SQL update statement to the cluster...
[INFO] Table update statement has been successfully submitted to the cluster:
Job ID: a1b2c3d4e5f60718293a4b5c6d7e8f90
```

### 3. JobManager builds the JobGraph and schedules tasks onto TaskManagers

The submitted job (JAR or SQL-compiled plan) is turned into a **JobGraph** (operators + their parallelism), which the JobManager's scheduler turns into an **ExecutionGraph** — one parallel task instance per operator per parallelism slot — and assigns each task to a **task slot** on a TaskManager. Check placement directly:

```bash
flink list -a
```

```
------------------ Running/Restarting Jobs -------------------
03.09.2026 14:22:10 : 3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c : ClickCounterJob (RUNNING)
```

```bash
curl http://jobmanager:8081/jobs/3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c/vertices
```

```json
{"vertices":[
  {"id":"cbc357ccb763df2852fee8c4fc7d55f2","name":"Source: clickstream[Kafka]","parallelism":24},
  {"id":"88d3c1f5b5e7a...","name":"TumblingWindow(user_id, 1min)","parallelism":24},
  {"id":"a92b7e...","name":"Sink: click-counts[Kafka]","parallelism":24}
]}
```

Each of the 24 parallel subtasks of the window operator is a chain of Kafka partition consumption → keyed windowing → sink, scheduled onto whichever TaskManager has a free slot, ideally colocated to minimize network shuffle for the keyBy that groups by `user_id`.

### 4. Watermarks generate and propagate for event-time correctness

Each of the 24 source subtasks generates its own watermark from the events it reads (per the `WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND` clause above, or the DataStream equivalent shown in Deep Internals), and a downstream operator's effective watermark is the **minimum** across all its input channels — one slow/idle partition holds back the whole operator's notion of event-time progress. The Flink Web UI's watermark view per operator (or the metrics REST endpoint) shows this directly:

```bash
curl http://jobmanager:8081/jobs/3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c/vertices/88d3c1f5b5e7a.../watermarks
```

```json
{
  "watermarks": [
    {"subtask": 0,  "watermark": 1757750340000},
    {"subtask": 1,  "watermark": 1757750340000},
    {"subtask": 7,  "watermark": 1757750280000}
  ]
}
```

Subtask 7 is 60 seconds behind the others (partition-level lag or skew) — the operator's effective watermark is `min(...)` = subtask 7's value, so windows that could otherwise close based on subtasks 0/1's progress stay open, waiting on subtask 7. This is the concrete mechanism behind "watermark lag" as an operational metric, and it's why a single skewed Kafka partition can hold back an entire windowed job's output.

### 5. A checkpoint is triggered and completes mid-job

While the job runs, the CheckpointCoordinator fires checkpoint 42 on schedule (`checkpoint-interval 60000` from the CLI submission above). Barriers flow through source → window operator (aligning across all 24 keyBy input channels) → sink, each operator snapshots per the Signature Mechanism section, and the Web UI's Checkpoints tab shows the completed result:

```
Checkpoint 42
  Status:        COMPLETED
  Trigger Time:  14:31:00.000
  Latest Ack:    14:31:00.842
  Duration:      842 ms
  State Size:    1.08 GB   (RocksDB incremental delta since checkpoint 41)
  Alignment:     124 ms buffered (subtask 7's slow channel)
```

The 124ms of alignment buffering directly reflects subtask 7's watermark lag above — the same slow partition that holds back watermark progress also holds back checkpoint-barrier alignment, because both are gated on "slowest input channel."

---

## Deep Internals

### Event Time vs. Processing Time, Watermarks, and Late Data

Flink supports three time notions, but **event time** (the timestamp embedded in the record itself, e.g. when a click actually happened) is what makes results reproducible regardless of processing delays — **processing time** (wall-clock time at the operator) is faster and simpler but non-deterministic: rerunning the same job over the same data at a different processing speed produces different windowing results.

A **watermark** is a declaration: "no more events with event time ≤ W will arrive on this stream" (an unenforceable-but-assumed contract that the engine uses to decide when a time-based window can be considered closed). Watermark generation, literally, in the DataStream API:

```java
DataStream<ClickEvent> withWatermarks = clicks
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<ClickEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
            .withTimestampAssigner((event, ts) -> event.getEventTimeMillis())
    );
```

`forBoundedOutOfOrderness(Duration.ofSeconds(5))` means: watermark = `max event time seen so far - 5s`. This is a deliberate slack, not a guess at perfection — it says "I'm willing to wait up to 5 extra seconds of wall-clock-observed event time before I consider a window closed, to tolerate up to 5s of out-of-order arrival."

**Worked example — late data and allowed lateness.** A 1-minute tumbling window `[14:31:00, 14:32:00)` on `user_id=42`:

```
Events arrive (event_time, arrival order):
  14:31:10  -> arrives on time
  14:31:45  -> arrives on time
  14:31:58  -> arrives on time
  Watermark advances to 14:32:03 (max seen 14:32:08 - 5s slack, from a later event on another key)
  -> window [14:31:00,14:32:00) FIRES with count=3, since watermark (14:32:03) > window end (14:32:00)

  14:31:52  -> arrives LATE (event time is inside the already-fired window,
               but it showed up after the watermark passed the window's end)
```

Without `allowedLateness`, that 14:31:52 event is dropped — it's simply discarded once the window has been purged. With allowed lateness configured:

```java
DataStream<Tuple2<String, Long>> counts = withWatermarks
    .keyBy(e -> e.getUserId())
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .allowedLateness(Time.seconds(30))     // window state kept 30s past watermark-triggered firing
    .sideOutputLateData(lateDataTag)       // events past even the allowed-lateness window go here instead of being silently dropped
    .sum(1);
```

Now the 14:31:52 late event (arriving only a few seconds after the window fired, well within the 30s allowance) triggers an **updated** firing for that same window — count becomes 4 — rather than being dropped. Anything arriving more than 30s after the watermark passed 14:32:00 instead goes to the late-data side output for separate handling (alerting, a "corrections" sink, etc.) instead of silently vanishing. This is the direct, concrete tradeoff the Gotchas section below returns to: too-tight a watermark slack or allowed lateness = dropped data; too-loose = every window waits longer to fire, i.e. added end-to-end latency.

### Windowing: Tumbling, Sliding, Session — a Worked Numeric Example

```java
// Tumbling: fixed, non-overlapping 1-minute buckets
.window(TumblingEventTimeWindows.of(Time.minutes(1)))

// Sliding: 1-minute window, evaluated every 10 seconds -> each event lands in
// multiple overlapping windows
.window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(10)))

// Session: windows close after a gap of inactivity per key, size is data-driven
.window(EventTimeSessionWindows.withGap(Time.minutes(5)))
```

Worked example for `SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(10))` — a 1-minute window sliding every 10 seconds means **6 overlapping windows** are active for any given instant (60s / 10s = 6), and a single event at `14:31:23` falls into every window whose `[start, start+60s)` contains it:

```
Windows containing an event at 14:31:23:
  [14:30:30, 14:31:30)  <- contains it (barely, near the end)
  [14:30:40, 14:31:40)
  [14:30:50, 14:31:50)
  [14:31:00, 14:32:00)
  [14:31:10, 14:32:10)
  [14:31:20, 14:32:20)  <- contains it (barely, near the start)
6 windows total = window_size / slide_interval = 60s / 10s
```

That single event is replicated into 6 window-state buckets — this is the direct memory/state-size cost of sliding windows vs. tumbling: state size scales with `window_size / slide_interval`, not just with distinct keys, which matters directly for RocksDB state-backend sizing (`data-structures/rocksdb.md`).

Literal Flink SQL equivalents (the `TUMBLE`/`HOP`/`SESSION` table-valued functions):

```sql
-- Tumbling
SELECT window_start, window_end, user_id, COUNT(*) FROM TABLE(
  TUMBLE(TABLE clicks, DESCRIPTOR(event_time), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, user_id;

-- Sliding/Hopping
SELECT window_start, window_end, user_id, COUNT(*) FROM TABLE(
  HOP(TABLE clicks, DESCRIPTOR(event_time), INTERVAL '10' SECOND, INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, user_id;

-- Session
SELECT window_start, window_end, user_id, COUNT(*) FROM TABLE(
  SESSION(TABLE clicks PARTITION BY user_id, DESCRIPTOR(event_time), INTERVAL '5' MINUTE))
GROUP BY window_start, window_end, user_id;
```

### Exactly-Once vs. At-Least-Once Sink Semantics: Two-Phase Commit Sinks

Checkpointing gives exactly-once **internal state** consistency (the barrier protocol above), but getting exactly-once **external output** (e.g. a Kafka sink, a JDBC sink) requires the sink itself to cooperate with the checkpoint protocol via a **two-phase commit (2PC)**:

1. On each checkpoint, the sink **pre-commits** (e.g. for `KafkaSink`, this means flushing a Kafka transaction that is open but not yet committed).
2. Only once the JobManager confirms the checkpoint is globally complete (every operator acknowledged) does the sink **commit** that transaction — via a `notifyCheckpointComplete` callback.
3. On recovery from a failure, any sink transaction that was pre-committed but never confirmed committed is **aborted** (or, for idempotent sinks, safely reapplied) — this is what prevents a partially-applied output from ever becoming visible downstream.

```java
KafkaSink<String> sink = KafkaSink.<String>builder()
    .setBootstrapServers("kafka-broker:9092")
    .setRecordSerializer(KafkaRecordSerializationSchema.builder()
        .setTopic("click-counts")
        .setValueSerializationSchema(new SimpleStringSchema())
        .build())
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)   // <- enables the 2PC path
    .setTransactionalIdPrefix("click-counter-txn-")
    .build();
```

```java
// vs. the cheaper, no-2PC option: acknowledges as soon as the record is
// handed to the Kafka producer client, no coordination with checkpoints
.setDeliveryGuarantee(DeliveryGuarantee.AT_LEAST_ONCE)
```

`EXACTLY_ONCE` costs real overhead: Kafka transactions held open across an entire checkpoint interval, and downstream consumers of that Kafka topic must set `isolation.level=read_committed` to actually observe the exactly-once guarantee (otherwise they'll see uncommitted/aborted records too) — this is a coordinated setting across producer and consumer, not something the Flink sink can enforce unilaterally.

### State Backends (Brief — See `rocksdb.md` for RocksDB Internals)

Flink has exactly two supported state backends as of Flink 2.0 (the legacy `MemoryStateBackend`/`RocksDBStateBackend` classnames were removed, FLINK-36323):

```java
// In-heap JVM objects: fast, but bounded by heap size and full-snapshot cost scales with state size
env.setStateBackend(new HashMapStateBackend());

// RocksDB-backed: state lives on local disk (+ block cache), unbounded by heap,
// supports INCREMENTAL checkpoints (only changed SSTables uploaded per checkpoint)
env.setStateBackend(new EmbeddedRocksDBStateBackend(true));  // true = incremental checkpoints
```

Checkpoint *storage* (where the durable snapshot lands) is a separate, orthogonal config from the state *backend* (where live state lives during processing):

```java
env.getCheckpointConfig().setCheckpointStorage("s3://my-bucket/flink-checkpoints/");
```

For the LSM write path, compaction interaction, block cache sizing, and the random-key-access problem that determines whether RocksDB is I/O-bound for a given keyed-state workload, see `data-structures/rocksdb.md`'s "RocksDB as Flink State Backend" section — this doc intentionally does not re-derive that material.

**2026 development**: Flink 2.0 introduced **disaggregated state management** (FLIP-423) via a new state store called **ForSt**, which treats a remote object store (S3/HDFS) as the primary state store and local disk as a cache, rather than local-disk-primary-with-checkpoint-upload-as-an-afterthought. This targets the specific pain points of RocksDB-on-local-disk in containerized/cloud-native deployments: local disk capacity limits, compaction I/O competing with foreground traffic on the same disk, and slow state migration during rescaling. As of the Flink 2.0/2.1/2.2 line in 2026 this is an actively-evolving area (a Rust-native rewrite of ForSt is in progress in the community) rather than a fully mature default — verify ForSt's stability status for your Flink version before treating it as a drop-in RocksDB replacement.

---

## Comparative: vs. Spark Structured Streaming

*(Spark's core batch engine — Catalyst, Tungsten, shuffle, AQE — is covered in `compute/spark.md`; there is no dedicated Structured Streaming doc in this KB yet, so this comparison is against the micro-batch model layered on that batch engine, not a peer streaming-engine doc. The comparison sharpens once such a doc exists.)*

| Dimension | Flink | Spark Structured Streaming |
|---|---|---|
| Execution model | Continuous, per-record/per-buffer dataflow | Micro-batch: each trigger interval runs a short batch job over accumulated input |
| Latency floor | Single-digit milliseconds typical (bound by buffer flush timeout, network) | Bound by the trigger interval — even "continuous processing mode" (an experimental low-latency Spark mode) has not become the mainstream default; standard micro-batch triggers are seconds-scale |
| Throughput ceiling | High, but per-record dispatch overhead is real | Batching amortizes per-record overhead across a whole micro-batch, giving a generally higher throughput ceiling for the same hardware at the cost of latency |
| Exactly-once mechanism | Chandy-Lamport-style checkpoint barriers — asynchronous, no stop-the-world | Batch-commit — each micro-batch's output is atomically committed as a unit (e.g. write-ahead log + idempotent sink writes), simpler to reason about because "the batch" is a natural transactional boundary |
| Failure recovery unit | Resume from last completed checkpoint's per-operator state snapshot | Re-run/resume the in-flight micro-batch from its input offsets — coarser-grained but conceptually simpler |
| Unified batch/streaming | Table API/SQL genuinely share a single planner and unified relational semantics; DataStream API supports an explicit `BATCH` execution mode for bounded input with different join/shuffle strategies; the pre-1.12 DataSet API was removed entirely in 2.0 | Structured Streaming already unified around DataFrame/Dataset semantics from the start — the same query text can target a batch `read`/`write` or a streaming `readStream`/`writeStream`, reusing Catalyst/Tungsten underneath either way |

The core differentiator to hold onto: Flink pays the barrier-alignment/checkpoint-coordination cost (Signature Mechanism above) specifically so it never has to introduce a batch boundary — this buys a lower latency floor. Spark Structured Streaming accepts the batch boundary as its unit of both execution and consistency, buying a higher throughput ceiling per unit of engineering complexity and a fault-tolerance story that's a natural extension of Spark's existing batch-commit semantics rather than a distinct distributed-snapshot protocol.

---

## Key Gotchas

- **Checkpoint barrier alignment stalls under backpressure**: an operator with multiple input channels must buffer records from any channel whose barrier arrived early, waiting on the slowest channel — if that slow channel is itself backpressured (e.g. a downstream sink can't keep up), alignment buffering grows unbounded and checkpoint duration balloons, sometimes past the checkpoint timeout, causing checkpoint failures under exactly the load conditions where you need checkpointing to succeed most. **Unaligned checkpoints** (`execution.checkpointing.unaligned.enabled=true`) sidestep this by snapshotting in-flight buffered records themselves rather than waiting for alignment — faster checkpoints under backpressure, at the cost of larger checkpoint state size (buffered in-flight data now counts as snapshotted state).
- **Watermark misconfiguration cuts both ways**: too-conservative a watermark (large `forBoundedOutOfOrderness` slack, or a slow/stalled partition dragging the min-watermark down per the walkthrough above) means windows sit open far longer than necessary, inflating end-to-end latency for on-time data waiting on a straggler. Too-aggressive a watermark (tight or zero slack) causes legitimately-in-order-but-slightly-delayed events to be classified as late and dropped (or diverted to a side output, if configured) — there is no single correct setting, only a tunable latency-vs-completeness tradeoff that must match the actual out-of-order characteristics of the real source.
- **RocksDB disk I/O becomes the bottleneck for very large keyed state with poor locality**: as detailed in `data-structures/rocksdb.md`'s "Random Access Pattern Problem," keyed state accessed in a non-locality-friendly order (e.g. events arriving in random key order against state far larger than the block cache) drives a high block-cache-miss rate, turning every state access into a disk read — this shows up as rising per-record processing latency and falling throughput well before it shows up as an obvious error.
- **Savepoints and checkpoints are not the same durability mechanism, and confusing them breaks upgrades**: checkpoints are automatic, lightweight (especially incremental RocksDB checkpoints), and tied to Flink's own internal recovery — they are not intended as a stable, portable format across job code changes. **Savepoints** are explicitly triggered, self-contained snapshots meant for planned operations (job upgrades, rescaling, migrating between clusters) and are Flink's officially supported cross-version-compatible format for that purpose:
  ```bash
  # Trigger a savepoint before upgrading job code
  flink savepoint 3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c s3://my-bucket/savepoints/

  # Stop the job, taking a final savepoint atomically (safer than savepoint-then-cancel)
  flink stop --savepointPath s3://my-bucket/savepoints/ 3f9a1b2c4d5e6f7a8b9c0d1e2f3a4b5c

  # Restart the upgraded job FROM that savepoint
  flink run --fromSavepoint s3://my-bucket/savepoints/savepoint-3f9a1b-abc123 \
    --class com.example.ClickCounterJobV2 ./click-counter-2.0.jar
  ```
  Restoring a savepoint after renaming/reordering stateful operators without assigning explicit **UIDs** (`.uid("window-op")` on each stateful operator) silently breaks state restoration — Flink's default operator IDs are derived from the topology, so a code change that alters generated operator IDs (even one that's semantically a no-op) can make an old savepoint fail to map cleanly onto the new job graph. Assign explicit UIDs to every stateful operator before ever taking a savepoint you intend to restore from later.

---

*Grounded against flink.apache.org release/download pages and FLIP-423 (VLDB 2026 paper on disaggregated state management) as of September 2026. Current release line: Flink 2.x, with 2.3.0 (2026-06-25) the latest stable minor and 2.2.1/2.1.3/2.0.2 patch releases also shipping in 2026; the legacy 1.x line's DataSet API and `RocksDBStateBackend`/`MemoryStateBackend` classnames were removed entirely in Flink 2.0 (FLINK-36323). Disaggregated state via the ForSt state store (FLIP-423) is real and shipping but still actively evolving — including an in-progress Rust-native rewrite — so treat its exact stability/GA status as something to re-verify against your specific Flink version rather than assumed-mature. Re-verify exact patch version before citing a specific number in a deliverable.*
