# Debezium

## 30-Second Intuition

Debezium is a change data capture (CDC) platform that turns row-level database changes — inserts, updates, deletes — into a stream of events, by tailing the *same replication log the database already uses to keep its own replicas in sync* (Postgres WAL via logical decoding, MySQL binlog, SQL Server CDC-capture tables/transaction log, MongoDB oplog), rather than polling tables. It usually runs as a source connector inside Kafka Connect, publishing one Kafka topic per captured table, though it can also run connector-less as Debezium Server (streams straight to Kinesis/Pub/Sub/Pulsar/other sinks, no Kafka Connect cluster) or embedded as the Debezium Engine (a library inside your own JVM app). The one fact that matters operationally: **Debezium's reliability is fundamentally borrowed from — and capped by — the source database's replication log retention.** A Postgres replication slot or a MySQL binlog is finite disk space on the *source* database; if Debezium falls behind or goes down for too long, the DB either can't recycle that log (risking its own disk exhaustion) or does recycle it out from under Debezium (forcing a full re-snapshot). Latest stable line as of September 2026: Debezium 3.6 (3.6.1.Final, August 2026), with 3.7 in alpha.

---

## Resource-Layer Map

| Layer | Role Debezium plays | What it's optimizing for |
|---|---|---|
| CPU | Light — log parsing/decoding and JSON/Avro serialization, no query execution | Debezium doesn't run SQL against the source; it deserializes a binary log format (WAL records, binlog events) into structured change events. CPU scales with change *volume*, not table size |
| Memory | Small heap footprint per connector task; larger for wide schemas or big transactions | Buffers in-flight transactions (a long-running multi-statement transaction on the source must be held in memory/offloaded until commit, since events are emitted in commit order) and the schema history cache |
| Disk | **Not Debezium's own disk — the source database's replication log disk** | This is the real constraint: Postgres WAL segments pinned by an unconsumed replication slot, or MySQL binlog files pinned by `Executed_Gtid_Set` lag, sit on the *source DB's* disk. Debezium itself is nearly stateless (Kafka Connect's offset topic holds its position, not local disk) |
| Network | Two legs: DB→Debezium (replication stream) and Debezium→Kafka (produce) | Both must keep up with the source's write rate; if the Kafka-facing leg falls behind, backpressure eventually stalls slot/binlog consumption, which is what makes the disk row above take on-call urgency |
| GPU | Not applicable | Debezium is a log-tailing and event-serialization process, not a compute engine |

Compare to Kafka Connect's own JDBC source connector: JDBC-source is CPU/network-light on Debezium's side but pushes *query* load onto the source DB (repeated `SELECT ... WHERE updated_at > ?` scans) — the resource cost that log-based CDC (Debezium) eliminates on the source DB, it pays for instead in "the source DB must retain its replication log for as long as Debezium might lag."

---

## The Signature Mechanism: Log-Based CDC via Native Replication Logs

Every database that supports physical or logical replication already maintains an ordered, durable log of every committed row change — because that's what replicas consume to stay in sync. Debezium's one idea is: **don't build a second change-tracking mechanism, tail the one the database already trusts for its own replication.**

Contrast with the two alternatives it deliberately avoids:

- **Dual writes** (app writes to DB and publishes an event in the same code path, no shared transaction): not atomic — a crash between the two writes loses the event or duplicates it. Debezium reads only from what actually committed to the log, so there is nothing to lose independently.
- **Polling / timestamp-based CDC** (`SELECT * FROM orders WHERE updated_at > :last_poll`): three concrete failure modes, not just "less efficient":
  1. **Misses deletes** — a `DELETE` leaves no row to poll for; you'd need a soft-delete convention (`deleted_at`) everywhere, which most schemas don't have.
  2. **Misses rapid intermediate updates** — if a row is updated twice between two poll intervals, polling only ever sees the final state; log-based CDC emits both.
  3. **Adds recurring query load to the source DB** proportional to poll frequency × table size, competing with the OLTP workload it's supposed to not disturb.

