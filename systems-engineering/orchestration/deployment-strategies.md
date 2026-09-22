# Deployment Strategies

## 30-Second Intuition

A deployment strategy answers one question: how does the running version of a workload become the new version, and what happens to traffic while that transition is in progress. Kubernetes ships one strategy natively (Rolling Update, with Recreate as the other built-in option) and stops there. The moment a team needs weighted traffic splitting between two live versions — canary, A/B, or a metrics-gated rollout — plain Kubernetes has no primitive for it. That gap is exactly why Argo Rollouts, Flagger, and the Gateway API's weighted backends exist. This doc draws that line precisely: what a base Kubernetes `Deployment` can do on its own, and where each pattern needs something else.

---

## Recreate: The Silent Default Trap

**What it is**: terminate every old pod, then start every new pod. Full downtime between the two steps.

```
Time →
Old pods:  [v1][v1][v1]  ✕✕✕ (all killed)
New pods:                     [v2][v2][v2] (all started, after old ones are gone)

User-facing downtime: from "all old pods dead" to "all new pods ready"
```

**When it's chosen deliberately**: a workload where two versions cannot coexist at all — a schema migration that the old code cannot tolerate, or a singleton process that must never have two instances running.

**The gotcha**: Recreate is the default only if `spec.strategy.type` is unset in older Deployment manifests written against outdated examples, or if a team copies a manifest from a source that hardcoded it. Current Kubernetes defaults a `Deployment` to `RollingUpdate` when the strategy field is omitted. Teams that hit unexpected downtime on a routine deploy should check for an explicit `type: Recreate` line before assuming rolling update was silently skipped.

---

## Rolling Update: Kubernetes' Actual Default

**What it is**: replace old pods with new ones incrementally, keeping the service available throughout. This is the default `Deployment` strategy in current Kubernetes.

Two fields control the pace:

| Field | Meaning | Default |
|---|---|---|
| `maxUnavailable` | How many pods below the desired replica count the rollout is allowed to drop to | 25% |
| `maxSurge` | How many pods above the desired replica count the rollout is allowed to create | 25% or 1 |

**Worked example — 10 replicas, `maxSurge: 2`, `maxUnavailable: 2`:**

```
Start:        [v1 x10]                                    total=10, available=10

Step 1:       [v1 x10][v2 x2]                              total=12 (surge used: +2)
Step 2:       [v1 x8 ][v2 x2]  ✕2 old killed                total=10, available=10
Step 3:       [v1 x8 ][v2 x4]                              total=12
Step 4:       [v1 x6 ][v2 x4]  ✕2 old killed                total=10, available=10
...repeats until:
Final:        [v2 x10]                                     total=10, available=10
```

At no point does available capacity drop below 8 (10 - `maxUnavailable`), and total pod count never exceeds 12 (10 + `maxSurge`). Kubernetes controls the pace of the rollout through these two numbers alone — it does not control what fraction of *live traffic* reaches v1 versus v2 at any given step. Traffic split during a rolling update is simply the pod-count ratio, because a `Service` load-balances evenly across all pods matching its selector, old and new versions included.

**Gotcha**: `maxUnavailable: 0` requires `maxSurge >= 1`. With both effectively zero, the rollout has no room to create new pods or remove old ones and stalls.

---

## Blue-Green: Two Full Environments, One Switch

**What it is**: run the new version (green) fully scaled up, alongside the old version (blue), then cut traffic over in one move.

```
Before cutover:              After cutover:
Service selector → blue      Service selector → green
[blue x10] (serving)         [blue x10] (idle, kept warm for rollback)
[green x10] (idle, warmed)   [green x10] (serving)
```

On Kubernetes, the cutover is a `Service`'s `selector` field changing from matching the blue Deployment's labels to matching green's — traffic moves for every pod at once, instantly, because the `Service` is just relabeling which pods it forwards to.

| | Value |
|---|---|
| Downtime | None, if both versions are healthy at cutover |
| Rollback speed | Instant — flip the selector back |
| Resource cost | 2x during the overlap window |
| Traffic control granularity | All-or-nothing |

