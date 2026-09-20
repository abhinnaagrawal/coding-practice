# Distributed Transactions: 2PC, 3PC, and the Saga Pattern

## 30-Second Intuition

A distributed transaction has to make several independent services agree on the same outcome — all commit, or all abort — even though any one of them (or the network between them) can fail mid-decision. Two-Phase Commit (2PC) gets strong atomicity by having a coordinator ask everyone to vote, then telling everyone the verdict; the catch is that a participant who has voted "yes" must hold its locks and wait, and if the coordinator dies between the vote and the verdict, that participant is stuck — blocked, not just delayed, until the coordinator (or a human) comes back. This single failure mode is why almost no production microservice architecture uses 2PC across service boundaries: it trades availability for consistency in a way that doesn't survive real infrastructure failures. The pragmatic industry answer is the **Saga pattern** — a sequence of local transactions, each independently committed, with a compensating action defined for every step so a later failure can be undone by *doing something new*, not by rolling back time. The operational fact to retain: Sagas buy availability by giving up atomicity — intermediate, partially-applied states become visible to the rest of the system while the saga is in flight.

---

## Two-Phase Commit: Prepare, Then Commit

2PC coordinates a single atomic decision across N participants using a dedicated coordinator process (which may be one of the participants, or a separate transaction manager).

```
                    COORDINATOR                    PARTICIPANT A   PARTICIPANT B   PARTICIPANT C
                         │                                │               │               │
  PHASE 1 (vote)         │──── PREPARE ──────────────────▶│               │               │
                         │──── PREPARE ────────────────────────────────▶│               │
                         │──── PREPARE ──────────────────────────────────────────────▶│
                         │                                │               │               │
                         │        each participant runs its local transaction up to the   │
                         │        point of commit, writes an undo/redo log record, and     │
                         │        HOLDS its locks — but does not commit or release yet     │
                         │                                │               │               │
                         │◀─── VOTE-YES ──────────────────│               │               │
                         │◀─── VOTE-YES ────────────────────────────────│               │
                         │◀─── VOTE-YES ──────────────────────────────────────────────│
                         │                                │               │               │
                         │  coordinator writes its OWN decision to durable log HERE       │
                         │  (this is the point of no return)                              │
                         │                                │               │               │
  PHASE 2 (decide)       │──── COMMIT ────────────────────▶│               │               │
                         │──── COMMIT ──────────────────────────────────▶│               │
                         │──── COMMIT ──────────────────────────────────────────────────▶│
                         │                                │               │               │
                         │        each participant commits, releases locks, replies ACK    │
                         │                                │               │               │
                         │◀─── ACK ───────────────────────│               │               │
                         │◀─── ACK ─────────────────────────────────────│               │
                         │◀─── ACK ───────────────────────────────────────────────────│
```

If **any** participant votes `NO` (or times out during Phase 1), the coordinator sends `ABORT` to everyone instead of `COMMIT`, and every participant rolls back and releases its locks. Voting is unanimous-yes-required: one dissenter aborts the whole transaction.

The critical asymmetry: once a participant votes yes, it has given up its right to unilaterally abort. It must wait for the coordinator's word. That's the design choice that makes the blocking problem possible.

---

## The Blocking Problem: A Concrete Failure Trace

Business scenario: an order-processing transaction spans three services — **Inventory**, **Payment**, **Shipping** — coordinated by a Transaction Manager (TM) using 2PC.

```
t=0.000s  TM → Inventory: PREPARE(reserve 1x SKU-4471)
t=0.000s  TM → Payment:   PREPARE(charge $89.00 to card ****1234)
t=0.000s  TM → Shipping:  PREPARE(reserve shipping slot)

t=0.040s  Inventory: reserves unit, writes undo log, locks the row, replies VOTE-YES
t=0.055s  Payment:   places an auth hold on the card, replies VOTE-YES
t=0.061s  Shipping:  reserves a slot, replies VOTE-YES

t=0.061s  TM has all three YES votes. TM writes "COMMIT" to its own durable log.
          ── TM PROCESS CRASHES right here, before sending a single COMMIT message ──

t=0.061s+ STATE ON EACH PARTICIPANT:
          Inventory: row locked, unit reserved-but-not-confirmed, waiting for TM
          Payment:   auth hold live, waiting for TM to say capture or void
          Shipping:  slot held, waiting for TM

          None of the three can decide unilaterally:
          - They can't commit: they don't know if a peer voted NO.
          - They can't abort: they don't know if a peer already got COMMIT and
            already released its lock — aborting now would violate atomicity.
          - They CAN ask each other ("did you hear from the TM?"), but if none
            of them received Phase 2 either, that tells them nothing about the
            TM's actual decision — only that the TM hasn't told anyone yet.

t=5 min   Inventory's row lock is still held. Any other transaction touching
          SKU-4471 blocks behind it. Payment's auth hold sits unresolved,
          eating into the card issuer's hold-duration limit. Shipping's slot
          reservation blocks other orders from claiming it.

t=?       TM process is restarted (by an operator, or by orchestration/K8s).
          It reads its own durable log, sees "COMMIT" was already decided,
          and re-sends COMMIT to all three. They finally commit and release.
```