Log-based CDC gives a complete, ordered, low-overhead change stream because the log itself already *is* the database's source of truth for "what changed, in what order" — Debezium adds no new write path and issues no repeated scans; it opens one long-lived read stream.

### Concretely, for Postgres: replication slots + logical decoding

Postgres's WAL (write-ahead log) records every change before it's applied. **Logical decoding** is the Postgres feature that translates raw WAL bytes into a logical stream of row-level changes, via an output plugin — Debezium uses Postgres's built-in `pgoutput` plugin (no third-party plugin install needed since Postgres 10+). A **replication slot** is the server-side bookmark that tracks how far a given consumer (Debezium) has read, and — critically — **tells Postgres it must not recycle any WAL segment the slot hasn't yet consumed.**

```sql
-- On the Postgres source: verify logical decoding is enabled (postgresql.conf)
SHOW wal_level;
-- must return: logical
```

```sql
-- Create a PUBLICATION: the set of tables Postgres will emit logical changes for
CREATE PUBLICATION dbz_publication FOR TABLE inventory.customers, inventory.orders;
```

```sql
-- Create the replication slot Debezium will consume from, using pgoutput
SELECT pg_create_logical_replication_slot('dbz_slot', 'pgoutput');
```

```sql
-- Inspect slot state directly — this is the "is Debezium keeping up" query
SELECT slot_name, active, restart_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_wal_bytes
FROM pg_replication_slots
WHERE slot_name = 'dbz_slot';
```

```
 slot_name |  active | restart_lsn | retained_wal_bytes
-----------+---------+-------------+---------------------
 dbz_slot  |    t    | 2A/3B1C4D80 |           184320512
```

`retained_wal_bytes` growing without bound while `active` is `f` (false) is the single most important health signal in a Debezium-on-Postgres deployment — it means WAL is accumulating on the source DB's disk because nothing is draining the slot.

---

## High-to-Low Walkthrough: A Postgres `UPDATE` to a Kafka Topic

```
Application: UPDATE inventory.customers SET email = 'new@x.com' WHERE id = 1004;
      │
      ▼
Postgres WAL            transaction commits; WAL record written (this is the
(write-ahead log)       source of truth Debezium never queries directly)
      │
      ▼
Logical decoding         pgoutput plugin decodes the WAL record into a logical
(pgoutput plugin)        change: table, before-image (if REPLICA IDENTITY FULL),
                         after-image, transaction commit LSN
      │
      ▼
Replication slot          slot_name='dbz_slot' streams the decoded change over
(dbz_slot, streaming      the Postgres replication protocol (same protocol a
replication protocol)     physical/logical standby would use) to Debezium
      │
      ▼
Debezium Postgres          connector task (running inside Kafka Connect) reads
connector task              the streamed change, maps it to internal Envelope
                            schema (before/after/source/op/ts_ms)
      │
      ▼
Kafka producer              connector's embedded Kafka producer serializes the
(inside Connect worker)      event (JSON or Avro) and sends it keyed by the
                              table's primary key
      │
      ▼
Kafka topic                  topic "dbserver1.inventory.customers", partitioned
                              by key — see kafka.md for partitioning/ISR mechanics,
                              unchanged here; Debezium is just another producer
```

### The same trace, as literal config and output

Kafka Connect connector config (`debezium-connector-postgresql`), submitted via the Connect REST API:

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
  http://connect:8083/connectors/ -d '{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.dbname": "inventory",
    "topic.prefix": "dbserver1",
    "slot.name": "dbz_slot",
    "publication.name": "dbz_publication",
    "plugin.name": "pgoutput",
    "table.include.list": "inventory.customers,inventory.orders",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.inventory"
  }
}'
```

```json
{"name":"inventory-connector","config":{...},"tasks":[],"type":"source"}
```

Check the connector is actually running (not just accepted):

```bash
curl -s http://connect:8083/connectors/inventory-connector/status | jq
```

```json
{
  "name": "inventory-connector",
  "connector": { "state": "RUNNING", "worker_id": "connect:8083" },
  "tasks": [ { "id": 0, "state": "RUNNING", "worker_id": "connect:8083" } ],
  "type": "source"
}
```

Consume the resulting event from the Kafka topic — this is the literal JSON that lands there after the `UPDATE` above:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic dbserver1.inventory.customers --from-beginning --max-messages 1
```