**Gotcha — the database-migration problem**: blue and green both run against the same database during the overlap window. If the new version's schema migration is not backward-compatible, the old version breaks the moment the migration runs, before cutover even happens. Blue-green only works cleanly when both versions can operate against the same schema simultaneously — expand/contract migration patterns exist specifically to make this true.

---

## Canary: Where Kubernetes Stops and Argo Rollouts Starts

**What it is**: route a small percentage of traffic to the new version first, watch for problems, then increase the percentage gradually.

**What a base Kubernetes `Deployment` can approximate**: scale two separate Deployments to a pod-count ratio and point one `Service` at both, via matching labels. 1 new pod out of 10 total gives roughly 10% of traffic to the new version, because the `Service` load-balances per-pod, not per-percentage.

**What a base Kubernetes `Deployment` cannot do**:

| Capability | Native `Deployment` + `Service` | Needs |
|---|---|---|
| Exact traffic percentage, independent of pod count | No — only pod-ratio approximation | Argo Rollouts, Flagger, or Gateway API weighted `HTTPRoute` |
| Automated metrics-based promotion/rollback | No | Argo Rollouts `AnalysisTemplate`, or Flagger's built-in metric checks |
| Step-by-step manual gating | No | Argo Rollouts' explicit `steps` list |
| Header/cookie-based routing to canary | No | Service mesh or Gateway API `HTTPRoute` match rules |

This is the exact gap [`orchestration/argo-rollouts.md`](/systems-engineering/orchestration/argo-rollouts.md) fills — that doc covers the CRD mechanics (`AnalysisTemplate`, Istio weight patching, the `Experiment` CRD) in depth. This doc is the pattern-level statement of *why* that tool needed to exist: Kubernetes' pod-ratio approximation is not real traffic-percentage control, and it carries no metrics signal at all.

**2026 landscape**: Argo Rollouts and Flagger are the two dominant progressive-delivery controllers, both mature and actively maintained (Flagger v1.44.0 and Argo Rollouts v1.9.1 shipped three days apart in July 2026). They differ in ownership model:

| | Argo Rollouts | Flagger |
|---|---|---|
| Workload model | Replaces `Deployment` with a `Rollout` CRD | Leaves the `Deployment` untouched, adds a `Canary` resource alongside it |
| Control style | Explicit, step-defined | Metrics-driven, largely automatic |
| Common pairing | ArgoCD | Flux |

**A newer, lighter-weight option**: the Gateway API's `HTTPRoute` supports weighted backend references natively — multiple backends in one routing rule, each with a weight, and the weights are the split ratio's denominator across all of them. This gives exact traffic-percentage control without adopting a full progressive-delivery controller, though it still has no built-in metrics-based promotion — a team gets traffic-splitting for free but still needs to wire up their own promotion/rollback logic on top.

---

## Tying It to AWS Networking: Where the Same Gap Shows Up Outside K8s

The Kubernetes-versus-Argo-Rollouts gap above has a direct AWS-networking equivalent, one layer below Kubernetes entirely. See [`cloud-aws/aws-networking.md`](/systems-engineering/cloud-aws/aws-networking.md) for the VPC/subnet/routing layer this section builds on — this section is about L7 traffic control, one layer above that.

**Two AWS primitives do weighted traffic splitting, at two different layers:**

| Primitive | Layer | Splits traffic between | Granularity |
|---|---|---|---|
| ALB weighted target groups | L7 (HTTP), within one load balancer | Two target groups behind one ALB listener rule | Exact percentage, no DNS caching lag |
| Route 53 weighted routing | DNS | Two distinct DNS-addressable endpoints (e.g. two separate ALBs, or two regions) | Approximate percentage — client/resolver DNS caching means the actual split lags the configured weights |

