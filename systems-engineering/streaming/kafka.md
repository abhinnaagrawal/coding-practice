# Apache Kafka

## 30-Second Intuition

Kafka is a distributed, replicated, append-only commit log exposed as a publish/subscribe messaging system. Producers append records to partitions (ordered, immutable sequences), consumers read them at their own pace by tracking an offset, and the broker never has to parse or understand message content — it just appends bytes and serves byte ranges back out. The one fact that matters operationally: Kafka's entire performance story is built on **not touching the CPU or user-space memory on the hot path** — reads are served via a kernel-level zero-copy transfer straight from page cache to network socket, so a caught-up Kafka cluster does almost no disk I/O and almost no CPU work per message, even at very high throughput. Everything else (partitioning, replication, consumer groups) is standard distributed-systems machinery; the zero-copy read path is the specific reason Kafka can sustain GB/s-class throughput on commodity hardware.

---

## Resource-Layer Map

| Layer | Role Kafka plays | What it's optimizing for |
|---|---|---|
| CPU | Nearly idle on the hot read/write path by design | Broker does no serialization/deserialization of message payloads — it treats records as opaque byte blobs, so CPU is spent on network I/O bookkeeping and replication, not message processing |
| Memory | OS page cache is the *de facto* read buffer, not JVM heap | Kafka deliberately keeps broker JVM heap small and lets the OS page cache hold hot log segments — a caught-up consumer reads entirely from page cache, never touching disk |
| Disk | Sequential-only append + sequential read | All writes are appends to the tail of a log segment file; all reads (for a caught-up or slightly-behind consumer) are sequential scans forward — this is why Kafka on spinning disks was viable even in 2011: sequential I/O on HDDs rivals random I/O on SSDs |
| Network | The actual bottleneck in a well-run cluster | With CPU and disk both nearly free on the common path, network bandwidth (producer→broker, broker→replica, broker→consumer) is what a Kafka cluster is usually sized against |
| GPU | Not applicable | Kafka has no compute/transform role — it moves bytes, it doesn't process them |

This is the differentiator vs. a request/response message broker (e.g. traditional RabbitMQ/AMQP usage patterns): those brokers do per-message routing/ack bookkeeping that costs real CPU per message; Kafka pushes that cost down to "append to a file" and "serve a byte range," both of which the OS already does efficiently.

---

## The Signature Mechanism: Zero-Copy Sendfile

The traditional way an application serves a file over a socket costs **4 data copies and 4 context switches**: disk → kernel read buffer (DMA), kernel buffer → user-space application buffer (context switch, CPU copy), application buffer → kernel socket buffer (context switch, CPU copy), socket buffer → NIC (DMA). Two of those four copies exist purely to hand bytes to your application code and back, even though your application never actually inspects or modifies them — it just forwards them.

Kafka's broker never needs to look inside a message to serve a fetch request, so it uses Java NIO's `FileChannel.transferTo()`, which on Linux maps to the `sendfile()` syscall. `sendfile()` tells the kernel "move bytes from this file descriptor to this socket descriptor directly" — the data goes disk → kernel page cache → NIC via DMA, **without ever crossing into user-space memory**. This cuts the operation to 2 data copies and 2 context switches, and on NICs that support scatter-gather DMA, the kernel-buffer-to-NIC copy can be eliminated too, leaving only the initial disk-to-page-cache read (which itself is skipped entirely if the segment is already in page cache — the common case for a caught-up consumer).

```
Traditional read+write:                    Kafka's sendfile path:

Disk ──DMA──▶ Kernel buffer                Disk ──DMA──▶ Kernel page cache
                  │ (context switch,                          │
                  │  CPU copy)                                 │ (kernel-internal,
                  ▼                                             │  no context switch,
            User-space buffer                                   │  no CPU copy)
                  │ (context switch,                            ▼
                  │  CPU copy)                             Socket buffer
                  ▼                                              │
            Socket buffer                                        │ DMA
                  │ DMA                                           ▼
                  ▼                                              NIC
                 NIC

4 copies, 4 context switches                2 copies, 2 context switches
(app never touched the bytes                (bytes never leave kernel space
 either time — pure overhead)                 at all)
```