```json
{
  "before": {
    "id": 1004,
    "first_name": "Anne",
    "last_name": "Kretchmar",
    "email": "annek@noanswer.org"
  },
  "after": {
    "id": 1004,
    "first_name": "Anne",
    "last_name": "Kretchmar",
    "email": "new@x.com"
  },
  "source": {
    "version": "3.6.1.Final",
    "connector": "postgresql",
    "name": "dbserver1",
    "ts_ms": 1757690000000,
    "snapshot": "false",
    "db": "inventory",
    "schema": "inventory",
    "table": "customers",
    "txId": 556,
    "lsn": 24023128,
    "xmin": null
  },
  "op": "u",
  "ts_ms": 1757690000123,
  "transaction": null
}
```

Annotated: `before`/`after` are the row images (an `INSERT` has `before: null`; a `DELETE` has `after: null`, followed by a Kafka tombstone record with a `null` value for log-compaction cleanup); `op` is `c` (create), `u` (update), `d` (delete), `r` (read/snapshot); `source.lsn` is the Postgres log sequence number this event was decoded from — the same coordinate space as `restart_lsn` in the slot query above, which is exactly how you'd correlate "how far behind is this topic" against "how much WAL is still pinned."

---

## Deep Internals

### Replication Slots and WAL Retention Risk

This is the gotcha-worthy mechanism, so it earns its own worked scenario. A logical replication slot is a *promise* Postgres makes: "I will not recycle any WAL segment past this slot's `restart_lsn` until you've consumed it." That promise is unconditional — Postgres does not know or care *why* the consumer stopped reading.

```sql
-- Simulate: Debezium connector is stopped/crashed, slot is now inactive
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
```

```
 slot_name |  active | restart_lsn
-----------+---------+-------------
 dbz_slot  |    f    | 2A/3B1C4D80
```

While `active=f`, every subsequent committed transaction still writes new WAL, and none of it can be recycled — WAL accumulates on the source DB's own disk, unbounded, for as long as the connector stays down.

```sql
-- Direct visibility into how much WAL disk this slot alone is pinning
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;
```

```
 slot_name |  wal_retained
-----------+---------------
 dbz_slot  |  14 GB
```

If this exceeds available disk on the Postgres host, Postgres itself can hit `ERROR: could not write to WAL` / stall writes, or in older/less-guarded setups, actually run the disk out — a CDC connector outage becomes a source-database outage. Mitigations, in order of preference:

```sql
-- 1. Alert on retained WAL size directly (Prometheus/Nagios-style check wraps this query)
SELECT slot_name, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots WHERE active = false;
```

```sql
-- 2. If the connector is permanently gone, drop the slot to let Postgres recycle WAL —
--    this loses the connector's position; it must fully re-snapshot on restart
SELECT pg_drop_replication_slot('dbz_slot');
```

```properties
# 3. Cap the blast radius with Postgres's own slot-level safety valve (PG 13+)
# postgresql.conf — abandons a slot automatically past this much retained WAL
max_slot_wal_keep_size = 20GB
```

`max_slot_wal_keep_size` trades "definitely lose the slot's position and require a re-snapshot" for "definitely won't run the source DB's disk out" — pick this over letting an inactive slot retain WAL indefinitely.

### Snapshotting: Initial and Incremental

On first start (no prior offset for that table), Debezium takes a **consistent snapshot** of the current table state before it starts streaming — the replication slot is created at the same transactional point the snapshot query reads from, so no change is missed or double-counted at the handoff. Snapshot mode is a literal connector config:

```json
{
  "snapshot.mode": "initial",
  "snapshot.locking.mode": "minimal"
}
```

