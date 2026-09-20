# Distributed Clocks and Event Ordering

## 30-Second Intuition

Two machines' wall clocks can never be trusted to agree on "what happened first" — NTP drift alone gives you tens of milliseconds of slop, and that's enough to misorder events that are microseconds apart. Distributed systems solve this by replacing "what time is it" with "what must have happened before what": Lamport timestamps give you a single counter that respects causality but can't tell you when two events are truly unrelated (concurrent); vector clocks fix that by giving every node its own counter and tracking the whole vector, at the cost of O(n) space per timestamp; Hybrid Logical Clocks (HLC) glue a logical counter onto physical time so you get causality tracking that also looks like a real timestamp; and Google's TrueTime sidesteps the whole logical-clock problem by making physical clocks trustworthy via GPS/atomic clock hardware plus an explicit uncertainty bound. The one fact that matters operationally: **no logical clock (Lamport, vector, or HLC) ever tells you true wall-clock ordering — it only tells you causal ordering, and two events with "the same timestamp" or a "concurrent" relationship might both be correct in either wall-clock order.**

---

## Why Physical Clocks Fail

Every machine has a local oscillator-driven clock, kept roughly in sync with real time via NTP (or PTP for tighter sync). Two problems make raw wall-clock reads unsafe for ordering events across machines:

- **Clock skew**: at any instant, node A's clock and node B's clock disagree by some offset — typically single-digit to tens of milliseconds with NTP, since NTP corrects periodically rather than continuously.
- **Clock drift**: between synchronizations, each clock's oscillator runs slightly fast or slow, so the offset keeps changing rather than staying fixed.

If node A timestamps an event at `12:00:00.100` and node B timestamps a causally later event (it received a message from A and then acted) at `12:00:00.095`, a naive "compare timestamps" ordering gets it backwards — B's clock was simply behind. This is the reason systems that need correct ordering build a notion of causality instead of trusting `System.currentTimeMillis()` comparisons across hosts.

---

## Lamport Timestamps: Causality Without Wall Clocks

Leslie Lamport's 1978 logical clock gives each node a single integer counter and two rules:

1. **Increment rule**: before executing any event (including sending a message), a node increments its own counter: `C = C + 1`.
2. **Max rule**: when a node receives a message carrying timestamp `T_msg`, it sets its counter to `max(C, T_msg) + 1` before processing the receive event.

This produces a **happens-before** relation (written `a → b`): if `a` happens-before `b`, then `Lamport(a) < Lamport(b)`. Happens-before is defined as the transitive closure of "same-node program order" and "send happens-before receive."

**What it does NOT capture**: the converse is false. `Lamport(a) < Lamport(b)` does **not** imply `a → b`. Two events on different nodes that never causally influenced each other (concurrent events, written `a ∥ b`) will still get comparable integer timestamps, and you cannot tell from the numbers alone whether one really happened-before the other or whether they were just independent and got arbitrary relative numbers. This blind spot is what vector clocks exist to fix.

---

## Vector Clocks: Detecting Concurrency

A vector clock gives every node in an `n`-node system its own slot in a length-`n` vector, `V = [c_1, c_2, ..., c_n]`.

**Rules**:
1. On a local event (including send), node `i` increments only its own slot: `V[i] += 1`.
2. On receiving a message carrying vector `V_msg`, node `i` sets `V[j] = max(V[j], V_msg[j])` for every slot `j`, then increments its own slot `V[i] += 1`.