This is why the resource-layer table above says CPU is "nearly idle on the hot path" and memory is "page cache is the de facto read buffer" — `sendfile()` is the specific kernel mechanism that makes both statements true simultaneously. One hard limitation worth flagging: `sendfile()` cannot be used when the broker needs to touch the bytes in user space, which is exactly what TLS/SSL encryption requires (the kernel can't encrypt data it's moving internally) — so a Kafka cluster with inter-broker or client-broker TLS enabled loses the zero-copy fast path for that traffic and falls back to the traditional copy path.

You can observe the syscall directly. Find the broker's PID and trace its file/socket syscalls while a consumer is actively fetching:

```bash
# Find the Kafka broker process
jps -l | grep kafka.Kafka
# 41213 kafka.Kafka

# Trace sendfile()/read()/write() syscalls on the broker while a consumer polls
strace -f -e trace=sendfile,read,write -p 41213 -T 2>&1 | grep sendfile
```

```
sendfile(112, 89, [4096], 65536) = 65536 <0.000041>
sendfile(112, 89, [69632], 65536) = 65536 <0.000038>
```

`sendfile(out_fd, in_fd, offset, count)` — fd 89 is the log segment file, fd 112 is the client socket; the kernel moved 65536 bytes directly, no `read()`/`write()` pair ever shows up for this data. On a TLS-enabled listener the same trace instead shows `read()` into a user-space buffer followed by `write()` back out — the two-copy fallback described above.

The same story is visible without root/ptrace access via JMX, which is the practical way to check this in production:

```bash
# Compare bytes served (network-out) against actual disk reads on a caught-up broker
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name 'kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec' \
  --jmx-url service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi --one-time true
```

```
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec:Count  482913840
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec:MeanRate  9421.3
```

A high, steady `BytesOutPerSec` with near-zero `LogFlushRateAndTimeMs` and low broker CPU is the JMX-level fingerprint of the zero-copy path doing its job — the broker is moving a lot of bytes without doing proportional CPU or disk-flush work.

---

## High-to-Low Walkthrough: `producer.send()` to Bytes on the Wire

```
producer.send(record)
      │
      ▼
Partitioner                     picks a partition: explicit key hash (default:
                                 murmur2 hash of key mod partition count) or
                                 round-robin/sticky if no key
      │
      ▼
Record Accumulator              record is appended to an in-memory batch for
(per-partition buffer)          that partition — NOT sent immediately; batches
                                 accumulate until linger.ms elapses or the batch
                                 fills to batch.size
      │
      ▼
Sender thread                   drains ready batches, groups them by destination
                                 broker, sends as a single network request per
                                 broker (not per partition) — this batching is
                                 what amortizes network round-trips
      │
      ▼
Broker socket / network layer   request deserialized just enough to route to the
                                 correct partition's log — broker does NOT parse
                                 individual record contents
      │
      ▼
Log segment append              raw bytes appended to the active segment file for
(sequential disk write)         that partition (a single active segment per
                                 partition at a time; segment rolls over at a size/
                                 time threshold to a new file)
      │
      ▼
Page cache                      the write lands in OS page cache; the kernel
                                 decides when to flush to physical disk (flush
                                 policy is tunable but defaults to OS-driven, not
                                 per-message fsync — durability instead comes from
                                 replication, not immediate fsync)
      │
      ▼
Replication to followers        leader broker's replica fetcher threads on
(network, follower side)        follower brokers pull the new bytes over the
                                 network using the SAME fetch protocol a consumer
                                 would use — replication is "consume from the
                                 leader," not a separate mechanism
      │
      ▼
acks satisfied → ack to producer   (acks=all waits for the write to reach every
                                     in-sync replica's log before acking; acks=1
                                     only waits for the leader's local append)
```

On the read side, the same log segment is later served to a consumer via the zero-copy `sendfile()` path described above — the exact same bytes written above are, for a caught-up consumer, served straight out of the page cache they already landed in.

### The same trace, as literal commands

Create the topic first (4 partitions, replication factor 3, so leadership/ISR below is meaningful):

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 4 --replication-factor 3
```

Produce a keyed record from the console producer — the key is what the partitioner in the walkthrough above hashes to pick a partition:

```bash
kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092 \
  --property "parse.key=true" --property "key.separator=:"
>customer-42:{"order_id": 9001, "amount": 129.99}
```

Now inspect where that record actually landed — `--describe` shows the partition/leader/replica/ISR state that "Log segment append" and "Replication to followers" refer to:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
```

