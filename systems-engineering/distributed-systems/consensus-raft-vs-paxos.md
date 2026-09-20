# Consensus Primitives: Raft vs Paxos

## 30-Second Intuition

Both Raft and Paxos solve the same problem — get a set of machines to agree on a sequence of values (a replicated log) even when some of them crash or the network drops messages — by requiring every decision to be acknowledged by a **majority quorum**, so any two decisions are guaranteed to share at least one witness. Paxos solves this in the abstract: any node can propose, any node can lead, and leadership itself is just an emergent side effect of who successfully drives a round of the protocol — which makes it powerful but genuinely hard to reason about (the original paper needed a follow-up paper to explain the follow-up paper). Raft solves the *identical* problem by making one structural bet: **elect a strong leader first, then treat replication as a solved problem — the leader always wins over followers, and log entries only ever flow one direction (leader → follower).** The one fact that matters operationally: **Raft trades a small amount of theoretical flexibility for the property that a competent engineer can implement it correctly from the paper alone — this is exactly why virtually every new consensus system built since 2014 (etcd, Consul, CockroachDB, TiKV, Kafka's KRaft) chose Raft, while Paxos survives mainly in systems old enough to predate Raft (Chubby) or that need Paxos's more flexible quorum/leaderless properties badly enough to pay the complexity tax (Spanner's Paxos groups)**.

For how Raft is actually consumed by a real system (log-as-source-of-truth, MVCC, watch mechanism, disk-fsync sensitivity), see `coordination/etcd.md` — this doc stays one level down, at the algorithm itself.

---

## Raft: Strong Leadership as the Organizing Principle

Raft decomposes consensus into three independent sub-problems, each easy to reason about on its own:

1. **Leader election** — exactly one node is the leader at any given time, or no leader exists (transient).
2. **Log replication** — the leader is the only node that accepts writes and pushes them to followers, in order.
3. **Safety** — a set of invariants (election restriction, commitment rules) that together guarantee no two leaders in the same term, and no committed entry is ever lost or reordered.

Every node is in one of three states:

```
   ┌──────────┐   election timeout    ┌───────────┐
   │ Follower │ ─────────────────────▶│ Candidate │
   └──────────┘                       └───────────┘
        ▲                                  │  │
        │ discovers current              wins│  │ split vote /
        │ leader or higher term          majority  higher term seen
        │                                  │  │  discovered
        │           ┌──────────┐           │  │
        └───────────│  Leader  │◀──────────┘  │
                     └──────────┘              │
                          ▲                     │
                          └─────────────────────┘
                        (retries election, new term)
```

**Terms** are Raft's logical clock: a monotonically increasing integer that every message carries. Every election bumps the term by 1. If a node ever sees a message with a higher term than its own, it immediately steps down to follower and adopts that term — this is the single mechanism that resolves "who's actually in charge" without any node needing global knowledge.

### Leader Election

```
Follower's view while leader is healthy:
  reset randomized election timer (150-300ms typical) on every
  valid AppendEntries (heartbeat) or RequestVote received

Election timer fires (no heartbeat seen in time):
  1. increment currentTerm
  2. transition to Candidate
  3. vote for self
  4. send RequestVote(term, candidateId, lastLogIndex, lastLogTerm)
     to every other node, in parallel

Each recipient grants its vote iff, in this term, it hasn't already
voted for someone else AND the candidate's log is at least as
up-to-date as its own (last entry's term, then index, as tiebreak)

Candidate becomes Leader once it holds votes from a MAJORITY
(including its own vote) — e.g. 3 of 5 nodes
```

The randomized timeout is what prevents every follower from becoming a candidate simultaneously forever — whichever follower's timer fires first usually gets its RequestVotes out before others time out, so it typically wins outright. This is a probabilistic, not guaranteed, tiebreak — see the split-vote example below.

### Log Replication

```
Client → Leader: Append(x=5)

Leader:
  1. appends {term: T, index: I, cmd: x=5} to its own log (uncommitted)
  2. sends AppendEntries(term, prevLogIndex, prevLogTerm, [entry], leaderCommit)
     to every follower in parallel

Follower:
  - rejects if its log doesn't contain an entry at prevLogIndex
    matching prevLogTerm (the "log matching" consistency check)
  - on accept: appends entry, truncating any conflicting suffix
    it may have had (from a former, now-stale leader)
  - replies success/failure

Leader:
  - once a MAJORITY of followers have replicated the entry,
    it is COMMITTED — leader applies it to its state machine
    and returns to the client
  - leaderCommit index piggybacks on the next heartbeat, telling
    followers which entries are now safe to apply locally
```

The **log matching property** (if two logs agree on the term of an entry at some index, they're identical in every entry up to that index) is what lets a follower with a diverging suffix be repaired by simple truncate-and-overwrite, rather than a merge algorithm — this is a direct consequence of "only leaders send entries; entries never flow follower→follower."

---

## Basic Paxos: Proposer / Acceptor / Learner

Basic (or "Single-Decree") Paxos agrees on exactly **one** value, ever, per instance. There is no built-in concept of leader or log — any node can act as **proposer** for any instance, and multiple proposers can compete concurrently.

```
  Proposer                 Acceptors (majority needed)         Learner
     │                            │                               │
     │──── Phase 1a: Prepare(n) ─▶│  (n = proposal number,        │
     │                            │   globally unique per         │
     │                            │   proposer, monotonic)        │
     │                            │                               │
     │◀─ Phase 1b: Promise(n, ────│  Acceptor promises never to   │
     │    prior accepted value)   │  accept any proposal < n;     │
     │                            │  returns highest-numbered     │
     │                            │  value it already accepted    │
     │                            │  (if any)                     │
     │                            │                               │
     │── Phase 2a: Accept(n, v) ─▶│  v = proposer's own value,    │
     │                            │  UNLESS a prior accepted      │
     │                            │  value came back in Promise — │
     │                            │  then v MUST be that value    │
     │                            │  (this is what makes Paxos    │
     │                            │  safe under concurrent         │
     │                            │  proposers)                   │
     │                            │                               │
     │◀─ Phase 2b: Accepted(n,v) ─│  acceptor accepts unless it's  │
     │                            │  since promised a higher n     │
     │                            │                               ▶ once a MAJORITY
     │                            │                                 accept (n, v),
     │                            │                                 v is chosen —
     │                            │                                 learners find out
     │                            │                                 by being told or
     │                            │                                 by polling acceptors
```

Two round trips, minimum, to agree on one value — and that's the *uncontended* case. The subtlety that makes Paxos hard to reason about: a proposer that loses phase 1 to a higher-numbered competitor may have to restart from Prepare with a still-higher number, and if two proposers keep leap-frogging each other's proposal numbers, the protocol can **livelock** (FLP impossibility made concrete — safety is preserved, liveness is not guaranteed without extra mechanism). Nothing in Basic Paxos elects a leader; "leader" only emerges informally if one proposer happens to win consistently.

### Multi-Paxos: Why It's Not Just "Paxos, Repeatedly"

Running Basic Paxos once per log slot would mean two full round trips per write, forever — unacceptable for a replicated log. **Multi-Paxos** optimizes this by having one proposer win **Phase 1 once, for a whole range of future instances** ("I am the distinguished proposer for instances N and up"), then skip straight to Phase 2 (Accept) for each subsequent write — collapsing steady-state replication to one round trip, matching Raft's normal-case cost.

This is the detail most "Paxos vs Raft" comparisons gloss over: **Multi-Paxos is Basic Paxos plus an ad hoc, paper-doesn't-fully-specify leader-stability mechanism bolted on** — the original Paxos paper says almost nothing about how to pick or maintain this distinguished proposer, how it hands off, or how followers catch up after a gap. Every production Paxos system (Chubby, Spanner) has had to independently invent and paper-formalize its own answer to this. Raft, by contrast, specifies leader election, log-gap repair, and leader stability as core parts of the algorithm itself, in one paper. This is the concrete, mechanical reason "Raft is Multi-Paxos with the leader-election and log-repair machinery made explicit and mandatory" is a fair characterization — not just a soundbite.

---

## Worked Example: 5-Node Raft Cluster, Split Vote, Term Increment

Cluster: nodes **A, B, C, D, E**. Current state: term 4, A is leader, all synced at log index 100.

**t=0**: A (leader) suffers a long GC pause / partial network partition — it stops sending heartbeats to B, C, D, E, though it hasn't crashed.

**t=180ms**: Each follower's randomized election timeout (150-300ms window) is counting down independently. B and D happen to have drawn nearly identical short timeouts (~180ms) due to randomization variance; C, E have longer ones (~250ms, ~270ms).

**t=181ms**: B's timer fires first (by 1ms).
- B: `currentTerm 4 → 5`, transitions to Candidate, votes for itself (1 vote), sends `RequestVote(term=5, candidateId=B, lastLogIndex=100, lastLogTerm=4)` to A, C, D, E.

**t=182ms**: D's timer fires (it hadn't yet received B's RequestVote — network delay).
- D: `currentTerm 4 → 5`, transitions to Candidate, votes for itself (1 vote), sends `RequestVote(term=5, candidateId=D, lastLogIndex=100, lastLogTerm=4)` to A, B, C, E.

**t=185-190ms**: Messages arrive.
- C receives B's RequestVote first (term 5 > its term 4): grants vote to B. C's state: `votedFor=B, term=5`. Moments later C receives D's RequestVote for the *same term 5* — but C already voted for B this term, so it **rejects** D (Raft allows at most one vote per term per node).
- E receives D's RequestVote first: grants vote to D. `votedFor=D, term=5`. Then rejects B's request for the same reason.
- B rejects D's vote request (B already voted for itself in term 5); D rejects B's for the same reason.
- A (still partitioned/stalled) sees neither request in time.

**Tally at term 5**: B has 2 votes (itself + C). D has 2 votes (itself + E). Neither reaches the majority of 3. **This is a split vote** — term 5 ends with no leader.

**t=210ms** (B's timeout, again, drawn fresh from the random range since it got no majority):
- B's election timer (now waiting for votes, also bounded by a timeout) expires with no leader emerging. B increments again: `currentTerm 5 → 6`, re-enters Candidate, votes for itself, sends new `RequestVote(term=6, ...)`.
- D independently does the same shortly after (D's retry timer, different random draw): `currentTerm → 6` — but by the time D's message goes out, it may see B's term-6 request first and, since terms are equal and B's log is at least as up to date, D grants its vote to B instead of re-becoming a candidate (Raft followers/candidates always defer to an equal-or-higher term with an at-least-as-current log, once they observe it first).
- C and E, both already at term 5, see B's `RequestVote(term=6)`, step up their term to 6, and grant their votes (their logs are equally caught up, and neither has already voted in term 6).

**Tally at term 6**: B now has votes from B, C, D, E — 4 of 5, a clear majority. **B becomes leader for term 6.**

- B immediately sends `AppendEntries(term=6, ...)` heartbeats to all other nodes, including the now-recovering A. A, on receiving a heartbeat with term 6 > its own stale term 4, steps down (it was never aware it lost leadership) and updates to term 6, follower.
- Any client write submitted during the term-5 window that never reached a majority is simply never committed — it's discarded/never acknowledged; the client would have to retry, and it succeeds only after B is established as term-6 leader.

This is a real, common scenario in 5-node (or any even-contention) clusters: split votes are why Raft's randomized election timeout exists at all — without randomization, ties like B/D would repeat indefinitely every term.

---

## Raft vs Paxos: Comparative

| Dimension | Raft | Paxos (Basic / Multi-Paxos) |
|---|---|---|
| **Understandability** | Designed explicitly for this; the Raft paper included a user study showing students scored ~25% better on a Raft-based exam than a Paxos-based one, after equivalent instruction. | Famously hard to teach — Lamport's own "Paxos Made Simple" was written because the original 1998 paper wasn't understood by most readers; Multi-Paxos's leader-stability layer isn't even in the base paper. |
| **Leadership model** | Strong leader is foundational and mandatory — log entries flow strictly leader→follower, never the reverse, and this single rule is what makes log repair a truncate-and-copy operation. | Basic Paxos has no leader concept at all — any node can be a proposer for any instance. Multi-Paxos adds a de facto leader (the distinguished proposer) as an optimization, not a structural requirement — some Paxos variants (EPaxos, see below) deliberately avoid a leader altogether. |
| **Message complexity (steady state)** | 1 round trip per committed entry (leader → followers → majority ack), once a stable leader exists. | Basic Paxos: 2 round trips (Prepare/Promise, then Accept/Accepted) per value, unless a stable distinguished proposer is established, in which case Multi-Paxos also drops to 1 round trip — same asymptotic cost as Raft in steady state. |
| **Why the strong-leader requirement exists (Raft)** | Because leader election, log matching, and commitment rules are designed together as one package — the leader-only-append rule is what lets Raft guarantee "if a log entry is committed, every future leader's log contains it" (the Leader Completeness property) with a simple election restriction (candidates must have logs at least as up-to-date as a majority) rather than a runtime reconciliation protocol. | Because Basic Paxos was built to tolerate concurrent, uncoordinated proposers *by design* — the two-phase Prepare/Accept structure with proposal numbers is exactly the mechanism that makes concurrent proposals safe without pre-electing anyone. This flexibility is also why reasoning about liveness (who eventually wins) is much harder. |
| **Adoption in new (post-2014) systems** | etcd, Consul, CockroachDB, TiKV, Kafka's KRaft (replacing ZooKeeper) all chose Raft. | New Paxos-family adoption is rarer and usually deliberate: Spanner uses per-shard Paxos groups (values Paxos's flexible quorum/leader-agility properties for a globally distributed setting); Chubby (Google's lock service, predates Raft) still runs Paxos. |
| **Leaderless variants** | Not a native concept — "Raft without a leader" isn't Raft. | EPaxos (Egalitarian Paxos, 2013) generalizes Paxos to let any replica commit commands directly, in one round trip in the common case, using per-command dependency tracking instead of a global log order — removes the single-leader bottleneck for geo-distributed workloads with independent commands. As of 2026 it remains primarily a research/advanced-systems technique (cited in newer work like the 2025 "Making Democracy Work" EPaxos fixes and used in some geo-replicated datastore designs), not a mainstream default the way Raft is. |
| **Flexible/generalized quorums** | Flexible Paxos's insight — that only the *election* quorum and the *replication* quorum need to intersect, not all quorums with each other — applies to Raft too: Raft's election and replication quorums can, in principle, be sized independently as long as they overlap, though mainstream Raft implementations (etcd/Consul/CockroachDB) still use plain majority quorums for both, keeping the guarantee simple rather than chasing the availability/latency wins Flexible Paxos enables. | Flexible Paxos (2016) is the origin of this generalization; it's the theoretical basis some newer systems use to reduce commit-path quorum size below strict majority for one of the two phases. |

---

## Key Gotchas

- **Split votes are expected, not a bug**: with 5+ nodes and simultaneous timeouts, no candidate may reach majority in a given term, forcing another election round at a higher term. Raft's randomized election timeout (150-300ms range is the paper's reference value) exists specifically to make repeated ties statistically unlikely, not impossible — see the worked example above, which needed two full terms to resolve.
- **A vote is per-term, not per-election**: a node grants at most one vote per term, period — this is what prevents both B and D in the worked example from getting a majority in the same term, and it's easy to implement incorrectly by tracking "have I voted in this election" instead of "have I voted in this *term*."
- **Single-decree Paxos ≠ a replicated log**: Basic Paxos agrees on one value. Treating "I implemented Paxos" as "I have a replicated log" skips the entire Multi-Paxos leader-stability and log-catch-up layer that the base paper doesn't specify — this is the single most common source of broken from-scratch Paxos implementations.
- **"Raft is just Multi-Paxos with the details filled in" is directionally true but not literally so**: Raft was independently designed (2014, Ongaro & Ousterhout) around the explicit goal of understandability, not derived from Multi-Paxos — but its steady-state message pattern and guarantees are close enough to Multi-Paxos that the comparison is a legitimate teaching shortcut, as long as you don't imply Raft is merely a renamed Paxos variant (it makes structural choices — e.g. mandatory strong leadership, log matching via prevLogIndex/prevLogTerm checks — that Paxos never requires).
- **A leader election pauses writes, in both protocols, once a leader is lost**: this is not a Raft-specific weakness — any system that stabilizes on a distinguished proposer/leader (which is every production Paxos system too, via Multi-Paxos) blocks new commits during the handoff window. Truly leaderless approaches (EPaxos) trade this away, at the cost of more complex conflict/dependency tracking per command.
- **Term/ballot numbers must never be reused across a restart**: both Raft's `currentTerm` and Paxos's proposal numbers must be durably persisted before responding to any RPC — a node that restarts and reuses a term/ballot number it already used pre-crash can violate safety (e.g., voting twice in what it thinks are two different terms but are actually the same one). This is a common implementation bug, not a protocol flaw.
- **Quorum size assumptions break silently under misconfiguration**: both protocols assume any two quorums intersect (classically, both are simple majorities of the same fixed member set). Reconfiguring cluster membership without a proper joint-consensus/joint-quorum transition (which both Raft and Paxos define precisely for this reason) can produce two non-overlapping quorums during the transition — the textbook "split-brain during reconfiguration" bug.

---

*Grounded via web research as of September 2026. Raft's understandability user-study result and its 2014 design goals are drawn from the original "In Search of an Understandable Consensus Algorithm" paper and are stable, citable facts. Adoption claims (etcd/Consul/CockroachDB/TiKV/Kafka-KRaft on Raft; Chubby/Spanner on Paxos-family protocols) reflect the current, actively-cited state as of 2026 search results — Spanner's continued reliance on Paxos-per-shard and Chubby's continued use of Paxos are both corroborated by 2025-era sources but should be re-verified if a specific version/architecture claim is needed for a deliverable. EPaxos's production-adoption status is the softest claim in this doc: as of 2026 it remains primarily research/advanced-systems territory (recent work like the 2025 OPODIS "Making Democracy Work" paper still frames it as needing fixes/simplification) rather than a mainstream production default — flag this if a reader needs a stronger production citation than "cited in ongoing research."*
