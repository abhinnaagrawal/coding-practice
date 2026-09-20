# Consistent Hashing

## 30-Second Intuition

Naive sharding (`hash(key) % N`) ties every key's owner to the total node count `N` — change `N` by one and the modulo result flips for almost every key, forcing a near-total cache/data remap. Consistent hashing fixes this by mapping both nodes and keys onto the same circular hash space (a "ring") and giving each key to the next node clockwise from it; adding or removing a node only touches the keys between that node and its neighbor, not the whole keyspace. The single operational fact to retain: **plain consistent hashing bounds remapping to roughly `1/N` of keys per membership change, but a small physical node count still needs virtual nodes (many ring points per physical node) to actually get uniform load — without them you can have correct remap bounds and badly skewed load at the same time.**

---

## Why `hash(key) % N` Fails

The naive scheme picks a node for a key by `node = hash(key) % N`. It's O(1) and perfectly uniform for a fixed `N`. It falls apart the moment `N` changes, because `% N` and `% (N±1)` agree on almost nothing:

```
5 nodes, N=5:
  hash("user:1")   = 1042  → 1042 % 5 = 2   (node 2)
  hash("user:2")   = 7781  → 7781 % 5 = 1   (node 1)
  hash("user:3")   = 3390  → 3390 % 5 = 0   (node 0)
  hash("user:4")   = 5566  → 5566 % 5 = 1   (node 1)
  hash("user:5")   = 9123  → 9123 % 5 = 3   (node 3)

Add a 6th node, N=6 — same hashes, new modulo:
  hash("user:1")   = 1042  → 1042 % 6 = 4   (node 4)  ← moved
  hash("user:2")   = 7781  → 7781 % 6 = 5   (node 5)  ← moved
  hash("user:3")   = 3390  → 3390 % 6 = 0   (node 0)  ← stayed (luck)
  hash("user:4")   = 5566  → 5566 % 6 = 4   (node 4)  ← moved
  hash("user:5")   = 9123  → 9123 % 6 = 3   (node 3)  ← stayed (luck)
```

Only 2 of 5 keys happened to stay here, and that's before accounting for the fact that nodes 0-4's *entire* previous ranges are now scrambled against a modulus that shares no structure with the old one. In expectation, changing `N` to `N+1` remaps `≈ (N)/(N+1)` of all keys — for `N=5→6` that's ~83%, and it gets worse (not better) as `N` grows. For a cache, that's a near-total cold start; for a data store, it's a near-total data migration. This is the problem consistent hashing exists to solve.

---

## The Hash Ring

Instead of hashing into `[0, N)`, hash into a large fixed circular space — classically `[0, 2^32)` or `[0, 2^128)` for a cryptographic hash, describable as degrees `[0, 360)` for diagrams. Both **nodes** and **keys** are hashed into this same space. A key is owned by the first node encountered walking clockwise from the key's position — its "successor" on the ring.

```
                    0°/360°
                       │
            N-E ●──────┼──────● N-A
                 \      │      /
                  \     │     /
        key3 ×     \    │    /
                     \  │  /
   270° ──────────────  ●  ──────────────  90°
                     /  │  \
                    /   │   \      × key1
                   /    │    \
                  /     │     \
            N-D ●───────┼───────● N-B
                        │
                       180°
                        │
                      ● N-C
                  × key2

Clockwise ownership (walk from each key to the next node CW):
  key1 (≈100°) → owned by N-B (≈130°)
  key2 (≈190°) → owned by N-C (≈200°)
  key3 (≈250°) → owned by N-D (≈280°)
```

When a node leaves, only the arc it owned falls to its clockwise successor — every other node's arc is untouched. When a node joins, it carves out a slice from exactly one existing node's arc (the one it lands inside) — again, every other node's arc is untouched. With `N` roughly-evenly-spaced nodes, a single join/leave event moves about `1/N` of the total keyspace, not `(N-1)/N` of it. That's the entire value proposition versus modulo sharding — the *bound* on movement, not zero movement.

---

## Worked Example: 5 Nodes, Ring of Size 360, Then a 6th Node Joins

Use a toy hash `h(s) = (sum of ASCII codes of s) * 7 mod 360` — deterministic, hand-traceable, not cryptographically real, but arithmetically honest for tracing ownership.

**Place 5 nodes:**

```
h("node-A") : sum=414*7=2898 mod 360 = 138
h("node-B") : sum=417*7=2919 mod 360 = 159
h("node-C") : sum=420*7=2940 mod 360 = 180
h("node-D") : sum=423*7=2961 mod 360 = 201
h("node-E") : sum=311*7=2177 mod 360 = 137   →  wait, recompute cleanly below
```

To keep this traceable, use fixed placed positions directly (as if computed by the hash — this is what actually matters, the ring math, not ASCII arithmetic):