```
Topic: orders   TopicId: 4f2c...   PartitionCount: 4   ReplicationFactor: 3
        Topic: orders   Partition: 0   Leader: 1   Replicas: 1,2,3   Isr: 1,2,3
        Topic: orders   Partition: 1   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
        Topic: orders   Partition: 2   Leader: 3   Replicas: 3,1,2   Isr: 3,1,2
        Topic: orders   Partition: 3   Leader: 1   Replicas: 1,3,2   Isr: 1,3,2
```

`Leader` is the broker whose local disk actually received the "sequential disk write" step; `Replicas` is the full assignment; `Isr` (in-sync replicas) is the set the walkthrough's "acks satisfied" step waits on when `acks=all` — a replica drops out of `Isr` if its fetcher falls behind `replica.lag.time.max.ms`, at which point `acks=all` no longer waits for it.

Finally, check the consumer side of the same trace — a running consumer group's per-partition offset and lag, which is what "consumer tracks its own offset" cashes out to operationally:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

```
GROUP             TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG   CONSUMER-ID
order-processors  orders  0          8452            8452            0    consumer-1-a1b2
order-processors  orders  1          6118            6120            2    consumer-2-c3d4
order-processors  orders  2          9901            9901            0    consumer-1-a1b2
order-processors  orders  3          7201            7233           32    consumer-2-c3d4
```

`CURRENT-OFFSET` is the consumer's committed bookmark into `__consumer_offsets`; `LOG-END-OFFSET` is the partition's latest written offset (the tail the producer above just extended); `LAG` is the gap — partition 3's `LAG=32` means 32 records already durably written (and, if hot, already zero-copy-servable from page cache) haven't been consumed yet, the exact scenario the "slow consumer" gotcha below warns about.

---

## Deep Internals

### Partitions Are the Unit of Everything

A topic is a logical name; a **partition** is the actual physical log — an ordered, append-only sequence of records, each with a monotonically increasing **offset**. Ordering guarantees only hold *within* a partition, never across partitions of the same topic. Parallelism (both write throughput and consumer parallelism) is bounded by partition count: you cannot have more active consumers in a single consumer group reading a topic than it has partitions — extra consumers sit idle.

```
Topic "orders", 4 partitions:

Partition 0: [offset 0][offset 1][offset 2]...[offset 8452]  ← leader on broker 1
Partition 1: [offset 0][offset 1][offset 2]...[offset 6120]  ← leader on broker 2
Partition 2: [offset 0][offset 1][offset 2]...[offset 9901]  ← leader on broker 3
Partition 3: [offset 0][offset 1][offset 2]...[offset 7233]  ← leader on broker 1
```

Creating this layout and reasoning about parallelism is literal, not abstract — partition count sets the hard ceiling on consumer-group parallelism referenced above:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 4 --replication-factor 3 \
  --config min.insync.replicas=2
```

```
Created topic orders.
```

`min.insync.replicas=2` on a `replication-factor=3` topic means `acks=all` only needs 2 of the 3 replicas (not all 3) to have the write before acking — the standard "tolerate 1 broker down without blocking writes" setting. Adding partitions later is one-way (see gotchas below):

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic orders --partitions 8
```

### Log Segments and Retention

Each partition's log is physically split into **segment files** (default roll: 1GB size or a configured time interval, whichever comes first). Only the active (newest) segment is written to; older segments are immutable. Retention (time-based or size-based) deletes whole segment files once every record in them is past the retention threshold — retention is a segment-granularity operation, not a per-record one, which is why you can't reliably delete "just this one record" via retention config.

The segment files are visible directly on a broker's data disk — a partition's directory holds one `.log`/`.index`/`.timeindex` triplet per segment:

```bash
ls -la /var/lib/kafka/data/orders-0/
```

```
-rw-r--r-- 1 kafka kafka 1073741824 Sep 12 14:02 00000000000000000000.log
-rw-r--r-- 1 kafka kafka   10485760 Sep 12 14:02 00000000000000000000.index
-rw-r--r-- 1 kafka kafka   10485760 Sep 12 14:02 00000000000000000000.timeindex
-rw-r--r-- 1 kafka kafka  536870912 Sep 13 09:41 00000000000008452123.log   ← active segment
-rw-r--r-- 1 kafka kafka   10485760 Sep 13 09:41 00000000000008452123.index
```