`snapshot.locking.mode: minimal` takes a brief table lock only long enough to establish the consistent starting point, then releases it — versus `extended`, which holds locks for the whole snapshot (rarely needed, and risky on large/busy tables since it blocks writers).

The problem with a full initial snapshot on a huge table: it either requires a long-held lock (`extended`) or, without any lock, risks reading an inconsistent view if concurrent writes race the scan. **Incremental snapshotting** (stable since Debezium 1.6+, now the default recommended approach for large/ad-hoc snapshots in 3.x) solves this by chunking the table into primary-key ranges and interleaving each chunk's read with the ongoing streaming — no long lock, and it can pause/resume.

You drive it via a **signal table** — a small table Debezium watches for commands:

```sql
-- One-time: create the signal table Debezium polls for ad hoc commands
CREATE TABLE inventory.dbz_signal (
  id VARCHAR(42) PRIMARY KEY,
  type VARCHAR(32) NOT NULL,
  data VARCHAR(2048) NULL
);
```

```json
{
  "signal.data.collection": "inventory.dbz_signal",
  "incremental.snapshot.chunk.size": "1024"
}
```

```sql
-- Trigger an incremental (re-)snapshot of a specific table on demand
INSERT INTO inventory.dbz_signal (id, type, data)
VALUES ('ad-hoc-1', 'execute-snapshot',
        '{"data-collections": ["inventory.orders"], "type": "incremental"}');
```

```sql
-- Pause an in-progress incremental snapshot (e.g. during a load spike)
INSERT INTO inventory.dbz_signal (id, type, data)
VALUES ('pause-1', 'pause-snapshot', '{}');
```

Each chunk is read with a `SELECT ... WHERE pk > :last_pk ORDER BY pk LIMIT :chunk_size FOR UPDATE` (or the connector-specific equivalent) interleaved with live streaming events for the same rows — Debezium reconciles overlaps by preferring the streamed (newer) event over the snapshot chunk's row for any PK touched by both.

### The Outbox Pattern

Tailing arbitrary application tables for CDC has a real cost: it couples the internal schema of every table to every downstream consumer's expectations, and "what changed" (a row diff) is a weaker signal than "what business event happened" (an order was placed). The **outbox pattern** sidesteps this: instead of publishing events by tailing arbitrary domain tables, the application writes an explicit event to a dedicated `outbox` table **in the same local transaction** as the domain write — CDC then only ever tails this one narrow, stable-schema table.

```sql
CREATE TABLE inventory.outbox (
  id UUID PRIMARY KEY,
  aggregatetype VARCHAR(255) NOT NULL,
  aggregateid VARCHAR(255) NOT NULL,
  type VARCHAR(255) NOT NULL,
  payload JSONB NOT NULL
);
```

```sql
-- Application transaction: domain write + outbox write, atomic together
BEGIN;
  UPDATE inventory.orders SET status = 'SHIPPED' WHERE id = 1004;
  INSERT INTO inventory.outbox (id, aggregatetype, aggregateid, type, payload)
  VALUES (gen_random_uuid(), 'Order', '1004', 'OrderShipped',
          '{"orderId": 1004, "shippedAt": "2026-09-13T10:00:00Z"}');
COMMIT;
```

Debezium's `outbox.event.router` single message transform (SMT) then reshapes each captured outbox row into a properly-topic-routed, properly-keyed event — instead of every row landing in one generic `dbserver1.inventory.outbox` topic:

```json
{
  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.route.by.field": "aggregatetype",
  "transforms.outbox.route.topic.replacement": "outbox.event.${routedByValue}",
  "transforms.outbox.table.field.event.key": "aggregateid",
  "transforms.outbox.table.field.event.payload": "payload"
}
```