```
Ring positions (0-360), 5 physical nodes:
  node-A =  20°
  node-B =  95°
  node-C = 160°
  node-D = 230°
  node-E = 310°

Keys:
  key-alpha = 35°
  key-beta  = 100°
  key-gamma = 150°
  key-delta = 245°
  key-epsil = 355°
```

**Ownership (next node clockwise):**

```
key-alpha ( 35°) → next CW node ≥ 35°  → node-B ( 95°)
key-beta  (100°) → next CW node ≥ 100° → node-C (160°)
key-gamma (150°) → next CW node ≥ 150° → node-C (160°)
key-delta (245°) → next CW node ≥ 245° → node-E (310°)
key-epsil (355°) → wraps past 360° → node-A ( 20°)
```

**Now add a 6th node, `node-F` at 240°** (lands between node-D at 230° and node-E at 310°):

```
New ring: node-A(20) node-B(95) node-C(160) node-D(230) node-F(240) node-E(310)

Re-check ownership:
  key-alpha ( 35°) → node-B ( 95°)   UNCHANGED
  key-beta  (100°) → node-C (160°)   UNCHANGED
  key-gamma (150°) → node-C (160°)   UNCHANGED
  key-delta (245°) → next CW node ≥ 245° → node-E (310°)   UNCHANGED (245 > 240, still past node-F)
  key-epsil (355°) → node-A ( 20°)   UNCHANGED
```

None of these 5 sample keys moved, because `node-F` only claims the arc `(230°, 240°]` — previously owned by `node-E` — and none of the sample keys fell in that narrow slice. Only keys with hash values in `(230°, 240°]` (formerly routed to node-E, the old successor of node-D's arc) now redirect to node-F. That arc is `10/360 ≈ 2.8%` of the ring here — in a real system with hundreds/thousands of hashed points this converges toward the `≈1/N` expectation (`≈1/6 ≈ 16.7%` for perfectly even spacing, less in this lumpy 5-point example). Compare directly to the modulo case above, where changing `N=5→6` remapped 3 of 5 keys (60%) with no locality to the change at all — every node's ownership scrambled, not just the neighborhood of the new node.

---

## Virtual Nodes: Fixing Uneven Ring Distribution

Hashing 3-10 physical nodes directly onto the ring produces uneven arcs purely from the luck of where each hash lands — with few points, gaps are large and irregular:

```
3 physical nodes, hashed directly onto a 360° ring (bad luck, but plausible):
  node-A =  10°
  node-B =  40°
  node-C = 320°

Arc widths (each node owns the arc from the previous node, exclusive, to itself):
  node-A owns (320°, 360°] + (0°, 10°]  = 50°   (13.9% of ring)
  node-B owns (10°, 40°]               = 30°   ( 8.3% of ring)
  node-C owns (40°, 320°]              = 280°  (77.8% of ring)  ← massively overloaded
```

`node-C` gets ~78% of all keys just because of where its single hash happened to land. This isn't a corner case — with only a handful of points on a large ring, this kind of clumping is the expected outcome, not the exception (it's the same "birthday problem" clustering behavior that shows up any time you scatter few points onto a large space).

**Fix**: hash each physical node to many points on the ring ("virtual nodes" / "vnodes"), typically by hashing `node_id + "#0"`, `node_id + "#1"`, ... `node_id + "#(V-1)"` for some `V` per node (100-256 is a common range; Cassandra defaulted to 256 per node pre-4.0, and DataStax's own guidance since has pushed toward far fewer — commonly 4-8 — once token-allocation-aware algorithms made low counts viable). Each physical node now owns many small, scattered arcs instead of one large contiguous one, and the law of large numbers evens out total ownership per node:

```
Same 3 physical nodes, 100 virtual points each (300 total ring points):
  node-A's 100 points scattered ~uniformly around the ring
  node-B's 100 points scattered ~uniformly around the ring
  node-C's 100 points scattered ~uniformly around the ring

Expected aggregate ownership per physical node: ~33.3% each
Observed variance with V=100 per node: single-digit percent, not the 78%/14%/8%
  skew of the single-point case
```

This is exactly Dynamo's stated rationale for introducing virtual nodes: single-point-per-node placement caused non-uniform data and load distribution, and giving each physical node multiple ring positions (Dynamo calls them "tokens") is the fix (Amazon's 2007 Dynamo paper, SOSP). Cassandra inherited the same mechanism directly from Dynamo's design and used it as the deployment default for over a decade before improved token-allocation algorithms (Cassandra 3.x+ / `allocate_tokens_for_keyspace`) made small vnode counts (as low as single digits) practical without reintroducing the imbalance problem the high vnode count was originally covering for.

---

## Bounded-Load and Rendezvous Hashing (Brief)

