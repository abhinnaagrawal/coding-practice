# Data Platform Architecture Patterns

## 30-Second Intuition

These patterns answer two different questions that get conflated constantly. One is **how does data physically flow and get refined** (Medallion, Lambda, Kappa — pipeline shape). The other is **who owns and governs the data, and how do consumers find it** (Data Mesh, Data Fabric — organizational/access model). You can run Medallion refinement *inside* a Data Mesh domain. You can run Kappa streaming *underneath* a Data Fabric's virtualization layer. These aren't mutually exclusive tiers — they're answers to different axes of the same overall design. The one fact that matters operationally, grounded against 2026 industry data: **the pure form of almost every one of these patterns loses to a hybrid**. Pure Data Mesh succeeds in only ~38% of implementations vs. 52% for hybrid mesh/centralized approaches (McKinsey, Oct 2025 survey). The 2026 consensus on Lambda vs. Kappa is that neither pure form is optimal anymore — most production systems borrow from both.

---

## Medallion Architecture (Bronze/Silver/Gold) — Your Baseline

```
Raw source (Kafka topic, CDC stream, batch file) 
      │
      ▼
BRONZE   — raw, unmodified, append-only ingestion (schema-on-read, keep everything,
            including bad/malformed records — this layer is the "we can always
            re-derive everything downstream from here" insurance policy)
      │
      ▼
SILVER   — cleaned, deduplicated, conformed to a schema, joined/enriched against
            reference data — still row-grain, not yet business-aggregate shaped
      │
      ▼
GOLD     — business-level aggregates, star-schema fact/dimension tables
            (see data-modeling/star-schema-dimensional-modeling.md), ready for
            BI tools/dashboards/ML feature consumption
```

This is a **progressive-refinement pipeline shape**, not a technology. It's typically implemented as:

- Table format: Iceberg or Delta Lake — [`data-formats/apache-iceberg.md`](/systems-engineering/data-formats/apache-iceberg.md), [`data-formats/delta-lake.md`](/systems-engineering/data-formats/delta-lake.md)
- Storage: object storage underneath the table format — [`storage/s3.md`](/systems-engineering/storage/s3.md)
- Compute: Spark or a similar engine running the Bronze→Silver→Gold transformation jobs — [`compute/spark.md`](/systems-engineering/compute/spark.md)

**The tradeoff you're already living with**: Medallion's core bet is that storage is cheap enough to keep three (or more) full copies of increasingly-refined data. In exchange, you get two things. First, reprocessing resilience — a bug in a Silver transform can be fixed and Silver/Gold rebuilt from Bronze without re-ingesting from the source. Second, clear operational boundaries — a Gold-table consumer never has to reason about raw source quirks, and a Silver-layer engineer never has to reason about business-aggregate logic. The cost: every layer transition is a real batch/micro-batch job with real latency. Bronze's "keep everything, including garbage" policy also means storage and later reprocessing cost scale with total raw volume, not useful volume.

---

## Lambda Architecture — Parallel Batch + Speed Layers

```
                    ┌──────────────────┐
Raw data source ──▶ │   BATCH LAYER    │──▶ Batch views (complete, slow,
                    │  (Spark, hours)  │     eventually correct — the
                    └──────────────────┘     authoritative long-term answer)
       │
       │            ┌──────────────────┐
       └──────────▶ │   SPEED LAYER    │──▶ Real-time views (approximate,
                    │ (Flink, seconds) │     fast — bridges the gap until
                    └──────────────────┘     the batch layer catches up)
                              │
                              ▼
                    Query merges both views: recent (speed layer) +
                    historical (batch layer), speed layer's window
                    discarded once batch layer covers the same period
```

**The core idea**: get a fast-but-approximate answer immediately (speed layer), and a slow-but-exact answer eventually (batch layer). Merge them at query time so users always see *something* current while the authoritative number catches up.

- Classic batch-layer engine: Spark — [`compute/spark.md`](/systems-engineering/compute/spark.md)
- Classic speed-layer engine: Flink, via its true streaming model — [`compute/flink.md`](/systems-engineering/compute/flink.md)

**The problem that made this fall out of favor**: the same business logic ("compute revenue per user per day") has to be implemented *twice* — once in the batch engine's API, once in the streaming engine's API. These two implementations drift out of sync over time as one gets a bug fix the other doesn't. This dual-codepath maintenance burden is the most-cited reason teams move away from pure Lambda.

