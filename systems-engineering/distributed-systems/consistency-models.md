# Consistency Models: CAP, PACELC, and the Linearizability-to-Eventual Spectrum

## 30-Second Intuition

CAP theorem says that during a network partition you must choose consistency or availability — but partitions are rare, and CAP says nothing about the trade-off that happens on *every single request*: consistency vs. latency, even when the network is perfectly healthy. That's what PACELC adds: **if Partitioned, choose Availability or Consistency; Else (normal operation), choose Latency or Consistency.** The four points on the "else" side aren't binary — they form a spectrum (linearizable → sequential → causal → eventual), each one relaxing a specific guarantee about what order operations appear to happen in, in exchange for lower latency and higher availability. The one fact that matters operationally: **most systems are not "a CP system" or "an AP system" — they are tunable per-request, and the real design question is which consistency level you pick for which read, not which theorem your database satisfies.**

---

## CAP: What It Actually Says (and What It Doesn't)

CAP (Brewer's theorem, formalized by Gilbert & Lynch) applies to exactly one scenario: **a network partition is happening right now.** During that partition, a node cut off from its peers has two choices for any request it receives:

- Answer it anyway, possibly with stale data → **Availability** (system stays up, consistency guarantee is broken)
- Refuse to answer until it can confirm it isn't stale → **Consistency** (guarantee preserved, system is unavailable for that node)

You cannot have both, for the simple reason that "confirm I'm not stale" requires talking to the rest of the cluster, which is exactly what a partition prevents.

The common "pick 2 of 3" framing is misleading: partition tolerance isn't optional in a real distributed system (the network *will* partition eventually), so in practice CAP only ever asks you to pick between C and A, and only during the partition window. **The rest of the time — which for most systems is the overwhelming majority of the time — there is no CAP trade-off being made at all.** This is exactly why CAP alone is an incomplete design tool: it's silent about the 99.9%+ of requests that aren't happening during a partition.

---

## PACELC: The Trade-off That Happens on Every Request

PACELC (Abadi, 2012) closes that gap:

```
if Partition:
    choose Availability or Consistency        (CAP's trade-off)
else:
    choose Latency or Consistency             (PACELC's addition)
```

The "else" branch is the practically important one, because it's not conditional on a rare event — it's a decision embedded in the system's architecture that applies to every read and write. Wanting a linearizable read means contacting enough replicas (or the leader) to prove the answer is current, which costs a round trip. Accepting a slightly stale answer means answering from whatever replica is closest, at lower latency. This trade-off is live continuously, not just during failures.

Classifying real systems on both axes:

| System | Partition behavior (CAP) | Normal-operation behavior (PACELC) | Classification |
|---|---|---|---|
| DynamoDB (default config) | Available | Latency favored (eventually consistent reads by default) | AP / EL |
| Cassandra (tunable) | Available (with low CL) | Tunable — CL=ONE favors latency, CL=QUORUM/ALL favors consistency | AP / EL (default), can shift toward EC per-query |
| Google Spanner | Consistent | Consistency favored (external consistency via TrueTime, at the cost of commit-wait latency) | CP / EC |
| etcd | Consistent (loses quorum → unavailable) | Consistency favored (linearizable reads confirm leadership before answering) | CP / EC — see `coordination/etcd.md` |
| DNS | Available (serves cached/stale records) | Latency favored (TTL-based caching, no coordination at read time) | AP / EL |

The useful reframing: **PA/EL and PC/EC are not two teams — they are two independent dials**, and a single product often sets them differently for different data (Cassandra literally lets you set the dial per-query via consistency level; DynamoDB lets you set it per-read via `ConsistentRead`).

---

## The Consistency Spectrum

CAP/PACELC tell you *when* you're trading consistency for something else. They don't tell you *what exactly* you gave up. That's what the consistency spectrum describes — each level is a specific, formally different guarantee about the order in which operations become visible, not just a vague "more or less consistent."

```
STRONGEST ─────────────────────────────────────────────────────────► WEAKEST
                                                                  (lower latency,
                                                                   higher availability)

Linearizability  →  Sequential Consistency  →  Causal Consistency  →  Eventual Consistency

"Every op appears    "All processes see      "Causally related     "No ordering guarantee
 to happen at one      the same total order    ops are seen in       at all — replicas
 instant between       of ops — but that       order; concurrent     converge to the same
 its start and end,    order need not match    unrelated ops can     value eventually if
 globally, in real     real-world wall-clock   be seen in different  writes stop, but any
 time. Equivalent      time. A process can     orders on different   read in the meantime
 to there being only   see writes 'early'      replicas."            can return anything
 one copy of the       relative to wall clock,                       from 'stale' to
 data."                as long as everyone                           'nonexistent'."
                       agrees on ONE order."
```

Each step down relaxes a specific promise:

- **Linearizability**: real-time order is preserved. If operation A finishes before operation B starts (in real wall-clock time), every observer must see A before B. This is the only level where "as if there's one copy of the data" is actually true.
- **Sequential consistency**: a single global order exists and every process agrees on it, but that order doesn't have to match real time — a write can appear to have happened "earlier" than it did in wall-clock terms, as long as the illusion is consistent for everyone.
- **Causal consistency**: only operations with a genuine cause-effect relationship (a read-then-write, or an explicit dependency) must be seen in order. Two unrelated concurrent writes can be observed in different orders by different replicas — that's allowed, because they weren't causally connected in the first place.
- **Eventual consistency**: the only promise is convergence — if writes stop arriving, all replicas eventually agree. There is no promise about what you see *before* convergence, and no promise about ordering at all.

---

## Worked Example: A "Like" Count on a Social Media Post, Across Two Regions

Setup: a post's like-count is stored on replicas in `us-east` and `eu-west`. Two users hit different regions milliseconds apart:

- User A (in `us-east`) likes the post at `t=0ms`. Request completes at `t=8ms`.
- User B (in `eu-west`) reads the like count at `t=10ms`.

Cross-region replication between `us-east` and `eu-west` takes ~70ms one-way in this example (realistic transatlantic RTT territory).

**Under linearizability** (e.g. a single global lock / Spanner-style external consistency):

```
t=0ms    User A's like → coordinator confirms with quorum/TrueTime commit-wait
t=8ms    Write acknowledged to User A — now globally "real" for anyone reading after t=8ms
t=10ms   User B's read in eu-west → must contact enough replicas to PROVE the current
         value, which forces it to wait for the write to actually be visible cluster-wide
t=~78ms  User B's read returns, now reflecting User A's like
```

User B's request pays real latency (waiting for the coordinator/quorum check) to guarantee it never sees a value older than what already happened in real time. The read is slow, but the count is always trustworthy the instant it returns.

**Under causal consistency** (e.g. a session-guarantee store that tracks "read-your-writes" and dependency chains):

```
t=0ms    User A's like recorded in us-east, tagged with a causal/version marker
t=8ms    Ack returned to User A immediately — no cross-region wait
t=10ms   User B's read in eu-west has NO causal relationship to User A's like
         (B never read anything A wrote, no shared session) → eu-west answers
         from its LOCAL replica, which hasn't received the replication yet
t=10ms   User B sees the OLD count (User A's like not yet visible)
t=~80ms  Replication arrives in eu-west; count updates locally
```

If User B had instead just liked the post themselves right after seeing a notification from A ("A liked this too!" — a causal link), causal consistency guarantees B would see A's like reflected, because that dependency is tracked. But an unrelated read from a third region has no such guarantee.

**Under eventual consistency** (e.g. DynamoDB default reads, Cassandra at CL=ONE):

```
t=0ms    User A's like written to us-east, ack'd immediately
t=8ms    Ack returned to A — us-east replica updated, replication to eu-west queued
t=10ms   User B's read in eu-west → answered from whatever eu-west's replica currently
         holds, no check against us-east, no ordering guarantee at all
t=10ms   User B sees the old count (same observable result as causal here, but for a
         different reason: there's no guarantee at all, it just happens to be stale)
t=~80ms  Replication eventually lands, eu-west converges to the same count
```

The key distinguishing case where eventual and causal actually diverge: imagine User B's client had *just* received a push notification confirming A's like (a causal dependency), then immediately queried the count. Under causal consistency, that query is guaranteed to reflect A's like. Under plain eventual consistency, it is not — the read might still return the stale count even though the client has direct proof the write already happened elsewhere. This is the concrete difference "eventual" and "causal" are describing: whether known dependencies are honored, not just whether things eventually converge.

**Under sequential consistency** (weaker use case, but illustrates the "one order, not necessarily real-time" point): if User A likes the post, and independently a moderator un-likes a different, unrelated old like on the same post from `us-east` a moment later, sequential consistency guarantees every reader sees those two operations in the *same* relative order — but does not guarantee that order matches true wall-clock arrival time, and does not require the moderator's op and A's op to be ordered relative to some third concurrent op happening in `eu-west` unless that order is also globally agreed. In practice, few production KV/counter stores expose pure sequential consistency as a distinct tier — it mostly appears as a stepping stone in the theory, and in some replicated log designs (e.g. certain multi-Paxos setups) that intentionally decouple commit order from real-time order.

---

## Key Gotchas

- **"Eventual consistency means the data is wrong"**: it doesn't — it means there is a window during which different replicas can disagree, and no ordering guarantee about what you'll see during that window. Once writes stop, replicas converge to the *same correct* value. The "wrongness" people mean is really "staleness during the convergence window," which is bounded in practice (typically milliseconds to low seconds for well-run systems) but not zero and not guaranteed by the model itself.
- **"CAP means you must always choose availability or consistency"**: CAP only applies *during a partition*. The rest of the time (which is most of the time for a healthy network), the real trade-off in play is PACELC's latency-vs-consistency, and many systems provide both consistency and availability simultaneously when nothing is partitioned. Treating CAP as an always-on binary ignores 99%+ of a system's actual operating time.
- **"My database is CP, so every read is safe"**: consistency levels in real systems are usually chosen per-operation, not fixed for the whole database. Cassandra at CL=ONE and Cassandra at CL=QUORUM are the *same* database giving *different* guarantees depending on what the caller asked for. Check the actual read/write path, not the marketing label ("Cassandra is AP" is a default-config generalization, not a hard constraint).
- **Linearizability is not free even absent a partition**: it costs a round trip (to a leader, or to enough replicas to prove currency) on every operation. This is why etcd's linearizable reads have the leader re-confirm leadership before answering (see `coordination/etcd.md`) — the cost is paid on every single read, not just during failures.
- **Causal consistency requires tracking dependencies, which has a real cost and a real failure mode**: if the causality-tracking metadata (vector clocks, session tokens, dependency vectors) is dropped or not propagated correctly (e.g. a client switches to a different session token, or metadata is stripped by an intermediate proxy), the system silently degrades to eventual consistency without telling you — the failure is invisible unless specifically tested for.
- **Sequential consistency is easy to confuse with linearizability**: the distinguishing test is real-time order. If a system lets a write "appear" to happen before it actually completed in wall-clock terms — as long as that fictional order is agreed upon by all readers — it's sequential, not linearizable. This distinction rarely matters for typical CRUD workloads but matters a great deal for distributed locks, leader election, and anything reasoning about "what really happened first."
- **Spanner's external consistency is strictly stronger than linearizability, not just another name for it**: linearizability is defined over single-object read/write operations; external consistency (what Spanner actually provides) extends the same real-time ordering guarantee to multi-object, multi-statement transactions — a strictly harder problem that TrueTime's bounded-uncertainty clock plus commit-wait latency is specifically built to solve.
- **Picking a consistency level is a per-field decision, not a per-database one**: a single application commonly needs linearizable reads for an inventory counter or account balance, and is perfectly happy with eventual consistency for a "likes" count or a recommendation feed. Applying one blanket consistency policy across an entire system usually means over-paying for latency somewhere or under-paying for correctness somewhere else.

---

*Grounded against Abadi's PACELC formulation, Gilbert & Lynch's CAP proof, Google Cloud's "TrueTime and external consistency" documentation, and current (2026) AWS DynamoDB and Apache Cassandra consistency-level documentation. DynamoDB and Cassandra classifications reflect their *default* configuration — both expose tunable consistency per-request (DynamoDB's `ConsistentRead` flag, Cassandra's per-query consistency level), so "DynamoDB is AP" and "Cassandra is AP" describe the common case, not an architectural ceiling. Re-verify current default consistency-level names and any provider-specific terminology before citing in a deliverable.*