The filename is the segment's base offset. Retention and segment-roll thresholds are per-topic configs, checkable and settable directly:

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --describe
```

```
Dynamic configs for topic orders are:
  segment.bytes=1073741824 sensitive=false synonyms={}
  retention.ms=604800000 sensitive=false synonyms={}
```

```bash
# Roll segments at 512MB instead of 1GB, retain 3 days instead of the 7-day default
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name orders \
  --alter --add-config segment.bytes=536870912,retention.ms=259200000
```

### Consumer Groups and Offset Tracking

Consumers in the same **consumer group** split a topic's partitions among themselves (one partition assigned to at most one consumer in the group at a time); each consumer tracks its own offset per partition, committed to an internal Kafka topic (`__consumer_offsets`) rather than to an external store. This is what makes "replay from an earlier offset" possible — the log itself is unchanged; only the consumer's bookmark moves.

The `--describe` output shown in the walkthrough above is the everyday view; resetting the bookmark to replay history is equally literal:

```bash
# Rewind the whole group to the earliest retained offset on every partition of "orders"
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processors --topic orders --reset-offsets --to-earliest --execute
```

```
GROUP                  TOPIC   PARTITION  NEW-OFFSET
order-processors       orders  0          0
order-processors       orders  1          0
order-processors       orders  2          0
order-processors       orders  3          0
```

Note this only moves the pointer stored in `__consumer_offsets`; the segment files above are untouched.

### KRaft: Metadata Without ZooKeeper

Through the 3.x line, Kafka used Apache ZooKeeper as an external coordination service for cluster metadata (broker membership, partition leader elections, ACLs, topic configs). **Kafka 4.0 (2026) removed ZooKeeper entirely** — the terminal ZooKeeper-capable release is 3.9, and every 4.x release runs in **KRaft** mode only, where cluster metadata is itself stored as a Kafka-style replicated log managed by a small quorum of controller nodes using the Raft consensus protocol. This is architecturally significant beyond "one less thing to operate": it means Kafka's own metadata now benefits from the same log-structured replication model as the message data it stores.

The controller quorum is configured directly in `server.properties`:

```properties
# server.properties on a KRaft controller node
process.roles=controller
node.id=1
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
```

and its live Raft state is queryable at runtime:

```bash
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
```

```
ClusterId:              4L6g3nShT-eMCtK--X86sw
LeaderId:               1
LeaderEpoch:            12
HighWatermark:          48213
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   14
CurrentVoters:          [1,2,3]
```

`HighWatermark` here is the metadata log's own high-watermark — the same concept as a partition's high-watermark for ordinary message data, just applied to Kafka's self-describing cluster state.

### KIP-848: Incremental Consumer Rebalancing

The consumer group rebalance protocol historically used a "stop-the-world" model: any membership change (a consumer joining/leaving) triggered a full revoke-and-reassign of every partition to every consumer, pausing the whole group. **KIP-848**, now generally available, replaces this with incremental cooperative rebalancing — only the partitions that actually need to move are reassigned, and unaffected consumers keep processing without interruption. This matters most for large consumer groups where membership churn (autoscaling, rolling deploys) was previously a recurring latency spike.

Opting a client into the new protocol is a single consumer config:

```properties
# consumer.properties
group.protocol=consumer
group.remote.assignor=uniform
```

`--describe --verbose` on the group shows the assignor and per-member epoch under the new protocol:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors --verbose
```

```
GROUP             COORDINATOR   ASSIGNOR   STATE     GROUP-EPOCH  MEMBERS
order-processors  broker-2      uniform    Stable    17           2
```

Under the old `eager` protocol, any membership change bumps `GROUP-EPOCH` and forces every member through revoke-then-rejoin; under `uniform`/cooperative assignment, only the affected members' assignments change on a bump.

### Tiered Storage (KIP-405)

Tiered storage (production-ready as of the KIP-405 rollout) lets a broker offload older log segments to remote object storage (e.g. S3) while keeping only recent/hot segments on local broker disk. This decouples retention window from local disk capacity — a topic can retain data for months without requiring months of local SSD per broker — at the cost of higher latency for reads that miss local disk and page cache and must fetch from remote tier.

Enabling it is a broker-level flag plus a per-topic opt-in:

```properties
# server.properties
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RSMFactory
```

```bash
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name orders \
  --alter --add-config remote.storage.enable=true,local.retention.ms=21600000
```