Plain consistent hashing (with or without vnodes) bounds *how many* keys move on membership change, but says nothing about per-node *load* if the key-access distribution is skewed — a node can still get hammered if it owns a few very hot keys. Two adjacent techniques worth knowing by name rather than reimplementing from scratch:

- **Consistent hashing with bounded loads**: caps how far any node's load can exceed the average (a tunable factor, e.g. 1.25x) by overflowing excess requests to the next node on the ring when a node is over its cap — trades a small amount of extra movement for a hard ceiling on load skew.
- **Rendezvous hashing (HRW — "highest random weight")**: no ring at all. For a key, compute a combined weight `weight(key, node)` for every candidate node and pick the max. Same guarantee as consistent hashing (only a removed/added node's keys move) with a simpler mental model and no ring data structure to maintain, at the cost of an O(N) scan per lookup instead of an O(log N) ring lookup — fine for small-to-medium node counts (this is roughly Google's Maglev load balancer's lineage, though Maglev itself precomputes a lookup table rather than scanning at request time).

Neither replaces application-level hot-key handling (see Gotchas) — they distribute *ownership*, not per-key traffic.

---

## Redis Cluster: A Related but Different Design

Redis Cluster does **not** use a hash ring. It uses 16384 fixed hash slots — every key maps to `CRC16(key) mod 16384`, and slots are assigned to nodes as static, explicit ranges (not derived from hashing node identities onto a ring at all). Resharding means explicitly migrating whole slots between nodes; there's no "next node clockwise" concept and no virtual-node mechanism because the slot space is already fixed and coarse-grained by design. See `caching/redis-vs-memcached.md` for the actual `CLUSTER KEYSLOT` / `CLUSTER SHARDS` walkthrough — worth reading side-by-side with this doc specifically because the two mechanisms solve the same problem (bounded remapping on membership change) with genuinely different data structures, and conflating "Redis Cluster uses consistent hashing" is a common but incorrect simplification.

---

## Key Gotchas

- **Vnode count is a real tradeoff, not "more is always better"**: more vnodes per physical node improves load balance but increases the metadata every node must track (ring position table) and, in systems with anti-entropy/repair (e.g. Cassandra), increases the number of node-pairs involved in every join/leave/repair operation — this is why Cassandra's own guidance moved from a flat 256-per-node default toward much lower counts (single digits, per DataStax recommendations) once token-allocation algorithms could hit good balance without brute-forcing it via point count.
- **Consistent hashing doesn't solve the hot-key problem**: if one key (or a few) receives disproportionate traffic, the node owning that key is overloaded regardless of how evenly the *keyspace* is distributed — ring balance is a statement about key *count* per node, not request *rate* per key. Fixes are orthogonal (client-side key splitting/sharding a hot key into `key#0..key#N`, request-level caching in front of the store, or bounded-load hashing above) — the ring itself won't rebalance around a hot key.
- **Redis Cluster's 16384 slots is a deliberately simpler, coarser design, not a hash ring**: don't reason about Redis Cluster resharding using ring-arc intuition ("the new node takes the arc between it and its counterclockwise neighbor") — slots are explicitly reassigned in whole units by an operator or rebalancer, not derived from node positions on a continuous hash space. See the cross-reference above for the actual mechanism.
- **"Bounded to ~1/N" is an expectation over reasonably-spaced points, not a per-event guarantee**: with few physical nodes and no virtual nodes, a single join/leave can still move a disproportionate share of keys if that node's arc happened to be unusually large — this is the same root cause as the load-imbalance problem above, and virtual nodes fix both simultaneously because they're the same underlying issue (uneven point spacing).
- **Replication factor complicates "who owns a key" beyond a single successor**: real systems (Dynamo, Cassandra) don't stop at the first clockwise node — they replicate to the next `R-1` distinct physical nodes walking clockwise, so a node join/leave event touches every replica set the affected arc participates in, not just one owner. This doc describes single-owner ring mechanics; production systems layer replication on top of it.

---

*Grounded against Amazon's 2007 Dynamo paper (SOSP, "Dynamo: Amazon's Highly Available Key-value Store") for the original consistent-hashing-plus-virtual-nodes rationale; Apache Cassandra's architecture docs and DataStax/community guidance for vnode defaults and their evolution (256 historically, single-digit counts now common with token-allocation-aware placement); and current (2026) community writeups on rendezvous/Maglev hashing as the bounded-load alternative. Redis Cluster's slot mechanism is grounded via `caching/redis-vs-memcached.md` in this KB, which inspects it directly via `CLUSTER KEYSLOT`/`CLUSTER SHARDS`. The exact recommended vnode count (4-8 vs 16 vs 256) varies by source and Cassandra version — treat any specific number here as illustrative of the tradeoff direction, not a current default to copy without checking the deployed version's own guidance.*
