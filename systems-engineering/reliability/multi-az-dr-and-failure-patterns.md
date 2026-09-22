# Multi-AZ, Disaster Recovery & Failure Patterns

## 30-Second Intuition

Reliability engineering at scale breaks into three separable questions. First, how do you survive losing a whole datacenter/AZ (DR strategy). Second, how do you keep a *healthy* system from cascading into failure when one dependency gets slow or partially down (resilience patterns: timeouts, retries, circuit breakers). Third, why does "average latency looks fine" hide the actual user-facing failure mode at scale (tail latency).

The fact that matters most operationally, quoting the AWS Builders' Library directly: **"a statically stable design... keeps working even when a dependency becomes impaired... without requiring any changes"**. A system that behaves *differently* under failure than it does normally is the bigger reliability risk. The failure-mode code path is the path tested least and trusted least, at exactly the moment it matters most.

---

## Disaster Recovery: The Four-Strategy Taxonomy

Source: *AWS Well-Architected Framework — Disaster Recovery of Workloads on AWS* whitepaper ([docs.aws.amazon.com](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)) — this is the canonical source; Azure and GCP's own DR guidance use the same four-tier shape with different names, so this taxonomy generalizes across clouds.

Two terms everything below is measured against, quoting the whitepaper's definitions directly:
- **RTO (Recovery Time Objective)**: "the time it takes after a disruption to restore a business process to its service level... if a disaster occurs at 12:00 PM and the RTO is eight hours, the DR process should restore the business process to the acceptable service level by 8:00 PM."
- **RPO (Recovery Point Objective)**: "the acceptable amount of data loss measured in time" — how much data you can afford to lose, expressed as "how far back does the last good backup/replica go."

```
Cost/complexity increases →, RTO/RPO decreases →

Backup & Restore  →  Pilot Light  →  Warm Standby  →  Multi-Site Active/Active
   hours-days          10s of min      minutes            near-zero
   (cheapest)                                              (most expensive)
```

1. **Backup & Restore**: "periodic data backups to Amazon S3, and if a disaster occurs, you restore EC2 instances and databases from these backups... the lowest-cost option, [but] results in the highest RTO and RPO." Right fit: non-critical workloads, dev/test, anything where hours-to-days of downtime is acceptable.
2. **Pilot Light**: "a minimal version of your core infrastructure is always running in a secondary... region, with data continuously replicated, but other resources remain 'dark' until triggered by infrastructure as code during a failover." The database is warm, replicating continuously. The compute layer is not — you're paying for data currency, not for idle compute capacity.
3. **Warm Standby**: "a scaled-down but fully functional environment in a secondary region. With continuous data replication and active services, it allows for faster failover." The difference from pilot light is qualitative, not just quantitative. Warm standby's compute is *already running* (just undersized), so failover is "scale up," not "boot from scratch."
4. **Multi-Site Active/Active**: "your workload is deployed to, and actively serving traffic from, multiple... Regions [requiring] you to synchronize data across Regions." RTO/RPO is near zero, but this is exactly where `distributed-systems/consistency-models.md`'s CAP/PACELC tradeoffs stop being theoretical. Synchronizing writes across regions with real speed-of-light latency between them means choosing linearizable-and-slow or eventual-and-fast, not both. No active/active pattern avoids this choice.

| Strategy | RTO | RPO | Relative cost |
|---|---|---|---|
| Backup & Restore | Hours–days | Hours–days | Lowest |
| Pilot Light | Tens of minutes | Minutes (DB replicates continuously) | Low |
| Warm Standby | Minutes | Seconds–minutes | Medium |
| Multi-Site Active/Active | Near zero | Near zero | Highest |

**Multi-AZ (within one region) is a different, cheaper problem than multi-region DR.** AZs within a region are connected by high-bandwidth, low-latency private links (single-digit milliseconds). That is why synchronous replication (RDS Multi-AZ, a leader with a synchronously-replicated standby) is viable at the AZ level but rarely viable cross-region without accepting real latency cost on every write. "Multi-AZ" and "multi-region DR" get conflated in conversation, but they are different engineering problems with different achievable RTO/RPO floors.

---

## Static Stability: The Discipline Behind Surviving an AZ Failure

Source: AWS Builders' Library, *"Static stability using Availability Zones"* (Becky Weiss & Mike Furr) — [PDF](https://d1.awsstatic.com/builderslibrary/pdfs/static-stability-using-availability-zones.pdf), [aws.amazon.com/builders-library](https://aws.amazon.com/builders-library/static-stability-using-availability-zones).

