# The Eight Fallacies of Distributed Computing

## 30-Second Intuition

In 1994, L. Peter Deutsch (building on four assumptions Bill Joy and Dave Lyon had already named) listed the false assumptions almost every engineer makes the first time they write code that talks to another machine; James Gosling added an eighth around 1997. The list hasn't aged — every fallacy below still causes real production incidents in 2026, and every one of them is the actual *reason* some mechanism elsewhere in this KB exists. The one fact that matters operationally: **this list isn't trivia, it's a checklist** — every fallacy below has a specific, already-documented remedy and a specific tradeoff that remedy costs you. Believing the fallacy isn't the mistake; not knowing what it costs to stop believing it is.

---

## The List, With Remedy and Tradeoff for Each

### 1. "The network is reliable."

**Reality**: packets get dropped, connections reset, requests silently disappear — at any scale, network failure is a when, not an if.

**Remedy**: timeouts + retries + idempotency, so a dropped request can be safely retried instead of leaving the system in an unknown state.

**Tradeoff**: retries without idempotency turn a transient failure into a *permanent* duplicated side effect (double charge, double insert) — the remedy is only safe if paired with idempotency, which itself costs engineering effort (idempotency keys, dedup logic) on every retryable operation. See [`reliability/multi-az-dr-and-failure-patterns.md`](multi-az-dr-and-failure-patterns.md)'s "Timeouts, Retries, Backoff+Jitter" section for the AWS Builders' Library-sourced specifics, and [`data-structures/lsm-trees.md`](../data-structures/lsm-trees.md)-adjacent systems' use of idempotency keys in practice.

### 2. "Latency is zero."

**Reality**: even light-speed-limited network round-trips are orders of magnitude slower than an in-process function call — a "quick" cross-service call is never actually free.

**Remedy**: batching (fewer, larger round-trips instead of many small ones), async/non-blocking I/O so a slow call doesn't block a whole thread, and caching to avoid the round-trip entirely when possible.

**Tradeoff**: batching trades latency-per-item for fewer round-trips overall (worse p50 for an individual small request, better aggregate throughput) — this is exactly the tension Kafka's producer batching (`linger.ms`/`batch.size`) makes explicit in [`streaming/kafka.md`](../streaming/kafka.md)'s high-to-low walkthrough. Caching trades staleness risk for latency — see the consistency-model tradeoffs in [`distributed-systems/consistency-models.md`](consistency-models.md).

### 3. "Bandwidth is infinite."

**Reality**: network throughput is a real, shared, finite resource — a chatty protocol or an oversized payload competes with every other flow on the same link.

**Remedy**: compact wire formats (binary over text), compression, and reducing round-trip chattiness (batch/multiplex instead of one request per item).