The window between "coordinator durably decided" and "coordinator told everyone" is unavoidable — no message is instant — and if the coordinator dies inside that window, participants are **blocked**, not merely delayed: they cannot make local progress by any local decision procedure, only by waiting for external recovery. A network partition that separates the TM from all three participants produces the identical symptom without any process actually crashing — this is why the blocking problem is a property of the protocol, not just "coordinator crash resilience."

---

## Three-Phase Commit: The Attempted Fix, and Why It Isn't Used

3PC inserts a **pre-commit** phase between vote and commit: `PREPARE → PRE-COMMIT → COMMIT`. The idea is that once every participant has acknowledged pre-commit, they collectively hold enough information to decide the outcome among themselves, even if the coordinator vanishes — because "everyone got to pre-commit" implies no one voted no, so participants can safely time out and commit rather than block indefinitely.

This works only under an assumption 3PC bakes in and real networks don't honor: **synchronous timeouts** — that a message not received within a bounded time means the sender failed, not that the network is merely slow or partitioned. Under an actual network partition, one subset of participants can time out and decide to commit while a disjoint subset times out and decides to abort — the protocol produces a split-brain, inconsistent decision rather than a blocked-but-safe one. It also costs a full extra network round trip on every transaction, all the time, to protect against a failure mode it doesn't even fully eliminate. The practical consequence: 3PC is taught as the "next logical step" from 2PC and is genuinely non-blocking under a synchronous, partition-free fault model, but it is close to unused in production distributed systems — the industry instead moved either toward quorum-based consensus (Raft/Paxos, which explicitly handle partitions by requiring a majority) for the cases that truly need cross-node agreement, or away from cross-service ACID transactions entirely via the Saga pattern below.

---

## Saga Pattern: The Same Scenario, Rebuilt Without a Blocking Coordinator

Same business transaction — reserve inventory, charge payment, create shipment — reimplemented as a saga: a sequence of independently-committed local transactions T1, T2, T3, each paired with a compensating transaction C1, C2, C3 that semantically undoes it if a later step fails.

```
Forward path:     T1: Reserve Inventory  →  T2: Charge Payment  →  T3: Create Shipment
Compensations:    C1: Release Reservation ← C2: Refund Payment  ← C3: (n/a — T3 failed)
```

### Orchestration style: one saga coordinator drives every step

```
                     SAGA ORCHESTRATOR
                            │
        1. Reserve Inventory│──────▶ Inventory Service
                            │◀────── OK (reservation confirmed, LOCAL commit)
                            │
        2. Charge Payment   │──────▶ Payment Service
                            │◀────── OK (charge captured, LOCAL commit)
                            │
        3. Create Shipment  │──────▶ Shipping Service
                            │◀────── FAILS (no carrier capacity)
                            │
        orchestrator sees the failure and walks the completed steps BACKWARD:
                            │
        4. Refund Payment   │──────▶ Payment Service     (compensating action C2)
                            │◀────── OK
        5. Release Reservation──────▶ Inventory Service  (compensating action C1)
                            │◀────── OK
                            │
        saga ends in "compensated / failed" state — a terminal, known-consistent
        outcome, reached without any participant blocking on a coordinator crash
        window, because each step (T1, T2, C2, C1) was its own committed local
        transaction the instant it ran.
```

An orchestrator here is a stateful workflow engine tracking "which step are we on" — the same role a tool like **Argo Workflows** (see `orchestration/argo-workflows.md`) plays for compute pipelines, with the DAG/Steps template and its per-node status in `Workflow.status.nodes` standing in for the saga's step log. The difference from 2PC's coordinator: the orchestrator's crash mid-saga does not leave participants holding locks — T1 and T2 already committed for real, and a restarted orchestrator simply re-reads its own persisted step log and resumes issuing compensations or forward steps from wherever it left off, the same recovery shape argo-workflows.md describes for its own controller reconciliation loop.

### Choreography style: each service reacts to the previous service's event

```
Inventory Service          Payment Service            Shipping Service
      │                          │                          │
      │─ publish: ─────────────▶│                          │
      │  InventoryReserved       │                          │
      │                          │─ publish: ──────────────▶│
      │                          │  PaymentCharged            │
      │                          │                          │─ attempts create
      │                          │                          │  shipment → FAILS
      │                          │◀─ publish: ───────────────│
      │                          │  ShipmentFailed            │
      │                          │  (compensate: refund)      │
      │◀─ publish: ──────────────│                          │
      │  PaymentRefunded          │                          │
      │  (compensate: release     │                          │
      │   reservation)            │                          │
```