**Comparison rule** (this is the part Lamport clocks can't do): given two vectors `V_a` and `V_b`,
- `V_a → V_b` (a happened-before b) iff `V_a[k] <= V_b[k]` for all `k`, and `V_a[k] < V_b[k]` for at least one `k`.
- `V_a ∥ V_b` (concurrent) iff neither `V_a <= V_b` nor `V_b <= V_a` holds — i.e., some slot is bigger in `V_a` and some other slot is bigger in `V_b`.

That comparison is exactly how Dynamo-style systems detect write conflicts: if two versions' vector clocks are ordered, the newer one strictly dominates and can be kept alone; if they're concurrent, both versions must be kept and reconciled (either by the application, or via a last-writer-wins policy, or via CRDT merge).

**Cost**: a vector clock is `O(n)` integers per event, where `n` is the number of nodes that have ever written. In systems with high client/node churn this grows unboundedly unless pruned — see Gotchas.

---

## Worked Example: Distributed Shopping Cart Across 3 Nodes

Three nodes, `A`, `B`, `C`, service requests for the same shopping cart (Dynamo-style: any node can accept a write). Sequence of events, told twice — once through Lamport timestamps, once through vector clocks — to show what Lamport loses.

**Scenario**: A adds an item. Concurrently (before seeing A's write), B removes an item. Then C, having heard from both A and B, reads the cart.

### Pass 1 — Lamport Timestamps

| Step | Event | Rule applied | Lamport value |
|---|---|---|---|
| 1 | A: `add(item)` | local event, A's counter 0→1 | `A:1` |
| 2 | B: `remove(item)` (no knowledge of step 1 yet) | local event, B's counter 0→1 | `B:1` |
| 3 | A sends its write to C | increment before send | `A:2` (sent with msg) |
| 4 | C receives A's message | `max(0, 2)+1 = 3` | `C:3` |
| 5 | B sends its write to C | increment before send | `B:2` (sent with msg) |
| 6 | C receives B's message | `max(3, 2)+1 = 4` | `C:4` |

Looking only at the numbers, C sees `A's write = 2` and `B's write = 2`, then computes its own local values 3 and 4. If you tried to order the *original* operations purely by comparing `2` (A's send-time counter) vs `2` (B's send-time counter), you'd find them tied — and even if they weren't tied, **a Lamport timestamp comparison cannot tell you whether A's add and B's remove were causally related or just coincidentally numbered**. From C's point of view, was the remove a reaction to the add (so remove should win) or fully independent (so both operations are a genuine conflict that needs reconciling, e.g., "add wins" per cart semantics)? Lamport timestamps alone cannot answer this — that ambiguity is exactly the information vector clocks are designed to preserve.

### Pass 2 — Vector Clocks `[A, B, C]`

| Step | Event | Vector before | Rule applied | Vector after |
|---|---|---|---|---|
| 1 | A: `add(item)` | `[0,0,0]` | A increments own slot | `[1,0,0]` |
| 2 | B: `remove(item)` | `[0,0,0]` | B increments own slot | `[0,1,0]` |
| 3 | A sends `[1,0,0]` to C | — | — | (in flight) |
| 4 | C receives A's msg | `[0,0,0]` | `max` elementwise then C increments own slot: `max([0,0,0],[1,0,0])=[1,0,0]`, then `[1,0,1]` | `[1,0,1]` |
| 5 | B sends `[0,1,0]` to C | — | — | (in flight) |
| 6 | C receives B's msg | `[1,0,1]` | `max([1,0,1],[0,1,0])=[1,1,1]`, then increment C: `[1,1,2]` | `[1,1,2]` |

Now compare A's write vector `[1,0,0]` against B's write vector `[0,1,0]` directly:
- Is `[1,0,0] <= [0,1,0]`? No — A's slot (1) is bigger than B's slot (0) in position 1.
- Is `[0,1,0] <= [1,0,0]`? No — B's slot (1) is bigger than A's slot (0) in position 2.

Neither dominates → **`[1,0,0] ∥ [0,1,0]`: the add and the remove are provably concurrent.** The vector clock proves what the Lamport timestamp could only hide: B's remove did not causally depend on A's add. C, holding both versions with concurrent vectors, must keep both and apply the cart's conflict-resolution policy (e.g. "removes are commutative with adds of a different item, so merge as a set union/diff" or surface both to the client) rather than silently picking whichever write arrived last.

---

## Hybrid Logical Clocks (HLC)

An HLC timestamp is a pair `(pt, l)`:
- `pt` — a physical time component, kept close to (never exceeding by much) the node's actual wall clock.
- `l` — a logical counter that only increments to break ties when multiple events would otherwise land on the same `pt`.

**Update rule** on a local event or send, given previous HLC `(pt_old, l_old)` and current wall clock `now`:
```
pt_new = max(pt_old, now)
l_new  = l_old + 1   if pt_new == pt_old   (wall clock didn't advance — use logical tie-break)
       = 0            if pt_new > pt_old   (wall clock advanced — reset logical counter)
```
On receiving a message with `(pt_msg, l_msg)`:
```
pt_new = max(pt_old, pt_msg, now)
l_new  = l_old + 1              if pt_new == pt_old == pt_msg
       = l_msg + 1              if pt_new == pt_msg > pt_old
       = l_old + 1              if pt_new == pt_old > pt_msg
       = 0                       if pt_new > both pt_old and pt_msg
```
The result: `HLC` is always `>=` physical wall time (so it can be used for TTLs, human-readable ordering, and "roughly when did this happen") and it still totally orders causally-related events using the logical counter as a tiebreaker.

**Worked example — identical physical timestamps disambiguated**: Two nodes' clocks both read `pt = 100` (in whatever unit) at the moment of two unrelated local events:
- Node X performs event `e1`: previous HLC was `(98, 3)`. Now `pt_new = max(98, 100) = 100 > 98`, so logical resets: `e1 = (100, 0)`.
- Node Y performs event `e2` at the same wall-clock instant `pt = 100`, previous HLC was `(100, 2)` (Y's clock reached 100 slightly earlier and already ticked twice at that value). `pt_new = max(100, 100) = 100 == pt_old`, so logical increments: `e2 = (100, 3)`.

Both events show physical time `100`, but `(100,0)` and `(100,3)` are distinguishable and totally ordered — `e1 < e2` — purely via the logical counter, with no coordination between X and Y needed. This is the mechanism CockroachDB uses for MVCC timestamps and transaction ordering, and the one MongoDB uses for its cluster-wide logical clock: each oplog entry's HLC timestamp only ticks the logical counter on real state-changing operations (not on every message send/receive), producing a monotonic `cluster time` used to implement causally consistent sessions (`afterClusterTime` reads block until the replica's clock catches up).

**HLC is not free of physical-clock trust**: CockroachDB still requires bounded clock skew across the cluster (default max offset 500ms) and maintains an uncertainty window around HLC timestamps to decide when a read must be retried at a higher timestamp to preserve serializability — HLC narrows the problem, it doesn't eliminate the need for *some* clock synchronization discipline.

---

## TrueTime: Making Physical Clocks Trustworthy Instead

Google Spanner takes a different approach entirely: rather than layering logical counters on top of untrustworthy physical clocks, it makes the physical clocks trustworthy enough to use directly, by attaching an explicit **uncertainty bound**.

`TT.now()` doesn't return a single timestamp — it returns an interval `[earliest, latest]`, guaranteed to contain the true absolute time at the moment of the call. Spanner's datacenters run GPS receivers and atomic clocks as time references (not just NTP against the public internet), so the uncertainty `ε` (half the interval width) stays small — Google reports `ε` typically under ~4ms at p99 and ~10ms at p999 in production.

**Commit-wait**: when a Spanner transaction commits, it picks a commit timestamp `s = TT.now().latest`, then — before releasing locks / making the result visible — waits until `TT.now().earliest > s` is guaranteed, i.e. it waits out the remaining uncertainty window. This guarantees that by the time any other transaction can observe the commit, real wall-clock time has actually passed `s` on every node's clock, which is what gives Spanner external consistency (linearizability) across the whole globally-distributed database using physical timestamps, not just causal ordering.

**Worked example — two transactions and commit-wait**:
- Transaction `T1` finishes its writes; `TT.now()` at that instant returns `[t=100ms, t=106ms]` (ε=3ms around a 103ms center — using arbitrary local units for illustration). Spanner picks commit timestamp `s1 = 106ms` (the interval's upper bound) and then blocks until `TT.now().earliest > 106ms`, i.e. until real elapsed time makes the *next* `TT.now()` call return an interval whose lower bound clears 106ms — roughly a further ~3-6ms wait.
- Only after that wait completes does `T1` release its locks and become visible. Now transaction `T2`, which starts afterward and causally depends on `T1` (e.g., reads a row `T1` wrote), is guaranteed to get `TT.now().earliest` already `> s1`, so `T2` is assigned a commit timestamp `s2 > s1` — the wall-clock order matches the causal order, verifiably, without ever comparing logical counters.

The cost is exactly that wait — a few milliseconds of added commit latency in exchange for provable external consistency. This is a fundamentally different trade from HLC/vector clocks: TrueTime spends hardware and a small fixed latency tax to make *physical* time trustworthy, instead of spending space/coordination to track *logical* causality.

---

## Comparative Summary

| Mechanism | Problem solved | Detects concurrency? | Gives real wall-clock time? | Overhead |
|---|---|---|---|---|
| **Lamport timestamp** | Total order consistent with happens-before | No — collapses concurrent events into an arbitrary total order | No | O(1) — single integer |
| **Vector clock** | Same as Lamport, plus can prove two events are causally unrelated | Yes — exact | No | O(n) space/comparison, n = number of nodes/actors |
| **HLC** | Causal ordering that also looks like (and bounds) physical time | No (single scalar pair, like Lamport — can't detect concurrency between arbitrary node pairs without full vectors) | Approximately — tracks physical time, offset by bounded logical skew | O(1) — one (physical, logical) pair; needs loosely synchronized clocks |
| **TrueTime** | External (wall-clock-linearizable) consistency across a global DB | N/A — sidesteps the question by making physical order provably correct | Yes, within ε — the entire point | Specialized hardware (GPS/atomic clock time masters) + commit-wait latency (single-digit ms) |

Production reality as of 2026:
- **CockroachDB** uses HLC as its core timestamp mechanism for MVCC and transaction ordering, still requiring a configured max clock offset (default 500ms) and using an uncertainty interval to force read retries when clock skew could reorder a read incorrectly.
- **MongoDB** uses a cluster-wide HLC (`ClusterTime`) since the 4.0-era research work (published SIGMOD 2019) to implement causally consistent sessions — logical component ticks only on oplog-visible state changes, not on every message.
- **Google Spanner** (and Cloud Spanner) still runs TrueTime as its foundation — GPS + atomic clock time masters, explicit uncertainty intervals, and commit-wait for external consistency; this remains a hardware-dependent approach that isn't practical outside Google's own datacenter footprint (which is why HLC-based systems exist for everyone else).
- **Vector clocks** are the textbook Dynamo-paper mechanism and Riak KV's original conflict-tracking approach, but Riak moved to **Dotted Version Vectors (DVVs)** as the default since Riak 2.0 specifically because plain vector clocks either grow unboundedly with client churn or lose accuracy when pruned; DVVs bound size to the replication factor while preserving accurate causality tracking. Note the AWS-managed **DynamoDB** service (distinct from the original Dynamo paper) dropped vector clocks entirely in favor of leader-based replication (Multi-Paxos per partition) — "Dynamo the paper" and "DynamoDB the product" diverge here and this is a common point of confusion.

---

## Key Gotchas

- **Vector clock size grows with cluster/actor count**: a vector clock is O(n) where n is every node or client that has ever contributed a write, not just the current cluster size. Systems that don't prune (or that track per-client rather than per-node vectors) can see clocks balloon to thousands of entries under high churn — this is precisely why Riak replaced plain vector clocks with Dotted Version Vectors, which bound the tracked size to the replication degree instead of the historical actor count.
- **Logical clocks (Lamport/vector/HLC) never give you real wall-clock time**: a common misconception is that a Lamport or vector timestamp can be used to answer "how long between these two events in real seconds." It cannot — it only encodes ordering/causality, not duration. Even HLC's physical component is only an approximation bounded by clock sync quality, not a certified wall-clock reading; only TrueTime-style explicit uncertainty intervals let you make real-time claims with a stated confidence bound.
- **HLC and Lamport clocks assume "loosely synchronized enough" physical/NTP clocks**: HLC's physical component tracks `max(local wall clock, previous HLC)` — if a node's NTP sync is badly broken (clock far in the future or stuck), it can poison the whole cluster's HLC values (every node that talks to it inherits the max), or force excessive uncertainty-interval retries in systems like CockroachDB. Don't treat HLC as a substitute for actually monitoring clock sync health.
- **"Concurrent" (vector clocks) means causally independent, not "happened at the same wall-clock instant"**: two vector-clock-concurrent events could have occurred seconds apart in real time — concurrency here is purely about the absence of a causal path between them, which is why concurrent writes need application-level (or CRDT) conflict resolution rather than "just pick whichever has the later real-world timestamp."
- **TrueTime's guarantees are hardware-dependent, not algorithm-dependent**: the external consistency Spanner provides comes from the *combination* of commit-wait and genuinely tight ε from GPS/atomic-clock time masters. Running the same commit-wait algorithm over ordinary NTP-synced clocks (where skew can be tens of ms with no certified bound) does not give you the same guarantee — you'd be waiting out a bound you can't actually trust.

---

## Sources

- [CockroachDB HLC glossary entry](https://www.cockroachlabs.com/glossary/distributed-db/hybrid-logical-clock-hlc-timestamps/)
- [CockroachDB Transaction Layer docs](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer)
- [MongoDB cluster-wide logical clock implementation (SIGMOD 2019, via muratbuffalo summary)](http://muratbuffalo.blogspot.com/2024/04/implementation-of-cluster-wide-logical.html)
- [MongoDB causal consistency guarantees (Lamport clock, cluster time)](https://vkontech.com/causal-consistency-guarantees-in-mongodb-lamport-clock-cluster-time-operation-time-and-causally-consistent-sessions/)
- [Google Cloud Spanner TrueTime and external consistency docs](https://docs.cloud.google.com/spanner-omni/true-time-external-consistency)
- [Spanner: Google's Globally-Distributed Database (OSDI 2012 paper)](https://research.google.com/pubs/archive/39966.pdf)
- [Riak: Vector Clocks Revisited Part 2 — Dotted Version Vectors](https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/index.html?p=9929.html)
- [Riak KV Dotted Version Vectors product docs (TI Tokyo)](https://riak.com/products/riak-kv/dotted-version-vectors/index.html?p=10941.html)
- [Redowan's Reflections: Dynamo paper vs DynamoDB product](https://rednafi.com/shards/2026/04/dynamo/)