---

## Kappa Architecture — Streaming-Only, Replay for Reprocessing

```
Raw data source ──▶ Single durable log (Kafka, long/infinite retention)
                              │
                              ▼
                    ONE streaming pipeline (Flink/similar)
                    processes it continuously
                              │
                              ▼
                    Query-ready views (materialized incrementally)

Reprocessing (e.g. a bug fix to the transformation logic):
  spin up a SECOND instance of the same pipeline, replaying the log
  from an earlier offset, running the NEW logic → swap views over
  once caught up → tear down the old pipeline instance
  (one codebase, no batch/speed duplication — reprocessing IS just
  "replay the same pipeline again from an earlier point")
```

**The core idea**: eliminate Lambda's dual-codepath problem by treating "reprocessing historical data" as nothing more than "replay the stream from an earlier offset through the same streaming pipeline." There's only ever one processing codebase. Two things make this work:

- The log must retain enough history to replay from. Kafka's tiered storage is what makes long-retention replay economically viable at scale — [`streaming/kafka.md`](/systems-engineering/streaming/kafka.md)
- The streaming engine must support true reprocessing semantics. Flink's checkpoint/savepoint model is the concrete mechanism — [`compute/flink.md`](/systems-engineering/compute/flink.md)

**2026 status, grounded**: neither pure Lambda nor pure Kappa is the current default recommendation. The 2026 consensus (RisingWave, AutoMQ, and others) is a hybrid **"streaming database/engine for real-time views + a lakehouse (Iceberg/Delta) for historical analytics"** pattern, sometimes labeled "Delta Architecture" in vendor material (not to be confused with Delta Lake the table format). This is functionally Medallion's Gold layer being fed by a Kappa-style streaming pipeline instead of a Lambda-style dual pipeline. One infrastructure shift is making pure Kappa more viable than it used to be: "diskless Kafka" (brokers as stateless compute writing directly to object storage, rather than local disk) changes the cost/retention math enough that keeping years of replayable history in the log itself is now economically reasonable in a way it wasn't a few years ago.

---

## Worked Example: One Click Event, Three Ways

Same input — a user click event — traced through all three pipeline shapes, showing where the data lives and how many processing paths touch it:

```
MEDALLION:
  click event → Bronze (raw JSON, Kafka→Iceberg, append-only)
              → Silver (Spark batch job: dedup, validate schema,
                enrich with session_id from a lookup table)
              → Gold (Spark batch job: SUM(clicks) per user per day,
                stored as a Gold fact table)
  Latency: Gold table is fresh as of the last batch job run
           (typically minutes to hours depending on schedule)
  Physical copies: 3 (Bronze, Silver, Gold), same event, 3 places

LAMBDA:
  click event → Kafka topic
              → BATCH: Spark job (hourly), computes exact
                SUM(clicks) per user per day → batch view table
              → SPEED: Flink job (continuous), computes approximate
                running SUM(clicks) for the CURRENT day only →
                real-time view table
  Query time: SUM(clicks) = batch view (all days except today)
              + speed view (today only, approximate until the next
              batch run supersedes it)
  Physical copies: raw log + 2 DIFFERENT aggregate implementations
                   of the same logic, in 2 different engines

KAPPA:
  click event → Kafka topic (long retention, e.g. tiered storage)
              → ONE Flink job, continuously, computes
                SUM(clicks) per user per day incrementally
              → materialized view table (always current)
  Reprocessing a bug fix: replay the SAME Flink job's logic from
  an earlier Kafka offset in a second instance, swap the output
  view over once it catches up to real-time
  Physical copies: raw log + 1 continuously-updated view
                   (no separate batch/speed implementations)
```

The Kappa version has the least duplicated logic (one implementation). It requires the log to hold enough history to replay from, and a streaming engine sophisticated enough to make "replay = reprocess" work correctly — exactly-once semantics under replay, not just under normal operation.

| | Medallion | Lambda | Kappa |
|---|---|---|---|
| Processing codebases | 1 (sequential Bronze→Silver→Gold) | 2 (batch + speed, separately implemented) | 1 (single streaming pipeline) |
| Freshness | Minutes-to-hours (batch schedule) | Seconds (speed layer), exact after batch catches up | Seconds, continuously |
| Reprocessing mechanism | Rebuild Silver/Gold from Bronze | Re-run the batch layer | Replay the log from an earlier offset |
| Main risk | Bronze storage/reprocessing cost growth | Batch/speed logic drift | Exactly-once-under-replay correctness |

