# etcd

## 30-Second Intuition

etcd is a distributed, consistent key-value store built on the Raft consensus algorithm — it exists to hold a small amount of critical configuration/coordination state (cluster membership, service discovery, leader election, and — most visibly — the entire state of a Kubernetes cluster) with a guarantee that Redis/Memcached deliberately don't provide: every read reflects every acknowledged write, cluster-wide, even across node failures (see `caching/redis-vs-memcached.md`'s "cache vs. coordination store" distinction). The one fact that matters operationally: **the Raft log is the only real source of truth — the key-value store you query is a materialized view derived from replaying that log.** Every write must be committed to a quorum of the log before it's acknowledged, which is precisely why etcd is disk-fsync-bound and network-quorum-bound rather than memory-bound — it trades throughput and latency headroom for the linearizability guarantee its consumers (Kubernetes chief among them) actually depend on.

---

## Resource-Layer Map

| Layer | Role etcd plays | What it's optimizing for |
|---|---|---|
| CPU | Modest — Raft's own bookkeeping (log matching, term/index tracking) and MVCC B-tree operations are cheap relative to the other bottlenecks below | Not the constraint; etcd clusters rarely run into CPU limits before hitting disk or network limits |
| Memory | Holds the current MVCC key space and recent revision history in memory (via boltdb's mmap'd pages), but this is not the differentiator | Correctness (linearizable reads), not raw memory throughput — contrast directly with Redis/Memcached, where memory *is* the whole point |
| Disk | Every write requires an **fsync to the Write-Ahead Log** before it can be acknowledged — this is the dominant cost in etcd's entire performance profile | Durability of the consensus log above all else: an acknowledged write must survive a crash, which is exactly why disk fsync latency (not disk capacity) is etcd's most operationally sensitive metric — official guidance treats >10ms fsync latency as the point where performance visibly degrades, and >50ms as the point things start breaking |
| Network | Every write requires a **quorum round-trip** — the leader must get acknowledgment from a majority of members before committing | Correctness under partition/failure: a write is never considered durable until enough independent machines have it, which bounds throughput by the slowest quorum member's round-trip latency, not the fastest |
| GPU | Not applicable | Consensus and small-state storage have no GPU-parallelizable workload at all |

This is the exact inverse of Redis/Memcached's profile: those are memory-bound and treat disk/network as optional or best-effort; etcd is disk-fsync-bound and network-quorum-bound *by design*, because the thing it sells is "this write definitely survived, cluster-wide," not "this write was fast."

---

## The Signature Mechanism: The Raft Log Is the Only Source of Truth

etcd's key-value store is not an independently-maintained data structure that happens to be replicated — it is a **materialized view built by sequentially applying every entry in the Raft-replicated log**, in log order, on every member. This is the actual mechanism behind "the KV store is a view, not the truth": if you deleted every member's boltdb file and replayed the Raft log from the beginning, you'd get back the exact same key-value state, deterministically, because the KV store's current state is defined as nothing more than "the log, applied in order."

```
Client: Put("service/db/leader", "node-3")
      │
      ▼
Leader appends to its own Raft log       entry: {term: 7, index: 4021,
(not yet committed — just proposed)       data: Put(...)}
      │
      ▼
Leader replicates entry to followers      leader sends AppendEntries RPC to
(network — this is the quorum step)        every follower in parallel
      │
      ▼
Followers fsync entry to their own WAL    EACH follower that accepts the
                                            entry must durably persist it
                                            (fsync) before acking — this is
                                            why disk fsync latency directly
                                            gates write latency: the leader
                                            is waiting on these acks
      │
      ▼
Leader waits for ACKs from a MAJORITY      once (N/2)+1 members (including
of members (quorum)                         itself) have durably persisted
                                             the entry, it's COMMITTED —
                                             this specific point, not the
                                             initial proposal, is what
                                             "durable" means in etcd
      │
      ▼
Leader applies the committed entry to      only NOW does the entry get
its own MVCC state machine (boltdb)         applied to the actual key-value
                                             data structure — a new MVCC
                                             revision is created
      │
      ▼
Leader acknowledges the client's Put()      the client's write is only
                                             acknowledged after the log
                                             entry was durably committed
                                             by quorum, not merely proposed
      │
      ▼
Followers independently apply the           each follower applies the same
same committed entry to their own            committed entry to its own
local state machine, asynchronously          local boltdb, in the same log
(no further client-visible step)             order — this is what guarantees
                                              every member's KV state
                                              converges to the identical
                                              result, deterministically
```