The paper's core definition, quoted: "In a statically stable design, the overall system keeps working even when a dependency becomes impaired... everything it was doing before the dependency became impaired continues to work despite the impaired dependency." The opposite is **bimodal behavior**: a system that behaves one way normally and a *different* way during failure. Bimodal systems are dangerous because the failure-mode code path is the one exercised least often and trusted least, right when correctness matters most.

Why *reactive* scaling (auto-scale up the healthy AZs once you detect the bad one) is the wrong default, quoted directly: "the response to impairment requires actions from the control plane, which is typically more complex and more likely to misbehave when the overall system is impaired." The control plane you're relying on to save you is itself more likely to be degraded during the exact incident you need it to handle correctly. Reactive scaling bets your recovery on the thing least trustworthy at that moment.

The fix, quoted: "the key to static stability is to anticipate impairments before they happen... all capacity that will be needed in the event of an impairment is already fully provisioned and ready to go." Two concrete architectures the paper names:

```
Active-Active (stateless services):
  3+ AZs, each already over-provisioned to absorb the OTHER AZs' load
  AZ-B fails → traffic shifts to A and C → NO scaling event needed,
  A and C were already sized to handle this

Active-Standby (stateful services with a leader):
  AZ-A: [Leader]  →  synchronously replicates  →  AZ-B: [Standby]
  AZ-A fails → AZ-B promotes to leader → no data-plane scaling,
  just a leadership handoff (this is the exact same leader-election
  mechanism as distributed-systems/consensus-raft-vs-paxos.md,
  applied at the infrastructure-failover layer instead of the
  application-consensus layer)
```