The distinction matters operationally. An ALB's weighted target groups reassign traffic on the next request, because the ALB itself makes the routing decision. A Route 53 weighted record changes which IP a *resolver* hands back, and that resolver — or the client's OS, or an intermediate caching layer — may hold onto the old answer for the record's TTL. Route 53 weighted routing is the right tool for splitting traffic between two **separate, independently-addressable** stacks (two full ALBs, two regions); ALB weighted target groups are the right tool for splitting traffic between two versions **behind the same load balancer**.

**Blue-green and canary have first-class AWS support outside Kubernetes entirely**, via AWS CodeDeploy:

```
ECS blue/green via CodeDeploy:
  ALB listener ──► Target Group A (blue, live)
                    Target Group B (green, idle)

CodeDeploy shifts the LISTENER RULE'S weight from A to B,
using a predefined schedule, e.g.:
  CodeDeployDefault.ECSLinear10PercentEvery1Minutes
    → 10% to green, wait 1 min, 20%, wait 1 min, ... 100%
```

- **Lambda**: CodeDeploy natively supports canary traffic shifting for Lambda aliases — a percentage of invocations route to the new version, increasing on a schedule, with automatic rollback on a CloudWatch alarm.
- **ECS**: as of October 2025, ECS supports canary and linear deployment strategies natively, without CodeDeploy as a separate orchestrator — this is now the recommended default for new ECS deployments, though CodeDeploy-managed blue/green remains supported for existing pipelines.
- **EKS**: none of this native ECS/CodeDeploy tooling applies. An EKS workload is back to the exact Kubernetes-layer gap described above — plain `Deployment`/`Service` gives pod-ratio approximation only, and Argo Rollouts/Flagger/Gateway API are what actually closes it.

**The practical decision this creates**: a team running ECS gets canary and blue-green essentially for free from AWS-native tooling. A team running EKS does not — the same capability requires deploying and operating Argo Rollouts or Flagger themselves. This is a real, concrete cost difference between the two compute choices that rarely shows up in a compute-cost comparison, because it is an operational-tooling cost, not a compute-cost line item.

---

## A/B Testing: Same Mechanism, Different Goal

**What it is**: split traffic between two versions using the identical mechanism as canary — weighted routing, header/cookie-based targeting — but the goal is measuring a difference in user behavior, not verifying safety before a full rollout.

The distinction is intent, not implementation:

- **Canary** asks: is the new version safe to fully roll out?
- **A/B test** asks: does the new version perform better on a specific metric (conversion, engagement)?

Both commonly use the same traffic-splitting primitives (Gateway API weights, service mesh routing, Argo Rollouts steps). An A/B test typically needs to run longer and needs sticky routing (the same user consistently sees the same version) — a requirement canary usually does not have.

---

## Shadow / Dark Launch: Real Traffic, Zero User-Facing Risk

**What it is**: mirror a copy of real production traffic to the new version, but never return its response to the real user. The old version's response is what the user actually sees; the new version's response is discarded or logged for comparison.

```
Real request ──┬──► v1 (serves response to user)
               └──► v2 (mirrored copy, response discarded/logged)
```

This tests the new version under real production load and real request patterns with no risk of a bad response reaching a user. It does not test user-facing behavior differences the way A/B testing does — shadow traffic never reaches a real user's screen. Kubernetes has no native traffic-mirroring primitive; this needs a service mesh's mirroring feature (Istio's `mirror` field, Linkerd's traffic-split with a mirror target) or an API-gateway-level mirroring configuration.

---

## Feature Flags: Decoupling Deployment from Release

**What it is**: ship the new code to production, inactive, gated behind a runtime flag. The flag — not the deployment mechanism — decides who sees the new behavior and when.

This is a different axis from every strategy above, not a competing one:

| Deployment strategy | Feature flag combined with it |
|---|---|
| Rolling update | Code for both old and new behavior ships together; the flag decides which path runs, independent of which pods are running which pod image |
| Canary | Canary controls which pods receive traffic; the flag can additionally control which *users* see the new behavior, even within the canary pods |
| Blue-green | Green can go live with the flag still off, verify health, then flip the flag separately from the traffic cutover |