`local.retention.ms=21600000` (6 hours) keeps only 6 hours of segments on local broker disk while `retention.ms` (the overall retention set earlier) governs how long data lives in the remote tier — local disk sizing and total retention window become independent numbers.

---

## Comparative: vs. Pulsar and Redpanda

**Kafka vs. Apache Pulsar** — the core architectural split is storage/serving separation. Pulsar brokers are stateless: message data lives in a separate storage layer (Apache BookKeeper), and brokers just serve requests against it. This means a Pulsar broker can fail or rebalance without any data movement — a new broker just starts serving the same BookKeeper ledgers. Kafka's brokers *are* the storage layer (the partition's data is local disk on whichever broker is the leader), so broker failure/rebalancing in Kafka means partition leadership moves to a broker that already has a replica, not an arbitrary broker. Pulsar's design costs an extra network hop per write (broker → BookKeeper, rather than a purely local disk append) that Kafka's co-located design avoids on the write's local leg — this is the direct resource-layer tradeoff: Pulsar trades network hops for storage elasticity and better multi-tenancy isolation.

**Kafka vs. Redpanda** — Redpanda is a from-scratch, C++, Kafka-API-compatible reimplementation, built thread-per-core (each CPU core owns its own set of partitions and its own event loop, avoiding cross-core locking and context switches that a JVM thread-pool broker incurs) and without a JVM (no GC pauses, no page-cache-vs-heap tuning tension). It targets the same resource-layer profile Kafka aims for (CPU/network-light, disk-sequential) but removes the JVM-specific overhead (garbage collection pauses, heap/page-cache competition) from the equation. As of Kafka 4.0's ZooKeeper removal, much of Redpanda's original "operational simplicity" pitch (no separate coordination service) is narrower than it was — both now ship as a single coordinated binary/process model — so the remaining differentiator is more specifically the JVM-vs-native performance/ops profile than the ZooKeeper-vs-not distinction.

---

## Key Gotchas

- **TLS disables zero-copy**: enabling inter-broker or client-broker TLS means `sendfile()` can no longer be used for that traffic (the kernel can't encrypt in-flight bytes without user-space involvement) — expect materially higher CPU usage on brokers once TLS is turned on, not just a fixed security tax.
- **Page cache, not heap, is your read-performance memory budget**: over-provisioning broker JVM heap steals RAM from the OS page cache, which is what actually serves caught-up consumers — the common tuning mistake is a large heap "for safety," which paradoxically hurts throughput.
- **Partition count is a ceiling you can't easily lower**: you can add partitions to a topic, but you cannot remove them (this would break the key→partition hash mapping consumers rely on for ordering). Under-provisioning partition count limits future consumer-group parallelism; over-provisioning adds per-partition overhead (open file handles, replication traffic, controller metadata) — reason about intended max consumer parallelism up front.
- **A slow consumer breaks the zero-copy path**: `sendfile()`'s efficiency assumes the requested data is in page cache. A consumer that's fallen far behind (reading segments long since evicted from page cache) forces an actual disk read, which is much slower and can pressure page cache for everyone else — "one lagging consumer" is a real noisy-neighbor risk on shared brokers.
- **acks=1 vs acks=all is a real durability decision, not a knob to leave at default**: `acks=1` acknowledges once the leader's local log append succeeds — a leader crash before replication completes can lose that record even though the producer got an ack. `acks=all` (with `min.insync.replicas` set appropriately) waits for the write to reach every in-sync replica first.
- **KRaft migration is one-way per cluster**: once a cluster is fully migrated to KRaft (or created fresh on 4.x), there's no supported path back to ZooKeeper — treat the migration as a real cutover with the documented migration steps, not an experiment to casually revert.
- **Tiered storage changes tail-latency assumptions, not just capacity**: reads that fall back to the remote tier incur cross-network object-storage latency, which is a very different latency profile than local-disk/page-cache reads — an application relying on Kafka for low-latency replay of old data needs to account for this explicitly, not assume all historical reads behave like recent ones.

---

*Grounded against kafka.apache.org docs, Confluent's Kafka 4.0 release coverage, and community 2026 writeups on KRaft/tiered-storage status as of September 2026. Current release line: Kafka 4.x (4.0 GA February 2026, subsequent 4.1-4.3 releases through May 2026), KRaft-only (ZooKeeper support fully removed; 3.9 was the last ZooKeeper-capable line). Re-verify exact patch version and any newly-GA KIPs before citing a specific version number in a deliverable.*