The active-active pattern is "pay for idle over-provisioned capacity, always" in exchange for zero-action failover. The active-standby pattern trades some failover latency (promotion isn't instant) for not needing 3x redundant compute for stateless services that don't need a single leader anyway.

| | Active-Active | Active-Standby |
|---|---|---|
| Fits | Stateless services | Stateful services with a single leader |
| Idle capacity cost | High — every AZ over-provisioned to absorb the others' load | Lower — standby, not a full duplicate fleet |
| Failover action | None — traffic already flows to healthy AZs | Leader promotion (not instant) |
| Failure-mode risk | Lowest — no action required | Promotion path must itself be tested and trusted |

---

## Resilience Patterns: Timeouts, Retries, Backoff+Jitter

Source: AWS Builders' Library, *"Timeouts, retries, and backoff with jitter"* ([aws.amazon.com/builders-library](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter)) and Michael Nygard's *Release It!* (the origin of the named patterns Circuit Breaker/Bulkhead as a coherent taxonomy).

**Timeouts**: "set timeout on any remote call, and any call across processes on the same server, including both the connection timeout and request timeout." The gotcha, also from the source: "setting it too low can trigger a retry storm". A timeout that's too aggressive doesn't make the system faster. It makes it retry a request that would have succeeded, multiplying load on an already-struggling dependency.

**Retries**: not universally safe. "APIs with side effects aren't safe to retry unless they provide idempotency, which guarantees that the side effects happen only once no matter how often you retry." This is the same idempotency discipline covered concretely (with a SHA-256 key-derivation example) in the agentic-systems pitfalls material this session covered earlier. Retries without idempotent operations underneath them turn a transient network blip into a duplicated side effect (double charge, double insert).

**Backoff with jitter**: naive exponential backoff (every failed client waits `2^n` seconds, deterministically) causes every client that failed at the same moment to retry at the same moment again. That is a synchronized retry storm. The fix, quoted from the source's own recommendation: "inject jitter to the backoff... consider adding jitter to all timers, periodic jobs, and other delayed work to help spread out spikes of work."

```
Without jitter:                       With jitter:
Client A fails at t=0                 Client A fails at t=0
Client B fails at t=0                 Client B fails at t=0
Client C fails at t=0                 Client C fails at t=0

All retry at t=1s (2^0)               A retries at t=0.3s
All retry at t=2s (2^1)               B retries at t=0.9s
All retry at t=4s (2^2)               C retries at t=1.6s
   ↑ every retry wave is a            ↑ retries spread out, no
     synchronized load spike            synchronized spike on the
     hitting the recovering              recovering dependency
     dependency at once
```

**Managing load with a token bucket**: the source describes limiting retries locally via "a token bucket, which allows all calls to retry as long as there are tokens, and then retry at a fixed rate when the tokens are exhausted". It's a client-side circuit breaker of sorts, capping how much retry traffic any single client can generate regardless of how many logical requests are failing.

**Circuit Breaker and Bulkhead** (Nygard, *Release It!*): a circuit breaker tracks failure rate to a specific dependency. Past a threshold, it stops calling that dependency entirely for a cooldown period, failing fast instead of waiting out full timeouts on a dependency already known to be down. This is what prevents a slow dependency from exhausting the caller's own thread pool or connection pool. A bulkhead partitions resources (thread pools, connection pools) per-dependency so one overwhelmed downstream can't starve resources needed to serve requests to *other*, healthy dependencies. The naval-hull-compartment metaphor is literal: one compartment flooding shouldn't sink the whole ship.

| Pattern | What it does | Failure mode it prevents |
|---|---|---|
| Timeout | Bounds how long a caller waits on a remote call | Callers hanging indefinitely on a stuck dependency |
| Retry | Re-attempts a failed call | Transient (not persistent) failures causing a user-visible error |
| Backoff + jitter | Spaces out retry timing, randomized per client | Synchronized retry storms against a recovering dependency |
| Circuit breaker | Stops calling a dependency past a failure-rate threshold | Caller thread/connection pool exhaustion waiting on a known-down dependency |
| Bulkhead | Isolates resource pools per dependency | One overwhelmed dependency starving requests to unrelated, healthy dependencies |
| Token bucket | Caps retry rate per client once tokens run out | A single client generating unbounded retry load |

---

## Tail Latency: Why "Average Is Fine" Hides the Real Failure Mode at Scale

Source: Jeffrey Dean & Luiz André Barroso (Google), *"The Tail at Scale"*, Communications of the ACM, 2013 ([barroso.org PDF](https://www.barroso.org/publications/TheTailAtScale.pdf)).

The scale argument: if a single server has a 1-in-1000 chance of a slow response, and a single user request fans out to 100 servers (common for a search/aggregation-style backend), the probability that *at least one* of those 100 servers is slow is roughly `1 - (0.999)^100 ≈ 9.5%`. Nearly 1 in 10 user requests hits at least one slow component, even though each individual component is slow only 0.1% of the time. Averaging server-level latency completely hides this. Only the tail of the *end-to-end* request distribution shows it.

**Hedged requests**: "send a duplicate request to a different server if the first one doesn't respond within a specific time... wait until the first request has been outstanding for longer than the 95th percentile expected latency before sending a second request... significantly reduces tail latency while only adding a small amount (approx 5%) of extra load." The client races two replicas but only pays the tax of the second request in the rare case the first is slow.

**Tied requests**: a more aggressive, lower-overhead variant: "the client sends a request to two different replicas simultaneously, but attaches a tiny piece of metadata telling both replicas about each other. Once one replica actually starts processing the task, it sends a quick cancellation signal to the other, minimizing wasted compute effort." This targets a common source of tail latency the paper identifies: queueing delay *before* a request starts executing: "once a request is actually scheduled and begins execution, the variability of its completion time goes down substantially." Tied requests race on queueing delay, canceling the loser the instant either replica starts work, rather than waiting for a full response.

**Why this belongs in the same doc as multi-AZ/DR**: hedged/tied requests are the same "duplicate the work across independent failure domains, take whichever succeeds first" instinct as active-active multi-AZ, just applied at the single-request level instead of the whole-datacenter level. Both are static-stability-style thinking: pre-provisioned redundancy that requires zero reactive decision-making at the moment of failure, versus a control-plane action taken *after* detecting a problem.

---

## Cascading Failure: How a Local Problem Becomes a Global Outage

Synthesizing the mechanism common to Google's SRE book (*Site Reliability Engineering*, Ch. 22, "Addressing Cascading Failures," free at [sre.google/sre-book/addressing-cascading-failures](https://sre.google/sre-book/addressing-cascading-failures/)) and the resilience-pattern sources above:

```
1. One backend instance/AZ/dependency slows down or fails
        │
2. Callers' requests to it start timing out or queueing
        │  (if no circuit breaker: callers keep trying, exhausting
        │   their own thread/connection pools waiting on it)
        ▼
3. Callers themselves become slow (resource-starved, not because
   THEIR logic is broken, but because they're stuck waiting on #1)
        │
        ▼
4. Callers-of-callers see #3 as a new failure, and the same
   pattern repeats one layer up — the failure propagates
   OUTWARD from the original small problem
        │
        ▼
5. Load-balancer/orchestrator health checks may start killing
   and restarting the now-slow-but-not-broken instances,
   which REDUCES total capacity right when demand on the
   survivors spikes — this is the exact reactive-scaling trap
   the static-stability doc above warns against, occurring
   automatically via infrastructure automation instead of a human
```

The resilience patterns above are the interventions that stop this propagation at each step:

- **Timeouts** stop step 2 from hanging indefinitely.
- **Circuit breakers** stop step 3's resource exhaustion by failing fast instead of waiting.
- **Bulkheads** stop step 4 by ensuring one dependency's saturation can't consume resources needed for unrelated request paths.
- **Backoff with jitter** stops the recovery phase from re-triggering the same collapse via a synchronized retry storm.

---

## Comparative: Where the Distributed-Systems Primitives in This KB Show Up

- **Active-standby failover's leader promotion** uses the same mechanism as [`distributed-systems/consensus-raft-vs-paxos.md`](/systems-engineering/distributed-systems/consensus-raft-vs-paxos.md)'s leader election. A term/epoch increments, a new leader is chosen, and until that completes, writes are unavailable — the same "no leader, no progress" property covered there for etcd-style consensus.
- **Multi-site active/active's cross-region write synchronization** forces the same CAP/PACELC choice covered in [`distributed-systems/consistency-models.md`](/systems-engineering/distributed-systems/consistency-models.md). You cannot get RPO-zero cross-region writes without paying the latency cost of a synchronous quorum across regions. Most real active/active systems accept either higher write latency (linearizable) or a narrow window of possible data loss/conflict (eventual, with conflict resolution).
- **RDS Multi-AZ, Aurora Global Database, and DynamoDB Global Tables** are productized instances of the taxonomy above: synchronous single-region replication (viable because AZ-to-AZ latency is low), asynchronous cross-region replication with a defined replication lag (a real RPO, not zero), and multi-region active/active with last-writer-wins-style conflict resolution, respectively.
- **Cassandra's tunable consistency** ([`storage/cassandra.md`](/systems-engineering/storage/cassandra.md)) exposes this same RTO/RPO/consistency tradeoff as a per-request knob (`ONE`/`QUORUM`/`ALL`) instead of a fixed architectural choice — arguably a direct real-world instance of "the DR taxonomy above, but tunable per operation instead of fixed per deployment."

---

## Key Gotchas

- **Bimodal behavior is a root cause behind many failures, not just any single component failure.** Per the static-stability source, systems that behave differently under failure exercise their least-tested code path exactly when correctness matters most. Auditing "does my failure-mode behavior differ from normal-mode behavior" is a higher-leverage reliability review than auditing individual component MTBF.
- **Reactive auto-scaling as your only AZ-failure mitigation is a documented anti-pattern**, not a matter of taste. The control plane you're depending on to scale you out of trouble is statistically more likely to be degraded during the very incident you need it to work correctly for.
- **Retries without idempotency turn transient failures into permanent side effects.** A duplicated charge, insert, or message-send from a naive retry is usually worse for the business than the original transient failure would have been.
- **Untuned exponential backoff without jitter can make an outage worse, not better.** Synchronized retry waves from every client hitting a recovering dependency at the same instant are a self-inflicted second incident.
- **Multi-region "active/active" marketed as zero-RPO is often not testing the CAP tradeoff it's making.** Confirm whether writes are synchronously quorum-committed across regions (real zero RPO, real latency cost on every write) or eventually consistent with conflict resolution (RPO ≈ 0 only in the *typical* case, real RPO > 0 during an actual partition) before trusting a vendor's "active/active" label at face value.
- **Tail-latency mitigations (hedging/tied requests) have a load cost that must be budgeted, not assumed free.** The Tail at Scale paper's own ~5% overhead figure for hedged requests is the acceptable case because it's tuned to trigger only past the 95th percentile expected latency. Hedging too aggressively (triggering on a low threshold) can itself create the load spike it was meant to avoid.

---

*Every claim about the four-DR-strategy taxonomy, static stability, timeouts/retries/backoff, and hedged/tied requests above is a direct paraphrase or quote of the named primary sources (AWS Well-Architected DR whitepaper, AWS Builders' Library, Dean & Barroso's "The Tail at Scale," Nygard's Release It!, Google's SRE book). Sources are cited inline per section rather than only in a closing note, since this doc's value depends on being traceable back to authoritative, quotable sources rather than restated secondhand. Read the primary sources directly for full context — this doc is a structured index into them, not a replacement for reading them.*