The practical value: a bad rollback becomes flipping a flag (seconds) instead of rolling back a deployment (a new rollout, minutes). The cost: code paths for both old and new behavior must coexist in the same binary until the flag is fully retired, which is its own cleanup discipline teams frequently skip.

---

## Comparative Summary

| Strategy | Downtime | Rollback speed | Resource cost | Traffic control granularity | Native K8s support |
|---|---|---|---|---|---|
| Recreate | Full, during cutover | Redeploy old version | 1x | None (all-or-nothing) | Full — built in |
| Rolling Update | None | Redeploy old version (slow) | ~1.25x during rollout | Pod-ratio only | Full — built in, default |
| Blue-Green | None | Instant (flip selector) | 2x during overlap | All-or-nothing | Partial — `Service` selector swap only, no automation |
| Canary | None | Scale canary to zero | ~1x-1.25x | Exact %, with a controller; pod-ratio without one | None natively — needs Argo Rollouts/Flagger/Gateway API |
| A/B Testing | None | Scale variant to zero | ~1x-1.25x | Exact %, with a controller | None natively — same gap as canary |
| Shadow/Dark Launch | None | Stop mirroring | ~2x (both versions process every request) | N/A (mirrored, not split) | None — needs a service mesh |
| Feature Flags | None | Flip flag | No extra compute | Per-user/per-request, independent of deployment | None — an application-layer concern, not a K8s primitive |

---

## Key Gotchas

- **Recreate can appear from an unexpected place.** A manifest copied from an older example or template with `type: Recreate` hardcoded will silently reintroduce full downtime on every deploy — check the strategy field explicitly rather than assuming rolling update is in effect.
- **`maxUnavailable: 0` without `maxSurge >= 1` stalls a rollout.** With no room to surge and no room to go unavailable, the Deployment controller has nowhere to place new pods and the rollout hangs.
- **Blue-green's shared database is the real constraint, not the traffic switch.** The cutover itself is instant and safe; the risk is entirely in whether both versions can run against the same schema during the overlap window.
- **Pod-ratio canary is not the same signal as a real canary analysis.** Scaling 1 new pod alongside 9 old ones is not "10% of traffic is being safety-checked" — it is 10% of traffic with no metrics evaluation attached, unless an `AnalysisTemplate` (Argo Rollouts) or metric check (Flagger) is actually wired in. Traffic going somewhere is not the same as traffic being watched.
- **Shadow traffic doubles compute cost for the mirrored path**, since both versions process every mirrored request even though only one response reaches the user — budget for this before turning on mirroring at full production volume.
- **Feature flags left in code after their rollout is done accumulate as technical debt.** A flag that decoupled a risky release six months ago and was never removed is now permanent branching complexity with no remaining purpose.
- **Route 53 weighted routing lags its configured weights because of DNS caching, ALB weighted target groups do not.** A canary using Route 53 weights can see a much slower or lumpier actual traffic ramp than the configured percentages suggest, because resolvers and clients hold onto answers for the record's TTL — this is invisible until someone checks actual request logs against the configured weight and finds them mismatched.
- **EKS gets none of ECS's native canary/blue-green tooling.** A team moving a workload from ECS to EKS (or comparing the two) needs to budget for standing up Argo Rollouts or Flagger themselves — this is a real operational cost that a pure compute-cost comparison between the two will not surface.

---

*Grounded against kubernetes.io's Deployment documentation, the Gateway API's HTTPRoute traffic-splitting guide, AWS's own CodeDeploy/ECS/Route 53/ALB documentation, and 2026 community comparisons of Flagger and Argo Rollouts (both confirmed shipping actively maintained releases — Flagger v1.44.0 and Argo Rollouts v1.9.1 — three days apart in July 2026), as of September 2026. ECS-native canary/linear deployment support (October 2025) and Route 53-vs-ALB weighted-routing mechanics are both confirmed against current AWS documentation. Gateway API's weighted-backend traffic splitting is documented and in active use in 2026 sources; its formal GA/stable designation was not independently confirmed against the Gateway API project's own release notes and should be re-checked before citing a specific stability level externally.*