This produces a message on topic `outbox.event.Order`, keyed by `1004`, with the JSON payload as the value — downstream consumers see domain events, never raw outbox-table diffs. Still a first-class, widely-used pattern in 2026 (it remains Debezium's own documented reference example for the "atomic dual write" problem, distinct from a raw-table CDC feed).

### Single Message Transforms (SMTs) Beyond the Outbox Router

SMTs are Kafka Connect's general post-processing hook, and Debezium ships several beyond `EventRouter`:

```json
{
  "transforms": "unwrap",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.unwrap.drop.tombstones": "false",
  "transforms.unwrap.delete.handling.mode": "rewrite"
}
```

`ExtractNewRecordState` ("the unwrap SMT") flattens the before/after/source Envelope into just the `after` row — the common choice when a downstream sink (e.g. a JDBC sink connector or a simple consumer) wants "current row state," not the full CDC envelope, at the cost of losing the before-image and per-change metadata.

```json
{
  "transforms": "route",
  "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
  "transforms.route.regex": "dbserver1\\.inventory\\.(.*)",
  "transforms.route.replacement": "cdc.$1"
}
```

A generic `RegexRouter` renaming `dbserver1.inventory.customers` → `cdc.customers`, independent of any Debezium-specific logic — SMTs chain, so `unwrap` and `route` can both apply to the same connector.

### Delivery Semantics

Debezium's Postgres connector, running inside Kafka Connect, only commits its replication slot's LSN offset (to Kafka Connect's internal offset topic) **after** the corresponding record is acknowledged by Kafka — so a crash between "read from WAL" and "offset committed" replays that segment of WAL on restart. Net effect: **at-least-once delivery** — consumers must be idempotent (dedupe on `source.lsn` + `source.txId`, or on primary key + `op`, depending on need) rather than assume exactly-once. Debezium does not claim exactly-once end-to-end by default; Kafka's own idempotent producer (`enable.idempotence=true`, default since Kafka 3.0 for new producers) removes *duplicate publishes from retries* on the Debezium→Kafka leg specifically, but does not make the whole DB→Kafka pipeline exactly-once.

```json
{
  "producer.override.enable.idempotence": "true",
  "producer.override.acks": "all"
}
```

Per-connector producer overrides like these tune the Debezium→Kafka leg's durability (`acks=all` — see kafka.md's ISR discussion) independently of the worker's cluster-wide producer defaults.

### Offset and Schema History Tracking

Kafka Connect (not Debezium itself) persists the connector's position — for Postgres this is the WAL LSN, for MySQL the binlog file+position/GTID — into an internal Kafka topic:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic connect-offsets --from-beginning --max-messages 5 \
  --property print.key=true
```

```
["inventory-connector",{"server":"dbserver1"}]	{"lsn":24023128,"txId":556,"ts_usec":1757690000123000}
```

Debezium's schema history topic (`schema-changes.inventory` in the config above) is separate — it replays DDL Debezium has observed so the connector can reconstruct the exact table schema that was in effect for any historical event on restart, which matters for **schema evolution**: an `ALTER TABLE ... ADD COLUMN` on the source is captured from the WAL/binlog stream itself (Postgres logical decoding and MySQL binlog both include DDL-adjacent metadata Debezium parses), not queried out-of-band — but a DDL change that isn't cleanly representable (e.g. a column type change incompatible with in-flight Avro/JSON Schema evolution rules on the Kafka Schema Registry) can still break serialization for a topic already in use, requiring a coordinated schema-registry compatibility check before applying the DDL.

---

## Comparative: vs. Polling-Based CDC and vs. the JDBC Source Connector

| | Debezium (log-based) | Polling CDC / JDBC Source Connector |
|---|---|---|
| Completeness | Every insert/update/delete, in commit order, including rapid intermediate updates | Misses deletes (no row left to poll), misses updates that happen and are overwritten between polls |
| Source DB load | One long-lived streaming read connection; no repeated scans | Recurring `SELECT ... WHERE updated_at > ?` queries at poll interval — real, sustained query load, worse as table size or poll frequency grows |
| Ordering | Total order per source transaction log (per-table order guaranteed; use the `transaction` metadata field for cross-table transaction boundaries) | Only as good as the poll query's `ORDER BY`; concurrent writes during a poll window can be missed or reordered relative to true commit order |
| Source DB coupling | Must retain replication log until consumed (the WAL-retention gotcha above) | No log retention coupling — but requires an indexed, monotonically increasing "last modified" column on every polled table, which not all schemas have |
| Operational footprint | Kafka Connect cluster (or Debezium Server) + one replication slot/binlog reader per source DB | Just a scheduled query — often the "quick to stand up" choice, at the cost of the completeness gaps above |

The Kafka Connect JDBC source connector is not a lesser Debezium — it's solving a genuinely different problem (works against any JDBC-reachable DB with zero source-side replication setup) and is the right tool when: the source can't expose logical replication (some managed DB tiers restrict it), the table has a reliable `updated_at`/incrementing key and deletes are rare or handled via soft-delete, or the completeness gap is acceptable for the use case (e.g. periodic reporting sync). It is the wrong tool when the consumer needs a complete, low-latency, delete-inclusive change stream — which is Debezium's whole reason to exist.

---

## Key Gotchas

- **Replication slot WAL retention can take down the source database, not just Debezium**: a stopped, crashed, or too-slow-consuming Debezium connector leaves its Postgres slot `active=false` (or lagging), and Postgres will not recycle WAL past that slot's position — disk on the *source* DB fills up. Set `max_slot_wal_keep_size` as a hard cap and alert on `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` directly; don't rely on noticing lag downstream in Kafka.
- **Schema evolution on the source table can break the connector or its serialization**: DDL is captured from the log stream itself, but a change incompatible with the topic's existing Avro/JSON Schema (e.g. a narrowing type change) can break consumers or the connector's own schema-history reconstruction on restart. Coordinate schema-registry compatibility checks before shipping the source DDL, not after.
- **Snapshot locking on large tables is a real production risk if misconfigured**: `snapshot.locking.mode: extended` holds a lock for the whole snapshot duration — on a large, actively-written table this can block application writers for as long as the snapshot takes. Prefer `minimal` locking plus incremental snapshotting for anything non-trivial in size.
- **At-least-once, not exactly-once, is the default delivery guarantee**: a crash between WAL read and Kafka offset commit replays that segment on restart. Downstream consumers must dedupe (on `source.lsn`+`source.txId`, or a business key) rather than assume no duplicates.
- **Offsets live in Kafka Connect's internal `connect-offsets` topic, not in Debezium**: losing or corrupting that topic (or migrating a connector to a new Connect cluster without preserving it) means the connector has no memory of its position and must fall back to a fresh snapshot — treat `connect-offsets`, `connect-configs`, and the schema-history topic with the same durability care as any other critical Kafka topic (adequate replication factor, no accidental deletion).
- **A deleted row produces two Kafka records, not one**: the delete event itself (`op: "d"`, `after: null`) followed by a tombstone (key present, value `null`) for Kafka log-compaction cleanup — a consumer that only expects one record per logical change will double-count deletes if not handled explicitly.

---

*Grounded against debezium.io release notes and documentation (architecture, Debezium Server, signalling/incremental-snapshots, outbox-event-router transformation pages) as of September 2026. Current stable line: Debezium 3.6 (3.6.1.Final, August 4 2026; 3.7.0.Alpha1 in development). Re-verify the exact patch version before citing it in a deliverable — the 3.x line is releasing roughly every 1-2 months.*

**Open items flagged for your confirmation:**
- "Exactly-once" framing: some third-party blogs describe end-to-end exactly-once as achievable with idempotent consumers + Kafka idempotent producer; this doc follows Debezium's own at-least-once-by-default framing and treats "exactly-once" as a consumer-side property (dedup), not a Debezium guarantee. Confirm this matches how your pipelines actually treat delivery semantics.
- Debezium 3.6's OpenTelemetry-based pipeline monitoring dashboard (throughput/replication lag/snapshot progress) is new enough (this release cycle) that I did not verify it against a live deployment — flagging in case your environment is still on an earlier 2.x/3.x line where it doesn't exist.