Because the KV store is defined purely as "the log, replayed," etcd's **linearizable reads** (the default read consistency mode) work by having the leader confirm it is still the leader (via a fresh round of heartbea­t/quorum contact) before answering, precisely because the leader's in-memory view is only guaranteed current if it can prove it hasn't been superseded — this is the mechanism that makes a read reflect every acknowledged write, not merely "whatever this one node currently has cached."

**The same client `Put()` from above, run for real:**

```bash
$ etcdctl put service/db/leader node-3
OK
```

That `OK` is the client-visible acknowledgment shown at the bottom of the diagram — it's only printed after the entry was replicated, fsynced by a quorum of members, committed, *and* applied to the leader's own boltdb. Reading it back with `-w json` exposes the two revision counters the MVCC section below builds on:

```bash
$ etcdctl get service/db/leader -w json
{
  "header": {"cluster_id": 14841639068965178418, "member_id": 10276657743932975437, "revision": 1042, "raft_term": 7},
  "kvs": [
    {
      "key": "c2VydmljZS9kYi9sZWFkZXI=",
      "create_revision": 1038,
      "mod_revision": 1042,
      "version": 3,
      "value": "bm9kZS0z"
    }
  ],
  "count": 1
}
```

`mod_revision: 1042` is the revision this specific write landed on (matches `entry: {index: 4021}` in the diagram above, just renumbered from the KV store's point of view); `create_revision: 1038` is the revision the key was *first* created at — the gap between them (`version: 3`) means this key has been overwritten twice since creation, each overwrite bumping `mod_revision` without touching `create_revision`. Confirming which member actually acted as the leader for this write:

```bash
$ etcdctl endpoint status --cluster -w table
+------------------+------------------+---------+---------+-----------+-----------+------------+
|     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER| RAFT TERM  |
+------------------+------------------+---------+---------+-----------+-----------+------------+
| 10.0.1.10:2379   | 8211f1d0f64f3269 | 3.6.13  | 20 MB   |      true |     false |          7 |
| 10.0.1.11:2379   | 91bc3c398fb3c146 | 3.6.13  | 20 MB   |     false |     false |          7 |
| 10.0.1.12:2379   | fd422379fda50e48 | 3.6.13  | 20 MB   |     false |     false |          7 |
+------------------+------------------+---------+---------+-----------+-----------+------------+
```

`IS LEADER: true` on `10.0.1.10` and matching `RAFT TERM: 7` across all three rows is the operational confirmation of exactly the term/index pair shown in the diagram's Raft log entry — every member agrees on who the leader is and which term is current, which is the whole point of the quorum step.

---

## Deep Internals

### MVCC: Every Write Is a New Revision, Nothing Is Overwritten In Place

etcd never mutates a key's prior value in place — every `Put` creates a new, monotonically increasing **revision** of the entire keyspace, and old revisions remain queryable (`etcdctl get key --rev=<N>`) until explicitly compacted. This is the exact mechanism behind Kubernetes's `--watch` semantics: a watcher subscribes starting from a specific revision, and etcd streams every subsequent change as a sequence of revisioned events — the watcher never has to poll, and never misses an event, because "give me everything after revision N" is a well-defined, replayable request against the revision history, not a best-effort tail of recent changes.

```
Put("pod/nginx-1", "Running")   → creates revision 1042
Put("pod/nginx-2", "Pending")   → creates revision 1043
Put("pod/nginx-1", "Terminated")→ creates revision 1044 (old value at rev 1042
                                    still queryable until compaction)

Watcher subscribed from revision 1043 receives, in order:
  {rev: 1043, key: "pod/nginx-2", value: "Pending"}
  {rev: 1044, key: "pod/nginx-1", value: "Terminated"}
```

Because old revisions are never freed automatically, etcd requires periodic **compaction** (discard revision history older than a chosen point) — and separately, **defragmentation** (reclaim the disk space that compaction freed *within* boltdb's file, since compacting revisions leaves internal gaps in the backend file that the OS filesystem can't see as free space until defragmented). These are two distinct operations for two distinct problems, and conflating them is a common operational mistake: compaction controls how much history exists; defragmentation controls whether freed history actually returns disk space to the file.

**Querying a historical revision directly** (this is what "old values remain queryable" means in practice — revision 1042 is the `pod/nginx-1: Running` write from the diagram above, still readable at 1044 even though the key has since moved on):

```bash
$ etcdctl get service/db/leader --rev=1042
service/db/leader
node-1
```

**Watching from a specific revision** is the literal mechanism behind Kubernetes's `--watch`: give etcd a starting revision and it streams every subsequent event, in order, with no polling and no gaps:

```bash
$ etcdctl watch service/ --prefix --rev=1043
PUT
service/db/leader
node-2
PUT
service/db/replica-1
node-4
DELETE
service/db/replica-2
```

Each block is one streamed event (event type, key, value) — the client that issued this watch resumes exactly where the Raft-log-derived revision history left off, which is why a watcher can crash, restart with `--rev=<last-seen+1>`, and never miss or replay an event.

**Compacting away history up to a revision**, then reclaiming the file-level space that compaction leaves behind:

```bash
$ etcdctl compact 1044
compacted revision 1044

$ etcdctl defrag --cluster
Finished defragmenting etcd member[10.0.1.10:2379]
Finished defragmenting etcd member[10.0.1.11:2379]
Finished defragmenting etcd member[10.0.1.12:2379]
```

Note `compact` returns instantly (it just marks revisions ≤1044 as no longer queryable — `get --rev=1042` above would now fail with `etcdserver: mvcc: required revision has been compacted`), while `defrag` is the comparatively expensive, per-member, latency-spiking operation that actually returns disk pages to the filesystem — this asymmetry is exactly why they're scheduled differently in production (compact frequently and cheaply, defrag rarely and off-peak).

**Verifying every member's data is actually consistent** (a linearizability sanity check across replicas, not just "the cluster says it's healthy"):

```bash
$ etcdctl endpoint hashkv --cluster -w table
+------------------+------------+------------+
|     ENDPOINT     |    HASH    | HASH REVISION |
+------------------+------------+------------+
| 10.0.1.10:2379   | 1401410120 |       1044 |
| 10.0.1.11:2379   | 1401410120 |       1044 |
| 10.0.1.12:2379   | 1401410120 |       1044 |
+------------------+------------+------------+
```

Identical `HASH` values across all three members at the same revision is the concrete, checkable proof that "every member's KV state converges to the identical result" (from the Signature Mechanism diagram) actually held — a mismatched hash at the same revision means divergence, which should never happen absent a serious bug or data corruption.

### Storage Engine: boltdb (bbolt)

The KV data itself lives in **bbolt**, an embedded, single-writer, copy-on-write B+tree-backed key-value store (a maintained fork of the original BoltDB), memory-mapped for reads. Every etcd revision is effectively a row in this underlying B+tree, keyed by revision number — this is the concrete storage substrate the "materialized view of the Raft log" abstraction above actually writes into.

Seeing which members actually make up the cluster whose boltdb files must stay in sync — this is the membership list the hashkv comparison above is checking *against*:

```bash
$ etcdctl member list -w table
+------------------+---------+--------+------------------------+------------------------+------------+
|        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      | IS LEARNER |
+------------------+---------+--------+------------------------+------------------------+------------+
| 8211f1d0f64f3269 | started | node-1 | http://10.0.1.10:2380  | http://10.0.1.10:2379  |      false |
| 91bc3c398fb3c146 | started | node-2 | http://10.0.1.11:2380  | http://10.0.1.11:2379  |      false |
| fd422379fda50e48 | started | node-3 | http://10.0.1.12:2380  | http://10.0.1.12:2379  |      false |
+------------------+---------+--------+------------------------+------------------------+------------+
```

### Raft Leader Election, Briefly

If the current leader stops heartbeating (network partition, crash, GC pause long enough to miss heartbeats), followers' election timers expire and a new leader election begins — a follower becomes a candidate, requests votes, and wins if it receives votes from a majority. During this window, the cluster cannot commit new writes (no leader to propose them), which is the direct operational implication of "Raft requires a leader to make progress" — a longer or flappier election process (often caused by exactly the same disk/network latency issues that slow down normal writes) directly extends write unavailability.

Re-running `endpoint status` after an election is the concrete way to observe the handoff — compare against the earlier output where `node-1` (`10.0.1.10`) was the leader at term 7:

```bash
$ etcdctl endpoint status --cluster -w table
+------------------+------------------+---------+---------+-----------+-----------+------------+
|     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER| RAFT TERM  |
+------------------+------------------+---------+---------+-----------+-----------+------------+
| 10.0.1.10:2379   | 8211f1d0f64f3269 | 3.6.13  | 20 MB   |     false |     false |          8 |
| 10.0.1.11:2379   | 91bc3c398fb3c146 | 3.6.13  | 20 MB   |      true |     false |          8 |
| 10.0.1.12:2379   | fd422379fda50e48 | 3.6.13  | 20 MB   |     false |     false |          8 |
+------------------+------------------+---------+---------+-----------+-----------+------------+
```

`RAFT TERM` incremented from 7 to 8 and `IS LEADER` moved to `node-2` — every member agreeing on both the new term and the new leader is what "won a majority of votes" looks like from the outside; any write attempted between the old leader going silent and this new leader being confirmed would have blocked or errored with `etcdserver: leader changed` / `etcdserver: request timed out`.

### v2 API Removal

The legacy v2 API/store (a pre-Raft-KV-abstraction data model) has been fully removed as of etcd 3.6 — the `--enable-v2` flag no longer exists. Any tooling still assuming the v2 API/data model needs migration; the v3 API (gRPC-based, the one all of the mechanisms above describe) has been the only supported interface for some time and is now the only one present at all in 3.6+. Concretely, on 3.6.13:

```bash
$ etcdctl --version
etcdctl version: 3.6.13
API version: 3.6

$ ETCDCTL_API=2 etcdctl get service/db/leader
Error: unknown command "get" for "etcdctl"
```

There is no v2 fallback to reach for anymore — `ETCDCTL_API=2` doesn't select a different code path in 3.6+, it just points at a client that has nothing left to talk to.

---

## Comparative: vs. ZooKeeper, vs. Consul

**etcd vs. ZooKeeper**: both are consensus-replicated coordination stores (ZooKeeper predates etcd and uses its own consensus protocol, ZAB, rather than Raft) with a similar operational shape (small critical state, watch-based notification, quorum writes). The practical differences are largely ecosystem and API surface: etcd's gRPC-based v3 API and its role as Kubernetes's literal datastore made it the default choice for cloud-native infrastructure, while ZooKeeper remains heavily entrenched in the Kafka/Hadoop-era ecosystem (though Kafka itself dropped its ZooKeeper dependency in favor of KRaft — see `streaming/kafka.md`), illustrating a broader industry trend of projects absorbing their own Raft-based consensus rather than depending on an external coordination service.

**etcd vs. Consul**: Consul also uses Raft for its consistent KV store but bundles substantially more scope into one system — native service discovery with health checking, a service mesh control plane, and multi-datacenter federation via a gossip protocol (Serf) layered on top of the Raft-based core. etcd deliberately stays narrower: a consistent KV store and that's essentially it, with higher-level systems (Kubernetes itself, or purpose-built service discovery tools) built on top rather than bundled in. Choosing between them is less "which has better consensus" (both use Raft correctly) and more "do you want a minimal KV primitive to build on, or an all-in-one service-mesh/discovery platform."

---

## Key Gotchas

- **Disk fsync latency is THE metric to watch, not disk throughput or IOPS in the generic sense**: etcd's official guidance treats fsync latency above ~10ms as visible degradation and above ~50ms as the point things start breaking — this means noisy-neighbor virtualized/shared storage (common on undersized cloud volumes) is a direct etcd availability risk, not just a "slightly slower" inconvenience. Dedicated, low-latency disks (ideally local NVMe) are the standard production recommendation specifically because of this sensitivity. Checking whether a given deployment's storage is actually fast enough *before* it becomes an outage:

  ```bash
  $ etcdctl check perf
  60 / 60 Booooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00%
  PASS: Throughput is 150 writes/s
  PASS: Slowest request took 0.056824s
  PASS: Stddev is 0.005089s
  PASS
  ```

  and, more directly, benchmarking write latency against the WAL disk itself:

  ```bash
  $ benchmark put --target-leader --conns=1 --clients=1 \
      --key-size=8 --val-size=256 --total=10000
  Summary:
    Total:        4.2113 secs.
    Slowest:      0.0121 secs.
    Fastest:      0.0002 secs.
    Average:      0.0004 secs.
    Stddev:       0.0003 secs.

  Latency distribution:
    10% in 0.0002 secs.
    50% in 0.0004 secs.
    90% in 0.0007 secs.
    99% in 0.0015 secs.
  ```

  A p99 creeping toward 10ms here — even though the mean looks fine — is the leading indicator for the disk-fsync-bound degradation this whole doc keeps coming back to; it shows up here before it shows up as election churn or client timeouts.

- **"etcdserver: mvcc: database space exceeded" is a real production outage mode, not a rare edge case**: etcd defaults to a conservative maximum backend size (historically 2GiB, with guidance to raise it but generally stay well under ~8GiB), and hitting that limit means etcd stops accepting writes entirely — for a Kubernetes cluster, this means no new pods can be scheduled, no state can be updated, cluster-wide, until space is reclaimed. This is caused by unbounded revision history growth (no compaction) far more often than by genuinely large data volume. Diagnosing and recovering from it, in order:

  ```bash
  $ etcdctl put service/db/leader node-5
  {"level":"warn","msg":"etcdserver: mvcc: database space exceeded"}

  $ etcdctl alarm list
  memberID:8211f1d0f64f3269 alarm:NOSPACE
  ```

  The `NOSPACE` alarm is what actually blocks writes cluster-wide — not just the size limit being near, but etcd deliberately refusing further writes once it's tripped, as a safety valve. Recovery is compact → defrag → disarm, in that order (disarming without freeing space first just re-trips the alarm on the next write):

  ```bash
  $ etcdctl compact $(etcdctl endpoint status -w json | grep -o '"revision":[0-9]*' | head -1 | cut -d: -f2)
  compacted revision 1044

  $ etcdctl defrag --cluster
  Finished defragmenting etcd member[10.0.1.10:2379]
  Finished defragmenting etcd member[10.0.1.11:2379]
  Finished defragmenting etcd member[10.0.1.12:2379]

  $ etcdctl alarm disarm
  memberID:8211f1d0f64f3269 alarm:NOSPACE

  $ etcdctl alarm list
  $ # (empty output = no active alarms; writes now accepted again)
  ```

- **Compaction and defragmentation are two separate steps, and doing one without the other doesn't fully solve the space problem**: compaction discards old revision history (controls how much logical history exists); defragmentation reclaims the *physical* file space that compaction's discarded revisions left as internal gaps in boltdb's file. Most production setups run both on a schedule (defragmentation commonly as an off-peak periodic job, since it's a member-at-a-time operation with a real latency-spike cost while running) — running `etcdctl compact` alone without ever running `etcdctl defrag` is precisely how a cluster can hit `NOSPACE` again and again despite "already compacting regularly."
- **A minority-side network partition makes etcd correctly unavailable, not incorrectly wrong**: if a partition splits the cluster such that no side has a quorum, the whole cluster stops accepting writes — this is Raft's consistency guarantee working as designed (better to refuse writes than risk a split-brain divergence), but it surprises operators expecting "at least the bigger side should still work" when the split is closer to even, or when enough members are simply unreachable rather than cleanly partitioned into two groups. From the client's side, this looks like a hang followed by a timeout, not a clean error:

  ```bash
  $ etcdctl put service/db/leader node-6 --dial-timeout=3s
  {"level":"warn","msg":"grpc: addrConn.createTransport failed to connect"}
  Error: context deadline exceeded
  ```

  `context deadline exceeded` here means exactly "no quorum was reachable to commit this write," not "the request failed" in the ordinary sense — retrying against the same minority side will fail identically until quorum is restored.