No service holds a global view of the saga; each one only knows "when event X arrives, run my local transaction (or my compensation) and publish my own event." A Kafka topic per event type (see `streaming/kafka.md`) is the typical backbone — the append-only, replayable log gives every service an independent, durable subscription to the events it cares about, and the consumer-group offset model means a service that was down when `ShipmentFailed` was published still processes it (and issues its compensation) once it comes back, rather than the event being lost the way a synchronous RPC callback would be.

---

## 2PC vs Saga: Consistency Strength vs Availability

| | Two-Phase Commit | Saga |
|---|---|---|
| Atomicity | True atomic commit — all-or-nothing, enforced by holding locks until the coordinator's verdict | No true atomicity — each local transaction commits independently; failure is handled by compensations, not rollback |
| Isolation | Full — locks held across the whole transaction hide intermediate state from other readers | None by default — other transactions can read inventory-reserved-but-payment-not-yet-charged state mid-saga |
| Availability under coordinator/orchestrator failure | Participants **block**, holding locks, until the coordinator recovers or is manually resolved | Already-committed steps stay committed; a restarted orchestrator (or a replayed event log) resumes from durable state — no participant blocks |
| Failure recovery | Requires the coordinator's durable decision log; manual intervention if that log is lost | Requires idempotent, well-defined compensations for every step; a mid-saga crash is resolved by re-running from persisted saga/event state |
| Where it fits | Tightly-coupled systems under one operational boundary — e.g. a single database's multi-shard commit, XA transactions inside one datacenter | Cross-service, cross-team, cross-failure-domain flows — the default shape for microservice business transactions |

2PC is the stronger guarantee bought at the cost of availability during coordinator failure — this is a direct instance of the CAP/PACELC tradeoff, not an implementation detail that better engineering fixes. Sagas invert the trade: they give up atomicity and isolation to keep every individual step available and progressing, pushing the consistency work into designing correct compensations instead of a locking protocol.

---

## Key Gotchas

- **Sagas give up atomicity, not just "eventual" atomicity**: between T1 committing and T3 failing, the system is in a real, externally-visible partial state (inventory reserved, payment charged, no shipment yet). Any other process reading inventory or payment state during that window sees it — design read paths (dashboards, other services) with this in mind, or add a "pending" status field so readers can distinguish in-flight sagas from settled ones.
- **Compensating actions are not rollbacks — they must be new, forward-moving operations, and idempotent**: `RefundPayment` doesn't erase the original charge, it issues a new refund transaction; it must handle being called twice (e.g. after an orchestrator crash-and-resume, or a choreography event redelivery) without double-refunding. Design every compensation with a natural idempotency key (the original saga/transaction ID), not just "run it again is probably fine."
- **Not every action is compensable**: sending a physical package or an email can't be truly undone by a compensating step — only mitigated (a follow-up "please disregard" email, a return-shipping label). Identify these non-compensable steps up front and either sequence them last in the saga or add a manual-intervention fallback.
- **2PC's coordinator is a structural single point of failure, not just an operational risk to mitigate with HA**: even a highly-available coordinator (replicated, fast failover) has a nonzero window between writing its commit decision and notifying all participants; a crash or partition inside that exact window blocks participants regardless of how good the coordinator's own uptime is.
- **3PC doesn't actually fix the problem it targets under real network conditions**: it assumes synchronous, bounded-delay message delivery; under an actual network partition it can produce a split-brain (part of the cluster commits, part aborts) rather than blocking safely — this is why it's rarely deployed despite being the "textbook fix" for 2PC's blocking problem.
- **Choreography sagas can produce implicit, hard-to-trace event chains**: with no central coordinator, "what triggers what" lives only in each service's event handlers — debugging a stuck or wrongly-compensated saga means reconstructing the flow from a distributed event log (e.g. tracing correlation/saga IDs across Kafka topics) rather than reading one orchestrator's step log. This is the main reason teams pick orchestration once a saga grows past 3-4 steps or needs conditional branching.

---

*Grounded against community and vendor writeups on 2PC/3PC blocking behavior (Medium/JavaCodeGeeks 2026 pieces on 2PC's coordinator SPOF, GeeksforGeeks/SystemDesign.academy on 3PC's partition limitation) and Saga-pattern references (microservices.io's canonical Saga pattern writeup, Temporal's and ByteByteGo's 2026 orchestration-vs-choreography coverage). No specific product-version claims are made in this doc; the underlying protocol behavior (2PC blocking, 3PC's partition failure mode, Saga compensation semantics) is decades-stable computer-science content, not something that goes stale with releases.*
