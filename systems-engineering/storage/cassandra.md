# Apache Cassandra

## 30-Second Intuition

Cassandra is a wide-column, **masterless** distributed database built for one thing above all else: sustained write throughput at linear horizontal scale, with no single point of failure and no leader to become a bottleneck. Every node is a peer (a "coordinator" for whatever request it happens to receive), data is placed on a consistent-hash ring (see `distributed-systems/consistent-hashing.md` — Cassandra is literally cited there as the production example of vnodes-on-a-ring), and the storage engine underneath each node is an LSM-tree (see `data-structures/lsm-trees.md` for the compaction/amplification mechanics — this doc doesn't re-derive them). Consistency is not a fixed property of the database, it's a **per-request dial**: every read and write specifies a consistency level (`ONE`, `QUORUM`, `ALL`, ...) trading latency and availability against durability guarantees, which is exactly the concrete instantiation `distributed-systems/consistency-models.md` gestures at when it lists Cassandra as "AP by default, tunable per-query." The one fact that matters most operationally: **Cassandra is a distributed hash table with a query language bolted on, not a distributed SQL engine** — every table's primary key design has to be chosen to match the queries you'll run against it, because a query without an equality predicate on the partition key does not do what you think it does (see Gotchas).

Current stable line as of September 2026 is **Cassandra 5.0** (GA'd September 2024; latest bug-fix release 5.0.9, August 2026), which shipped **Storage-Attached Indexes (SAI)**, a native **vector data type + ANN indexing** for AI/similarity search workloads, and the **Unified Compaction Strategy (UCS)**. **Cassandra 5.1** is in active development (performance/operational refinements, SAI maturation, JDK 21 support landing in production clusters through 2026); the headline next-generation feature — **Accord**, a leaderless consensus protocol enabling general-purpose ACID transactions across arbitrary keys — shipped experimentally behind a feature flag starting in 5.0, with full production-grade availability now targeted for **Cassandra 6.0** (in alpha as of 2026), not 5.1 as originally floated. Treat exact Accord GA timing as an open item to reconfirm against the release actually being deployed.

---

## Resource-Layer Map

| Layer | Role Cassandra plays | What it's optimizing for |
|---|---|---|
| Disk | Every write is a **sequential commit-log append** followed by an in-memory memtable insert — no random write ever touches the data files directly. Data files (SSTables) are immutable once flushed; the only other disk cost is **background compaction** (merging SSTables, rewriting data that's already durable) — see `data-structures/lsm-trees.md` for the write/read/space-amplification tradeoffs this implies. Disk is Cassandra's dominant cost center: the write path is nearly free (sequential), but compaction I/O and the read path (potentially checking multiple SSTables per partition) are where disk actually gets spent | Turn all writes into sequential I/O; push all the expensive, mergeable work (compaction) onto a background process that can be rate-limited and doesn't block the client |
| Memory | **Memtables** (one active, per-table, sorted, flushed to an SSTable when full), **bloom filters** (one per SSTable, off-heap, answer "definitely not in this SSTable" for point reads), and optional **key cache** (partition-key → row-index-position) and **row cache** (whole deserialized partitions, off by default — expensive per entry, only worth it for very hot, small partitions). Memory is what keeps a point read from becoming a disk seek per candidate SSTable | Avoid disk entirely for the common read case (bloom filter says no, or key/row cache already has it) and absorb write bursts before they have to become I/O |
| Network | **Gossip** (peer-to-peer heartbeat + state exchange, every node talks to a few random peers each second, no central membership authority) for cluster membership/failure detection, plus **hinted handoff** (coordinator stashes writes for a temporarily-down replica) and **read repair** (background reconciliation of replicas that disagree) traffic. There is no leader election traffic, no raft/paxos log replication for ordinary reads/writes — network cost is proportional to replication factor and consistency level, not to a leader's fan-out | Keep membership and replica-consistency traffic peer-to-peer and asynchronous wherever the requested consistency level allows it, so no single node's network link becomes a chokepoint |
| CPU | Mostly **not** the bottleneck relative to disk/network for typical workloads — CQL parsing, serialization, and compaction's merge-sort work are real but rarely the limiting resource. The one CPU-adjacent knob that matters operationally is the **consistency level**: `ONE` returns after the fastest replica responds (least CPU/network coordination), `QUORUM`/`ALL` require the coordinator to wait on and reconcile multiple replica responses before answering, which is coordination overhead, not raw CPU | Spend the minimum coordination work the requested consistency level actually requires — CPU scales with how many replicas you insist on hearing from, not with dataset size |
| GPU | Not applicable to the core write/read/compaction path — Cassandra 5.0's vector search (ANN over the new vector data type) runs its index build and similarity search on CPU; there is no GPU execution path in the OSS engine as of 5.0/5.1 | N/A |

This table is the mechanical explanation for why Cassandra and DynamoDB (both Dynamo-paper descendants, see Comparative below) still diverge operationally: Cassandra hands the operator a real, inspectable disk/compaction cost surface (`nodetool compactionstats`, SSTable counts, `gc_grace_seconds`) because you run the nodes yourself; DynamoDB hides that entire layer behind a managed-capacity abstraction. Same resource-layer shape underneath (LSM-tree, consistent hashing, tunable consistency), radically different operational surface on top.

---

## The Signature Mechanism: Masterless Architecture via Consistent Hashing + Gossip, with Per-Request Tunable Consistency

Cassandra has no primary/leader node for ordinary reads and writes — **any node can act as coordinator** for any request from any client, hashes the request's partition key onto a token ring (see `distributed-systems/consistent-hashing.md`), forwards it to the replicas that actually own that token range, and reconciles their responses according to whatever consistency level the client asked for. There is no single node whose failure blocks writes to the rest of the cluster, and no leader-election protocol in the ordinary read/write path (compare directly to etcd/Raft or Spanner, which route every write through a leader/TrueTime-ordered path — see `coordination/etcd.md` and `distributed-systems/consistency-models.md`). Membership itself is discovered and maintained via **gossip**: every node exchanges state (`up`/`down`, schema version, load, token ownership) with a few random peers roughly once per second, so the entire cluster's membership view converges without any node being told the full membership list authoritatively by a central service.

Layered on top of "no leader" is **tunable consistency**: every single read or write carries its own consistency level, independent of every other request against the same table.

```sql
-- Per-statement consistency level in cqlsh
CONSISTENCY QUORUM;
SELECT * FROM ecommerce.orders WHERE customer_id = 42;

CONSISTENCY ONE;
SELECT * FROM ecommerce.orders WHERE customer_id = 42;   -- same table, weaker read, lower latency
```

This is the concrete answer to `distributed-systems/consistency-models.md`'s abstract "linearizable → sequential → causal → eventual" spectrum: Cassandra doesn't implement one point on that spectrum, it lets the caller pick, per query, roughly where between `ONE` (closest to eventual/lowest latency) and `ALL` (closest to strongly consistent, at the cost of availability if any replica is down) they want to sit. `QUORUM` reads/writes on both sides of a request are what push Cassandra toward the "read-your-writes"/strongly-consistent end of that spectrum without ever electing a leader — see the quorum math worked out in Deep Internals below.

Everything else in this doc — replica placement, hinted handoff, read repair, the R+W>N math — is this one mechanism (masterless ring + gossip + tunable CL) worked out in detail.

---

## High-to-Low Walkthrough: One `INSERT` at `QUORUM`, Client to Bytes

The literal statement, run in `cqlsh`:

```sql
CREATE KEYSPACE ecommerce
  WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': 3};

USE ecommerce;

CREATE TABLE orders_by_customer (
    customer_id  int,
    order_id     timeuuid,
    order_total  decimal,
    status       text,
    PRIMARY KEY (customer_id, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);

CONSISTENCY QUORUM;
INSERT INTO orders_by_customer (customer_id, order_id, order_total, status)
VALUES (42, now(), 129.99, 'PLACED');
```

```
cqlsh (any node, client's chosen contact point)
      │
      ▼
Coordinator selection      The node the client's driver happened to connect
                           to becomes THIS request's coordinator — no fixed
                           "the" coordinator for the table, just whichever
                           node received this particular request
      │
      ▼
Partitioner hashes the     Murmur3Partitioner (default) hashes customer_id=42
partition key to a token   → a 64-bit signed token, e.g. -3242571452517099313
      │
      ▼
Ring lookup: which nodes   Coordinator consults its gossiped view of the ring
own this token, RF=3       to find the node whose token range contains this
                           token (the "primary" replica for this token) plus
                           the next RF-1 distinct nodes walking clockwise
      │
      ▼
Coordinator sends the      Write sent in parallel to all 3 replicas
write to all 3 replicas    (not just the primary) — this is unconditional;
in parallel                RF is about durability, CL is about how many
                           acks the coordinator waits for before replying
      │
      ▼
Each replica: commit-log   Sequential append to that node's commit log
append (sequential disk)  (durability, survives crash before memtable flush)
      │
      ▼
Each replica: memtable     Row inserted into the in-memory memtable for
insert                    orders_by_customer — no disk read/seek here
      │
      ▼
QUORUM = ceil((RF+1)/2)    Coordinator waits for 2 of the 3 replicas to ack
        = 2 of 3 acks      (QUORUM at RF=3), NOT all 3 — the 3rd ack (or a
                           hint, if that replica is down) still happens,
                           just not on the client's latency path
      │
      ▼
Client receives            Write is now durable per QUORUM's guarantee:
acknowledgment              any future QUORUM read is mathematically
                            guaranteed to overlap at least one replica
                            that has this write (see R+W>N below)
      │
      ▼
(async, off critical path)
Background: memtable       Eventually flushed to an immutable SSTable
flush + compaction         (sequential write); background compaction later
                            merges SSTables per the table's compaction
                            strategy (see data-structures/lsm-trees.md)
```

If one of the 3 replicas is down when the coordinator sends the write, the coordinator writes a **hint** locally (or on any live replica) recording "this write is owed to node X" and replays it once X rejoins — this is **hinted handoff**, and the write still succeeds at `QUORUM` as long as 2 of the 3 replicas actually ack.

**Confirming ring membership and token ownership** — `nodetool status` and `nodetool ring`:

```bash
nodetool status ecommerce
```

```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.0.1.11     412.4 KiB  16      33.4%              a1b2c3d4-...                          rack1
UN  10.0.1.12     398.1 KiB  16      33.1%              e5f6a7b8-...                          rack1
UN  10.0.1.13     405.7 KiB  16      33.5%              c9d0e1f2-...                          rack1
```

`UN` = Up/Normal. `Tokens: 16` means each physical node owns 16 vnodes on the ring (Cassandra's current default, down from the historical 256-per-node default — see `distributed-systems/consistent-hashing.md`'s vnode-count gotcha for why). `Owns (effective)` accounts for replication factor, not raw token-range share.

```bash
nodetool ring ecommerce
```

```
Datacenter: datacenter1
==========
Address       Rack        Status State   Load            Owns    Token
                                                                  -3255831813396456422
10.0.1.12     rack1       Up     Normal  398.1 KiB       33.1%   -3242571452517099313
10.0.1.13     rack1       Up     Normal  405.7 KiB       33.5%   -1102938475610283746
10.0.1.11     rack1       Up     Normal  412.4 KiB       33.4%    1029384756102938475
...
```

The row whose `Token` value is the first one **≥** the write's computed token (`-3242571452517099313` here) is that write's primary replica — this is the literal ring-lookup step from the walkthrough above, made visible.

---

## Deep Internals

### Partition Key vs. Clustering Key — Query-First, Then Model

Unlike a relational schema (design the tables first, then write whatever query you need against them), Cassandra requires **query-first modeling**: you decide the exact query patterns your application needs, then design one table per query pattern, because the partition key is not just an index choice — it's the physical unit of data placement and the *only* efficient way to reach a row.

```sql
-- Query pattern: "give me all orders for customer 42, newest first"
-- → partition key = customer_id (co-locates all of one customer's orders on
--   the same node/replica set), clustering key = order_id (sorts within
--   the partition, DESC for newest-first with zero extra work at read time)
CREATE TABLE orders_by_customer (
    customer_id  int,
    order_id     timeuuid,
    order_total  decimal,
    status       text,
    PRIMARY KEY (customer_id, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);

-- Different query pattern: "give me all orders with status='PLACED' across
-- ALL customers" → the table above CANNOT serve this efficiently (it would
-- have to scan every partition on every node). This needs a SEPARATE table,
-- denormalized, with status baked into the partition key:
CREATE TABLE orders_by_status (
    status       text,
    order_id     timeuuid,
    customer_id  int,
    order_total  decimal,
    PRIMARY KEY (status, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);

-- Same underlying data, written twice (once per query pattern) at INSERT
-- time — this dual-write is the normal, expected cost of query-first
-- modeling, not a workaround
BEGIN BATCH
  INSERT INTO orders_by_customer (customer_id, order_id, order_total, status)
    VALUES (42, now(), 129.99, 'PLACED');
  INSERT INTO orders_by_status (status, order_id, customer_id, order_total)
    VALUES ('PLACED', now(), 42, 129.99);
APPLY BATCH;
```

The reason this is mandatory rather than a style preference: a partition is the atomic unit Cassandra hashes onto the ring. Reaching a partition means hashing its partition key and going to the (RF) nodes that own that token — there is no secondary global index that makes an arbitrary-column query as cheap as a partition-key lookup (SAI narrows this gap for some cases — see below — but doesn't eliminate the need to model around your access pattern for high-throughput paths).

### Tunable Consistency and the R+W>N Quorum Math (RF=3 Worked Example)

With replication factor `N=3`, `QUORUM` is defined as `⌈(N+1)/2⌉ = 2`. The classic strong-consistency condition is **R + W > N**, where `R` is the read consistency level's replica count and `W` is the write consistency level's replica count: if every read set and every write set are guaranteed to overlap by at least one replica, a read can never fail to see the most recent completed write.

```sql
-- Write at QUORUM (W=2), read at QUORUM (R=2), N=3
CONSISTENCY QUORUM;
INSERT INTO orders_by_customer (customer_id, order_id, order_total, status)
VALUES (42, now(), 45.00, 'PLACED');

CONSISTENCY QUORUM;
SELECT * FROM orders_by_customer WHERE customer_id = 42;
-- R(2) + W(2) = 4 > N(3)  →  guaranteed overlap: this read is guaranteed
-- to see the write above, because any 2-of-3 write set and any 2-of-3
-- read set on a 3-replica group must share at least one replica
```

```
All possible 2-of-3 subsets of {replica-A, replica-B, replica-C}:
  {A,B}  {A,C}  {B,C}
Any two subsets from this list always share at least one member —
there is no way to pick two disjoint pairs from a 3-element set.
```

Contrast with `W=1` (`ONE`), `R=1` (`ONE`), `N=3`: `1+1=2`, which is **not** `> 3` — a write acknowledged by replica A alone and a read served entirely from replica C (which never got the write yet) is a completely valid outcome. This is the numeric shape of "CL=ONE risks stale reads" from the Gotchas section, made concrete: it's not a vague warning, it's this specific arithmetic failing.

```sql
-- Common tunable combinations and what they cost/guarantee at RF=3
CONSISTENCY ONE;      -- R=1 or W=1: lowest latency, weakest guarantee, tolerates 2 nodes down
CONSISTENCY QUORUM;   -- R=2 or W=2: R+W>N when paired with QUORUM on the other side, tolerates 1 node down
CONSISTENCY ALL;      -- R=3 or W=3: strongest per-request guarantee, zero fault tolerance (any replica down blocks the request)
CONSISTENCY LOCAL_QUORUM;  -- QUORUM computed within the local datacenter only — avoids cross-DC latency on the coordination path
```

### Gossip Protocol Basics

```bash
# See this node's live gossip state about every peer it knows about —
# generation number, heartbeat version, application state (load, status,
# schema version, tokens, DC/rack)
nodetool gossipinfo
```

```
/10.0.1.12
  generation:1700000012
  heartbeat:284913
  STATUS:NORMAL,-3242571452517099313
  LOAD:398134.0
  SCHEMA:e1cba99b-...
  DC:datacenter1
  RACK:rack1
  RELEASE_VERSION:5.0.9
```

Gossip exchanges are peer-to-peer and probabilistic (each node picks ~1-3 random peers per round, roughly once per second) rather than a full broadcast — cluster-wide membership convergence time grows logarithmically with cluster size, not linearly with a central coordinator's fan-out. `nodetool describecluster` surfaces the same underlying gossip-derived state in a more digestible form, including schema-version agreement, which is the most common gossip-visible symptom of a cluster mid-rolling-upgrade or with a stuck node:

```bash
nodetool describecluster
```

```
Cluster Information:
        Name: ecommerce_cluster
        Snitch: org.apache.cassandra.locator.SimpleSnitch
        DynamicEndPointSnitch: enabled
        Partitioner: org.apache.cassandra.dht.Murmur3Partitioner
        Schema versions:
                e1cba99b-...: [10.0.1.11, 10.0.1.12, 10.0.1.13]
```

A cluster with more than one schema version listed means at least one node hasn't converged on the latest schema change yet — worth checking before assuming a `CREATE TABLE`/`ALTER TABLE` has fully propagated.

### Read Path: Digest Reads, Read Repair, and Speculative Retry

A `QUORUM` read doesn't naively fetch the full row from every contacted replica — the coordinator sends one **data request** (full row) to one replica and **digest requests** (hash of the row) to the others, to save network bandwidth, then compares digests:

```sql
CONSISTENCY QUORUM;
TRACING ON;
SELECT * FROM orders_by_customer WHERE customer_id = 42;
```

```
 activity                                                                | source     | source_elapsed
--------------------------------------------------------------------------------------------------------
 Parsing SELECT * FROM orders_by_customer WHERE customer_id = 42        | 10.0.1.11  |               0
 Sending READ message to /10.0.1.12                                     | 10.0.1.11  |             112
 Sending READ_DIGEST message to /10.0.1.13                               | 10.0.1.11  |             114
 Executing single-partition query on orders_by_customer                 | 10.0.1.12  |             340
 Digest mismatch detected -- initiating read-repair                     | 10.0.1.11  |            1204
 Read repair: sending write to /10.0.1.13                                | 10.0.1.11  |            1250
Request complete
```

If the digests disagree (one replica is behind), the coordinator triggers **read repair**: it reconciles the mismatch (last-write-wins by timestamp for regular columns) and pushes the correct value to the stale replica(s) — traffic that only exists because a `QUORUM`/`ALL` read happened to notice a divergence. This is separate from **hinted handoff** (which repairs a replica that was fully unreachable at write time) and from **anti-entropy repair** (`nodetool repair`, a full, explicit reconciliation pass, typically run on a schedule to catch divergence that neither hints nor read-repair happened to touch):

```bash
# Full anti-entropy repair for a keyspace — merkle-tree comparison across
# replicas, typically scheduled weekly (must run within gc_grace_seconds,
# see tombstone gotcha below)
nodetool repair ecommerce

# Check hint backlog on this node (undelivered writes owed to down replicas)
nodetool statushandoff
nodetool tpstats | grep HintsDispatcher
```

### Compaction Strategy Selection (mechanics in `data-structures/lsm-trees.md`)

Cassandra ships three main compaction strategies plus the newer Unified Compaction Strategy (UCS, 5.0+); the general write/read/space-amplification tradeoffs are covered generically in `data-structures/lsm-trees.md` — this section is only the Cassandra-specific "which one, for which table" guidance:

```sql
-- SizeTieredCompactionStrategy (STCS) — default historically, good for
-- write-heavy, no-TTL workloads; higher space amplification (see lsm-trees.md)
ALTER TABLE orders_by_customer WITH compaction =
  {'class': 'SizeTieredCompactionStrategy'};

-- LeveledCompactionStrategy (LCS) — better read amplification/predictable
-- latency for read-heavy tables with lots of updates/overwrites to the
-- same rows; pays more write amplification (see lsm-trees.md's leveled row)
ALTER TABLE orders_by_customer WITH compaction =
  {'class': 'LeveledCompactionStrategy'};

-- TimeWindowCompactionStrategy (TWCS) — for time-series/TTL'd data written
-- once and rarely updated (metrics, events, logs): groups SSTables into
-- time buckets so whole buckets can be dropped wholesale once their
-- TTL/window expires, avoiding compacting data that's about to be deleted
-- anyway
ALTER TABLE sensor_readings WITH compaction =
  {'class': 'TimeWindowCompactionStrategy', 'compaction_window_size': '1', 'compaction_window_unit': 'DAYS'};

-- UnifiedCompactionStrategy (UCS, Cassandra 5.0+) — a single configurable
-- strategy that can emulate STCS- or LCS-like behavior via a "scaling
-- parameter," intended to reduce the need to choose and re-tune between
-- the three legacy strategies as a table's workload shifts over time
ALTER TABLE orders_by_customer WITH compaction =
  {'class': 'UnifiedCompactionStrategy'};
```

Guidance in one line each: STCS for generic write-heavy tables without TTLs, LCS for read-heavy tables with frequent overwrites of the same keys, TWCS for append-mostly time-series with TTL-based expiry, UCS when you don't want to commit to picking between the first three upfront (it's newer — verify current recommended defaults for your workload against the deployed 5.0.x/5.1 release notes before standardizing on it fleet-wide).

### Accord: General-Purpose ACID Transactions (Newer, Verify Against Deployed Version)

Accord is a leaderless, EPaxos-family consensus protocol designed to give Cassandra genuine multi-key, serializable ACID transactions — a capability the classic Paxos-based lightweight transactions (`IF NOT EXISTS`/CAS, single-partition only) never provided. Unlike Paxos-per-operation (which still needs a coordinator round per conditional write and is limited to one partition), Accord is designed to commit transactions spanning **arbitrary keys** without a fixed leader, aiming for a fast path with minimal round trips when replicas agree, falling back to a recovery path when they don't.

```sql
-- Classic Paxos-based lightweight transaction (available today, single
-- partition only, NOT what Accord replaces conceptually so much as extends)
INSERT INTO orders_by_customer (customer_id, order_id, order_total, status)
VALUES (42, now(), 75.00, 'PLACED')
IF NOT EXISTS;

-- Accord-based general-purpose transaction syntax (experimental/behind a
-- feature flag as introduced in 5.0; treat exact syntax and GA status as
-- something to reverify against the specific 5.x/6.0 release you're on)
BEGIN TRANSACTION
  LET current_total = (SELECT order_total FROM orders_by_customer WHERE customer_id = 42 AND order_id = ?);
  SELECT current_total;
  IF current_total.order_total < 100.00 THEN
    UPDATE orders_by_customer SET status = 'PLACED' WHERE customer_id = 42 AND order_id = ?;
  END IF
COMMIT TRANSACTION;
```

Grounded status as of this writing: Accord shipped experimentally starting with Cassandra 5.0, but full production-grade general-purpose ACID transactions are tracked for **Cassandra 6.0** (in alpha in 2026), not the 5.1 line some earlier community roadmaps floated it for — this is a live-moving target and worth reconfirming against release notes before depending on it in production.

---

## Comparative

### vs. DynamoDB (the other Dynamo-paper descendant)

Cassandra and DynamoDB share direct lineage — both implement the core ideas from Amazon's 2007 Dynamo paper (consistent hashing + vnodes, tunable per-request consistency, gossip-style membership in Cassandra's case) — see `distributed-systems/consistent-hashing.md`'s citation of that paper. Where they diverge is almost entirely in the resource-layer table above: Cassandra is **self-hosted**, so the operator directly sees and tunes the disk/compaction/gossip layer (`nodetool compactionstats`, choosing a compaction strategy, running `nodetool repair`); DynamoDB is **fully managed**, so that entire layer (partitioning, compaction-equivalent internal maintenance, replica placement) is invisible and non-configurable, exposed only through provisioned/on-demand capacity units and a `ConsistentRead` boolean flag rather than a full consistency-level dial. Consequently Cassandra gives you strictly more consistency granularity (`ONE`/`QUORUM`/`LOCAL_QUORUM`/`EACH_QUORUM`/`ALL`, per statement) where DynamoDB gives you a binary choice (eventually consistent vs. strongly consistent reads, same region only). Both are classified `AP`/`EL` by default in `distributed-systems/consistency-models.md`'s PACELC table, and both can be pushed toward the consistent end per-request — Cassandra via `QUORUM`+`QUORUM`, DynamoDB via `ConsistentRead=true` (which is still single-region-scoped, unlike Cassandra's `EACH_QUORUM` which explicitly reasons about multiple datacenters).

### vs. etcd (see `coordination/etcd.md`, `distributed-systems/consistency-models.md`)

Cassandra and etcd sit at opposite ends of the AP/CP framing `distributed-systems/consistency-models.md` uses to classify systems: Cassandra is the canonical **AP** example there (available under partition, tunable toward consistency per-request but never mandatorily so), etcd is the canonical **CP** example (loses quorum → refuses writes entirely, every linearizable read pays a leader-reconfirmation round trip). This is a direct resource-layer consequence, not a coincidence: etcd routes every write through a Raft leader and treats availability as something explicitly sacrificed to guarantee linearizability; Cassandra has no leader in the ordinary path at all and treats strong-consistency reads/writes as an opt-in cost the caller pays per statement via `QUORUM`/`ALL`, not a cluster-wide invariant. Practically: pick etcd (or a Raft/Paxos-based store) when correctness of a small amount of coordination state (leader election, config, locks) must never be ambiguous even at the cost of unavailability during a partition; pick Cassandra when you need to keep accepting writes at scale through partial failures and are willing to reason, per query, about how stale a read is allowed to be.

---

## Key Gotchas

- **Bad partition key choice creates a hotspot, not just an inefficiency**: a partition key with low cardinality or a skewed access pattern (e.g. partitioning by `status` when 90% of rows are `'ACTIVE'`, or by a single tenant ID in a multi-tenant system where one tenant dominates traffic) sends a disproportionate share of reads/writes to the same handful of physical nodes that own that token range — consistent hashing bounds *keyspace* distribution, not *traffic* distribution (see `distributed-systems/consistent-hashing.md`'s identical hot-key gotcha). Detect via `nodetool tablestats` partition-size percentiles and per-node load skew in `nodetool status`; fix by adding a higher-cardinality bucketing column into the partition key (e.g. `(tenant_id, shard_bucket)` instead of `tenant_id` alone).
- **Queries without the partition key are an anti-pattern, not a convenience**: `SELECT * FROM orders_by_customer WHERE status = 'PLACED'` on a table partitioned by `customer_id` has no way to route to a bounded set of nodes — Cassandra requires `ALLOW FILTERING` to even run it, and what that actually does is scan every partition on every node checking the predicate row-by-row. `ALLOW FILTERING` existing as a flag you can add to make the error go away is exactly the trap: it makes a full-cluster scan syntactically legal, not efficient. The real fix is the query-first modeling from Deep Internals — a second table (or a SAI index, for lower-cardinality secondary-predicate cases) built for that access pattern.
  ```sql
  -- This "works" but silently means "scan the whole cluster":
  SELECT * FROM orders_by_customer WHERE status = 'PLACED' ALLOW FILTERING;

  -- SAI narrows this gap for genuinely secondary, lower-throughput queries —
  -- verify current SAI query-planning guidance for your data shape before
  -- relying on it for a high-QPS path
  CREATE INDEX orders_status_idx ON orders_by_customer (status) USING 'sai';
  SELECT * FROM orders_by_customer WHERE status = 'PLACED';  -- no ALLOW FILTERING needed with SAI
  ```
- **Tombstone accumulation from deletes/TTLs slows reads, sometimes catastrophically**: every `DELETE` (and every TTL expiry) writes a tombstone, not an immediate removal — the underlying LSM mechanics are in `data-structures/lsm-trees.md`'s tombstone section. A read that has to skip past thousands of tombstones before finding a live row (common with wide partitions that delete/expire heavily, e.g. a queue-like table) can time out entirely; Cassandra logs a warning past a configurable tombstone-scan threshold and will abort the query past a hard limit.
  ```sql
  -- Per-table tuning: how long a tombstone must live before compaction can
  -- discard it (must exceed your repair cadence — see Deep Internals'
  -- read-repair/anti-entropy-repair note)
  ALTER TABLE orders_by_customer WITH gc_grace_seconds = 864000;  -- 10 days, default

  -- Find tables accumulating tombstones badly
  nodetool tablestats ecommerce.orders_by_customer | grep -i tombstone
  ```
  Detect via `nodetool tablestats` (tombstone/live-cell ratio per read) and server-side warnings in the logs (`Read X live rows and Y tombstone cells`); fix by redesigning away from delete-heavy wide partitions (e.g. TWCS + short TTLs so whole SSTables expire instead of accumulating scattered tombstones) rather than just raising the scan-limit thresholds.
- **`CONSISTENCY ONE` risks stale reads by design, not by bug**: as the R+W>N math above shows numerically, `W=1`/`R=1` at `RF=3` has no overlap guarantee at all — a read immediately after a write can legitimately return the pre-write value if it happens to land on a replica that hasn't received the write yet. This is a valid, intentional tradeoff for latency/availability-sensitive workloads (e.g. metrics ingestion, high-volume event logging) and a correctness bug waiting to happen for anything that needs read-your-writes behavior (session state, inventory counts) — the fix is choosing `QUORUM` (or stronger) on the specific tables/queries that need it, not applying one consistency level cluster-wide (see `distributed-systems/consistency-models.md`'s "picking a consistency level is a per-field decision" gotcha, which applies to Cassandra by name).
- **Unbounded partition growth ("wide partitions") degrades everything at once**: a partition key that never rotates (e.g. `PRIMARY KEY (device_id, reading_time)` for a device that streams forever with no time-bucketing in the partition key) grows a single partition without bound — it fragments across more and more SSTables, compaction has to handle an ever-larger single unit, and a full-partition read gets slower over time even with no design change on the read side. Detect via `nodetool tablestats`' max/mean partition size and Cassandra's own large-partition warning threshold in the logs; fix by adding a time-bucket into the partition key up front (`PRIMARY KEY ((device_id, day_bucket), reading_time)`) so partitions naturally cap in size and old buckets can be dropped/expired wholesale.
  ```sql
  -- Unbounded: one partition per device_id, forever
  CREATE TABLE readings_unbounded (
      device_id int, reading_time timestamp, value double,
      PRIMARY KEY (device_id, reading_time)
  );

  -- Bounded: partition rotates daily, capping partition size regardless of
  -- how long the device has been streaming
  CREATE TABLE readings_bucketed (
      device_id int, day_bucket date, reading_time timestamp, value double,
      PRIMARY KEY ((device_id, day_bucket), reading_time)
  );
  ```

---

*Grounded against the Apache Cassandra project blog ("Announcing Apache Cassandra 5.0", "Apache Cassandra 5.0 Features: Vector Search", "Apache Cassandra 5.0: Moving Toward an AI-Driven Future"), Instaclustr and AxonOps 2025/2026 community writeups on Cassandra 5.0/5.1/6.0 and Accord, and Cassandra's own `nodetool`/CQL documentation for command syntax, as of September 2026. Open/conflicting items worth reconfirming before relying on them operationally: (1) exact current patch version and 5.1 release timing — sources agree on 5.0 GA (September 2024) and 5.0.9 (August 2026) but 5.1's own GA date wasn't pinned down by research; (2) Accord's precise GA vehicle — community sources disagree on whether general-purpose ACID transactions land as a 5.1 feature or are deferred entirely to 6.0 (currently in alpha), and the exact `BEGIN TRANSACTION` CQL syntax shown here is illustrative of the direction, not verified against a shipped, non-experimental release; (3) SAI's exact query-planning guarantees for compound/multi-column predicates should be reverified against the specific 5.0.x/5.1 release in use before treating it as a full substitute for query-first modeling on high-QPS paths.*