---

## Data Mesh — Decentralized, Domain-Oriented Ownership

```
                    ┌─────────────────────────────────┐
                    │   Federated Computational        │
                    │   Governance (central IT sets     │
                    │   hard boundaries: security,      │
                    │   compliance, interoperability     │
                    │   standards)                       │
                    └────────────┬──────────────────────┘
                                  │
        ┌─────────────────┬──────┴───────┬─────────────────┐
        ▼                 ▼              ▼                 ▼
   Domain: Orders   Domain: Users   Domain: Payments   Domain: Catalog
   (owns its OWN    (owns its OWN   (owns its OWN      (owns its OWN
    data product,    data product,   data product,      data product,
    pipeline, SLA)   pipeline, SLA)  pipeline, SLA)      pipeline, SLA)
        │                 │              │                 │
        └────── each domain publishes a "data product" with a ──────┘
                 defined contract/schema/SLA that OTHER domains
                 consume directly — no central team in the
                 critical path of every cross-domain data need
```

**The core idea**: Data Mesh is an *organizational* change (who owns what, who's accountable for data quality) expressed through architecture, not a technology choice. This is the distinction to hold onto: most Data Mesh failures are attempts to buy the org-structure benefit with pure tooling.

**2026 grounded reality, and it's a real cautionary tale**: only about 18% of organizations have the governance maturity to execute Data Mesh well. Fully decentralized governance (no central standards at all) has documented failure modes — fragmented security policies, duplicated effort. Measured impact in failed rollouts: 40-60% drops in governance compliance and 50-100% increases in time-to-deploy new projects. The 2026 winning model, per Thoughtworks and corroborating sources, is **federated computational governance**: central IT sets hard, non-negotiable boundaries (security, compliance, core interoperability standards) while domains retain real autonomy over everything else. This is not the "fully decentralized, no central authority" version some early 2020s Data Mesh advocacy implied. McKinsey's October 2025 survey found hybrid mesh/centralized approaches succeeding at 52% vs. 38% for pure-mesh attempts. Organizations that spent 12 months honestly assessing their own governance maturity before choosing an approach outperformed fast-committers by 25-40%.

---

## Data Fabric — Metadata-Driven Virtualization, Not an Org Change

```
   Source A          Source B          Source C          Source D
  (Postgres)          (S3/Iceberg)      (Salesforce)      (Kafka)
      │                   │                  │                 │
      └───────────────────┴──────────────────┴─────────────────┘
                            │
                  ┌─────────▼──────────┐
                  │   ACTIVE METADATA   │   catalogs, lineage,
                  │   + VIRTUALIZATION  │   quality signals, and a
                  │        LAYER        │   query-federation layer
                  └─────────┬──────────┘   that lets a consumer query
                            │               across sources WITHOUT
                            ▼               copying data into one
                  Unified, governed          place first
                  access surface for
                  consumers/agents
```

**The core distinction from Data Mesh**: Data Fabric is a *technology* pattern — a metadata-driven integration/virtualization layer you can lay over *existing* infrastructure without restructuring teams. Data Mesh is an *organizational* pattern of restructuring ownership around domains. Data virtualization (query multiple sources as if they were one) is a component *inside* a Data Fabric, not the whole thing. Fabric additionally requires active metadata, lineage tracking, and governance automation on top of raw virtualization.

**2026 reality**: most enterprises now run both together rather than choosing one. Data Fabric supplies the connective metadata/governance layer. Data Mesh supplies domain accountability for the data products flowing through it. Adoption of any single one of lakehouse/fabric/mesh as a standalone label is modest — survey data puts each around 8-12% standalone usage. That itself signals that "pick exactly one named architecture" is the wrong framing for most real platforms.

---

## Underneath All of These: Data Warehouse vs. Data Lake vs. Lakehouse

Brief, since this is the storage-architecture question every pattern above sits on top of, not a competing pipeline pattern:

| | Data Warehouse | Data Lake | Lakehouse |
|---|---|---|---|
| Storage | Proprietary, structured, schema-on-write | Object storage, any format, schema-on-read | Object storage + open table format (Iceberg/Delta) |
| Transactions | Strong (ACID, mature) | Weak/none natively | ACID via the table format — see `data-formats/apache-iceberg.md`/`delta-lake.md` |
| Query engine coupling | Tightly coupled to the warehouse's own engine | Any engine can read the files | Any engine can read AND write, transactionally |
| Cost profile | Higher storage cost, optimized for structured BI | Cheapest storage, no built-in query performance | Cheap storage + warehouse-grade transactional guarantees |

Medallion architecture is almost always implemented as a Lakehouse today *because* the table-format layer (Iceberg/Delta) is what makes Bronze/Silver/Gold's "rebuild downstream layers from upstream" promise safe. Without ACID guarantees at the table level, a failed Silver rebuild job can leave Silver in a half-written, inconsistent state.

---

## When to Use What

| Signal | Lean toward |
|---|---|
| Small-to-mid team, need reprocessing safety, latency in minutes-to-hours is fine | **Medallion on a lakehouse** — your current setup; formalize the Bronze retention policy and Silver/Gold rebuild runbooks rather than switching patterns |
| Need both an exact historical number AND a fast approximate current number, and can tolerate maintaining two codepaths | **Lambda** — increasingly rare as the sole justification, since modern streaming engines close much of the latency gap Lambda existed to solve |
| Need low end-to-end latency, want one processing codebase, and can afford long-retention replayable log storage | **Kappa** (or the 2026 hybrid: streaming engine for real-time views + lakehouse for historical, which is functionally Kappa feeding Medallion's Gold layer) |
| Multiple independent business domains, each with real data-engineering capability, and executive buy-in for a multi-year governance investment | **Data Mesh with federated computational governance** — not pure decentralization. Budget 12+ months of honest maturity assessment before committing, per the McKinsey findings above |
| Fragmented sources across many systems (some legacy, not team-restructurable), need unified governed access without an org change | **Data Fabric** layered over what you already have |
| Any of the above, at real enterprise scale, in 2026 | **A hybrid of at least two of these**, not a pure form — this is now the empirically documented default outcome, not a failure to commit |

---

## Key Gotchas

- **Medallion's Bronze "keep everything" policy needs an explicit retention/reprocessing runbook, not just a good intention.** Without one, Bronze becomes an unbounded-growth cost center. "Just rebuild Silver from Bronze" becomes a multi-day job nobody has rehearsed before the day they need it.
- **Lambda's dual-codepath drift is a maintenance tax that compounds, not a one-time cost.** Every business-logic change must be ported to both the batch and speed layer implementations. The two will disagree eventually if this discipline lapses even once.
- **Kappa's "reprocessing is just replay" promise is only as good as your streaming engine's exactly-once-under-replay guarantees.** A naive streaming pipeline replayed from an old offset can double-count or miss data if checkpoint/idempotency semantics weren't designed for this from the start. See [`compute/flink.md`](/systems-engineering/compute/flink.md) for the checkpoint-barrier mechanics that "correct" replay requires.
- **Pure Data Mesh without federated (not fully decentralized) governance has a documented, specific failure signature**: fragmented security policies, duplicated engineering effort across domains, and measurable compliance/deploy-time regressions. This isn't a hypothetical risk — it's the most common failure mode as of 2026.
- **Data Fabric's virtualization layer can hide, not fix, underlying data quality problems.** Querying garbage data across 4 sources through one clean-looking unified interface is still querying garbage data. A Fabric layer's governance/lineage tooling only helps if someone acts on what it surfaces.
- **"We should migrate off Medallion to Data Mesh/Kappa" is usually the wrong question.** These solve different problems — pipeline shape vs. ownership model. The 2026 data suggests the right move for most teams is layering a complementary pattern on top of what exists, not a wholesale architectural rewrite.

---

*Grounded against Thoughtworks' "State of Data Mesh 2026," McKinsey's October 2025 hybrid-vs-pure-mesh survey, Gartner Data Mesh 2026 Hype Cycle coverage (via Atlan), and 2026 Lambda/Kappa comparison writeups (RisingWave, AutoMQ, Flexera) as of September 2026. The governance-maturity (~18%), hybrid-success-rate (52% vs 38%), and adoption-percentage (8-12% each) figures are all survey-sourced and should be treated as directional industry signal, not precise universal constants — re-verify against a named, current survey before citing externally.*