- **Leader elections pause writes for their duration**: anything that delays heartbeats (disk latency spikes, network blips, even Go GC pauses in older/less-tuned deployments) can trigger an election, during which no new writes commit — chronic election churn is usually a symptom of the same disk/network issues covered above, not a separate problem.
- **etcd is not a general-purpose database, even though it's easy to reach for one once it's already running your cluster**: it's tuned and guarantees-shaped for a small amount of frequently-read, infrequently-large-but-often-written critical state — using it to store arbitrarily large or high-churn application data (a common Kubernetes anti-pattern, e.g. large CRDs or high-frequency status updates from many controllers) is what most commonly drives clusters into the space-exceeded and latency-degradation gotchas above. `endpoint status` is the early-warning check for exactly this — `DB SIZE` creeping toward the configured quota (2–8 GiB) is the leading indicator, well before `NOSPACE` actually trips:

  ```bash
  $ etcdctl endpoint status --cluster -w table
  +------------------+------------------+---------+---------+-----------+
  |     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER |
  +------------------+------------------+---------+---------+-----------+
  | 10.0.1.10:2379   | 8211f1d0f64f3269 | 3.6.13  | 1.9 GB  |     false |
  | 10.0.1.11:2379   | 91bc3c398fb3c146 | 3.6.13  | 1.9 GB  |      true |
  | 10.0.1.12:2379   | fd422379fda50e48 | 3.6.13  | 1.9 GB  |     false |
  +------------------+------------------+---------+---------+-----------+
  ```

---

*Grounded against etcd.io release notes/blog posts and Kubernetes ecosystem operational writeups as of September 2026. Current stable lines: 3.6.x (3.6.13, July 2026 patch) and 3.5.x (3.5.32, same patch cycle) both actively maintained; the v2 API is fully removed as of 3.6 (the `--enable-v2` flag no longer exists). Re-verify exact patch version and current default database-size guidance before citing in a deliverable — size-limit defaults and recommendations are the kind of operational detail that shifts release to release.*