**Tradeoff**: compact binary formats cost readability/debuggability (you can't `curl` and eyeball a Protobuf payload the way you can JSON) and require schema coordination between producer and consumer — this exact tradeoff, and Protobuf's actual wire-format mechanics, is worked out with a hand-verified byte-count example in [`networking/grpc.md`](../networking/grpc.md).

### 4. "The network is secure."

**Reality**: any hop your data crosses is a potential interception/injection point unless explicitly secured — assuming the network layer protects you is how internal-only services end up wide open.

**Remedy**: TLS everywhere (encrypt in transit, don't rely on network perimeter alone — zero-trust framing), explicit authorization checks at the service boundary rather than assuming "it's internal, so it's safe."

**Tradeoff**: TLS costs a real handshake round-trip (mitigated but not eliminated by TLS 1.3's 1-RTT/0-RTT resumption, per [`networking/networking-layers.md`](../networking/networking-layers.md)) and, for systems like Kafka, disables the zero-copy `sendfile()` fast path entirely once encryption is on (per [`streaming/kafka.md`](../streaming/kafka.md)'s gotchas) — security and the fastest-possible I/O path are in direct, named tension there, not an abstract tradeoff.

### 5. "Topology doesn't change."

**Reality**: nodes join, leave, fail, and get replaced continuously at any real scale — hardcoding "which node owns what" breaks the moment the cluster's membership changes even slightly.

**Remedy**: consistent hashing (bound the fraction of keys that remap when membership changes) and consensus-based membership/leader election (agree on the current topology instead of assuming a fixed one).

**Tradeoff**: consistent hashing's virtual-node tuning trades ring-balance quality for rebalancing overhead (worked out numerically in [`distributed-systems/consistent-hashing.md`](consistent-hashing.md)); consensus-based membership changes (Raft configuration changes) cost availability during the reconfiguration window — see the leader-election unavailability window in [`distributed-systems/consensus-raft-vs-paxos.md`](consensus-raft-vs-paxos.md) and [`coordination/etcd.md`](../coordination/etcd.md)'s "leader elections pause writes for their duration" gotcha.

### 6. "There is one administrator."

**Reality**: real systems span teams, organizations, and trust boundaries — nobody has unilateral authority to change every part of the system at once, which breaks any design that assumes coordinated, simultaneous rollout.

**Remedy**: backward/forward-compatible schema evolution (so producers and consumers can be upgraded independently, not in lockstep) and explicit versioned contracts between services.

**Tradeoff**: schema evolution discipline (field IDs instead of positional/name matching, no field-number reuse) costs upfront design rigor and forecloses certain "just change the type" shortcuts — see [`data-formats/apache-iceberg.md`](../data-formats/apache-iceberg.md)'s field-ID-based schema evolution and [`networking/grpc.md`](../networking/grpc.md)'s protobuf field-number-reuse gotcha, both direct instances of designing for "I don't control every consumer's upgrade schedule."

### 7. "Transport cost is zero."

**Reality**: serialization/deserialization, connection setup, and the actual data-movement work all cost real CPU and time — moving data is never free, even ignoring network latency itself.

**Remedy**: efficient serialization (binary formats, schema-driven codecs), connection reuse/pooling instead of a fresh connection per request, and — the most aggressive version of this remedy — avoiding the transport hop altogether via colocation.

**Tradeoff**: colocation-for-speed is precisely the tradeoff [`storage/hadoop.md`](../storage/hadoop.md)'s entire data-locality-scheduling design makes (move compute to data instead of data to compute, to avoid paying transport cost at all) — and precisely the tradeoff the S3-backed lakehouse model explicitly gives up in exchange for elastic, disaggregated storage, as covered in [`storage/s3.md`](../storage/s3.md)'s comparative section. Connection pooling/reuse trades a small amount of idle-connection resource cost for avoiding repeated handshake overhead — see the TLS handshake cost discussion in [`networking/networking-layers.md`](../networking/networking-layers.md).

### 8. "The network is homogeneous."

**Reality**: real networks mix protocols, hardware, and vendors — assuming every hop behaves identically (same MTU, same latency profile, same reliability) breaks the moment traffic crosses a boundary that isn't.

**Remedy**: protocol negotiation/version fallback (don't assume every peer supports your newest protocol) and abstraction layers that hide heterogeneous transport details behind a uniform interface.

**Tradeoff**: negotiation/fallback logic is itself extra complexity and an extra round-trip in the worst case (HTTP/2→HTTP/1.1 fallback, TLS version negotiation) — the exact mechanics of this layered negotiation, and why HTTP/3/QUIC's adoption is still partial specifically because of this heterogeneity problem, are covered in [`networking/networking-layers.md`](../networking/networking-layers.md)'s HTTP/1.1 vs 2 vs 3 comparative section.

---

## Comparative: Why This List Still Holds Cross-Cutting Value

Every other doc in `distributed-systems/`, `networking/`, `reliability/`, and half of `streaming/`/`storage/`/`compute/` in this KB is, at some level, a fully-worked-out answer to one of these eight lines — that's *why* this doc is short: it's an index, not new content. The value of keeping it as its own doc rather than folding it into `reliability/multi-az-dr-and-failure-patterns.md` is that the fallacies are the *cause*, and the reliability doc's patterns (timeouts, circuit breakers, static stability) are the *effect* — the fallacies explain **why** you'd ever need a circuit breaker or a retry-with-jitter policy in the first place, one layer more foundational than the patterns themselves.

---

## Key Gotchas

- **Fixing one fallacy can silently reintroduce another**: adding retries (fixing #1, "network is reliable") without idempotency can violate #6's implicit assumption of coordinated behavior if a retried, non-idempotent call reaches a downstream system another team owns and doesn't expect duplicate calls from.
- **The fallacies interact, they don't apply independently**: TLS (#4's remedy) directly costs you #2's remedy space (adds latency) and, for Kafka specifically, forecloses the zero-copy path relevant to #7 (transport cost) — security, latency, and transport-cost tradeoffs are not three separate line items to optimize in isolation.
- **"We're internal only" is exactly the #4 trap in its most common modern form** — internal service-to-service traffic is not automatically safe traffic; zero-trust architectures exist specifically because #4 keeps getting assumed inside a perimeter that itself keeps getting breached.
- **New protocols don't retire old fallacies, they just move where the fallacy bites**: HTTP/3/QUIC fixes TCP-level head-of-line blocking (partially addressing #1/#2 in a new way) but introduces new heterogeneity (#8) because not every network path/middlebox handles UDP-based QUIC traffic identically — see the HTTP/3 adoption-percentage uncertainty flagged in [`networking/networking-layers.md`](../networking/networking-layers.md).

---

*Grounded against the canonical list (Deutsch, 1994, extended by Gosling ~1997) via Wikipedia's "Fallacies of distributed computing" entry and corroborating 2025-2026 retrospective writeups (Ably, APNIC blog) confirming the list and attribution are unchanged and still actively cited as of September 2026. All remedy/tradeoff mappings above point to KB docs already fact-checked and grounded individually — this doc adds no new version-pinned technical claims of its own.*
