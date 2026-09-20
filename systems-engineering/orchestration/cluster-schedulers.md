# Cluster Resource Schedulers: Kubernetes, YARN, and What Happened to Mesos

## 30-Second Intuition

A cluster scheduler answers one question, over and over, at scale: given a pod/container/task that needs resources, which node should it run on? Kubernetes's `kube-scheduler` answers this per-pod, independently, via a two-phase filter-then-score pipeline — no pod knows or cares about any other pod's placement decision unless something explicitly tells the scheduler otherwise. The one fact that matters operationally, and the one that directly explains a real gotcha already flagged in [`compute/spark-on-eks.md`](../compute/spark-on-eks.md): **kube-scheduler has no native concept of "schedule all of these together, or none at all."** YARN's ApplicationMaster model gets this almost for free (an AM negotiates all the containers a job needs before the job starts); Kubernetes had to bolt gang-scheduling on afterward, as a separate ecosystem of tools (Kueue, Volcano, coscheduling), because the core scheduler was never designed around job-level, all-or-nothing placement in the first place.

---

## Resource-Layer Map

| Layer | Role kube-scheduler plays | What it's optimizing for |
|---|---|---|
| CPU/Memory | The primary bin-packing dimensions — every pod declares resource `requests` (and optionally `limits`); the scheduler's Filter phase eliminates nodes that can't satisfy requested CPU/memory, Score phase ranks survivors | Fit pods onto nodes efficiently using declared requests as the unit of account — **not** actual live usage, a distinction that causes real production confusion (see gotchas) |
| Network | Pod affinity/anti-affinity and topology spread constraints reason about network locality/blast-radius (e.g. "keep these pods on the same node," "spread these pods across zones") | Minimize cross-node/cross-zone chatter for latency-sensitive pairs, or maximize failure-domain spread for availability — two opposite goals, both expressed through the same affinity/topology machinery |
| Disk | Storage-class and local-PV awareness — a pod requiring a specific local volume can only schedule onto a node that actually has it | Correctness (a pod can't run where its required storage doesn't physically exist), not performance optimization per se |
| GPU | Not native to the core scheduler at all — requires the **device plugin** framework, a separate extension mechanism for advertising and allocating non-CPU/memory "extended resources" | Contrast directly with [`storage/hadoop.md`](../storage/hadoop.md)'s note that "YARN can schedule GPU-aware containers... but GPU scheduling is a bolt-on capability" — Kubernetes made the identical choice, for the identical reason: neither scheduler's core resource model was originally built around anything but CPU/memory as first-class dimensions |

---

## The Signature Mechanism: Filter, Then Score

Since Kubernetes 1.19, the scheduler has been a genuine plugin engine — the **Scheduling Framework** — with each pipeline stage (`PreEnqueue → QueueSort → PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind`) exposed as an extension point plugins can hook into. The two stages that matter most for understanding placement decisions:

**Filter** (historically called "predicates"): for a given pod, evaluate every node and eliminate any that can't possibly run it — insufficient CPU/memory vs. the pod's `requests`, a taint the pod doesn't tolerate, an affinity rule that excludes this node, a required volume that isn't attached here. A modern installation ships roughly 14 in-tree Filter plugins. This phase runs in parallel across nodes, since each node's filter check is independent.

**Score** (historically "priorities"): of the nodes that survived filtering, rank them — spread load evenly, pack tightly to enable scale-down, prefer nodes already holding the pod's affinity-preferred neighbors, etc. Roughly 9 in-tree Score plugins, also run in parallel, with weighted scores summed per node.

```
Pod "worker-7" needs: cpu=2, memory=4Gi, must NOT run on nodes tainted "spot=true"
                       unless it tolerates that taint (it does, in this example)

Cluster: Node A (cpu avail=1, mem avail=8Gi)   → FILTERED OUT (insufficient CPU)
         Node B (cpu avail=4, mem avail=8Gi, tainted spot=true) → survives (tolerated)
         Node C (cpu avail=4, mem avail=2Gi)   → FILTERED OUT (insufficient memory)
         Node D (cpu avail=8, mem avail=16Gi)  → survives

Filter phase result: {Node B, Node D}

Score phase (e.g. weighing "least allocated" + "balanced resource"):
  Node B: 2 cpu / 4 avail = 50% would be used  → score 60
  Node D: 2 cpu / 8 avail = 25% would be used  → score 85  (more headroom left)

Winner: Node D (higher score) → pod binds here
```

Because Filter and Score are both plugin extension points, this is exactly how ecosystem tools plug in without forking `kube-scheduler`'s source — Volcano, Kueue's underlying scheduling logic, and the `scheduler-plugins` coscheduling plugin all register against these same extension points rather than replacing the scheduler wholesale.

---

## High-to-Low Walkthrough: `kubectl apply` to Running Container

```
kubectl apply -f pod.yaml
      │
      ▼
API server persists the Pod object       pod.status.phase = "Pending", no node
to etcd (see coordination/etcd.md)         assigned yet — the scheduler hasn't
                                            acted on it
      │
      ▼
Scheduler's watch loop notices the        added to the internal scheduling
unscheduled pod                            queue (PreEnqueue/QueueSort phases
                                            order it relative to other pending
                                            pods by priority)
      │
      ▼
Filter phase (parallel across all         eliminate infeasible nodes (as
nodes)                                     worked above)
      │
      ▼
Score phase (parallel across              rank survivors
survivors)
      │
      ▼
Reserve → Permit → Bind                   scheduler writes the binding
                                            decision back to the API server
                                            (pod.spec.nodeName = "node-d")
      │
      ▼
kubelet on the winning node notices        pulls the container image, creates
the new pod assigned to it                 the container runtime sandbox,
                                            starts the container(s)
      │
      ▼
pod.status.phase = "Running"
```

The literal commands to observe each side of this:

```bash
kubectl apply -f worker-pod.yaml
kubectl get pod worker-7 -o wide
# NAME       READY   STATUS    NODE      NOMINATED NODE
# worker-7   0/1     Pending   <none>    <none>          ← before scheduling

kubectl describe pod worker-7
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  2s    default-scheduler  Successfully assigned
#                                                default/worker-7 to node-d
#   Normal  Pulling    1s    kubelet            Pulling image "worker:v3"
#   Normal  Pulled     1s    kubelet            Successfully pulled image
#   Normal  Created    1s    kubelet            Created container worker
#   Normal  Started    1s    kubelet            Started container worker
```

The `Scheduled` event's message is the scheduler's actual, final decision — `node-d`, matching the worked Filter/Score example above.

---

## Deep Internals

### Requests vs. Limits: The Scheduler Only Sees Requests

This is the single most common source of confused capacity math: `kube-scheduler`'s bin-packing decisions use only a pod's **requests** (the guaranteed floor), never its **limits** (the ceiling it's allowed to burst to). A node can be scheduled at 100% of requested CPU across all its pods and still, in practice, run those pods at higher actual usage up to their limits — meaning the scheduler's view of "how full is this node" and a node's actual real-time resource pressure are two different numbers by design, not by bug.

```yaml
resources:
  requests:
    cpu: "500m"      # ← the scheduler bin-packs against THIS number
    memory: "1Gi"
  limits:
    cpu: "2"          # ← irrelevant to scheduling; enforced by the kubelet/
    memory: "2Gi"      #   cgroup at runtime, not by the scheduler at placement time
```

### Taints and Tolerations

A **taint** on a node repels pods unless they carry a matching **toleration** — the inverse of affinity (which attracts). This is the mechanism behind, e.g., a spot-instance NodePool tainted `spot=true:NoSchedule` so that only pods explicitly tolerating spot eviction land there:

```bash
kubectl taint nodes node-b spot=true:NoSchedule
kubectl get pod worker-7 -o jsonpath='{.spec.tolerations}'
# [{"key":"spot","operator":"Equal","value":"true","effect":"NoSchedule"}]
```

### Affinity, Anti-Affinity, and Topology Spread Constraints

Pod affinity/anti-affinity express "run me near/away from pods matching this label"; **topology spread constraints** are the more general, zone/region/node-aware version:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway   # ← SOFT default; see gotchas
    labelSelector:
      matchLabels: {app: worker}
```

### The Gang-Scheduling Gap, and Why It Was Bolted On

`kube-scheduler` schedules pods **independently** by default — there is no core concept of "these N pods are one logical job; place all N or none." For a job whose pods must all be running simultaneously to make progress (a Spark job's driver+executors, an MPI training job), independent per-pod scheduling creates a real deadlock risk: some pods land, others stay Pending indefinitely waiting for capacity, and the pods that *did* land sit there holding resources while accomplishing nothing, because the job can't actually start until every pod is up.

This is precisely the gotcha already flagged in [`compute/spark-on-eks.md`](../compute/spark-on-eks.md)'s gang-scheduling section — YARN's ApplicationMaster model sidesteps this because the AM negotiates and receives all the containers a job needs (or fails to) as part of its own request/response protocol with the ResourceManager, before the job's tasks begin executing; `kube-scheduler`'s pod-at-a-time design has no equivalent negotiation step, because pods, not jobs, are the unit it reasons about.

Three tools close this gap, and 2026's landscape treats them as complementary layers rather than competing choices:

- **Kueue**: admission control and quota — decides *which* jobs are allowed to enter the scheduling pool at all, and enforces fairness/quota across teams/queues. Does not itself do gang placement; can delegate to coscheduling for that.
- **Volcano**: adds actual gang-scheduling semantics plus DRF-style (Dominant Resource Fairness) multi-tenant fairness at the scheduler level — the layer that actually enforces "all pods in this PodGroup, or none."
- **`scheduler-plugins`' Coscheduling plugin**: a lighter-weight, in-tree-adjacent way to get PodGroup-based gang scheduling without adopting Volcano's fuller feature set.

In production AI/ML training setups (per 2026 sources), it's common to run Kueue and Volcano *together* — Kueue handling admission/quota, Volcano handling the actual gang placement — layered rather than either one alone.

```bash
# Coscheduling: mark a group of pods as one gang via a PodGroup CR
kubectl apply -f - <<EOF
apiVersion: scheduling.sigs.k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: spark-job-42
spec:
  minMember: 4          # driver + 3 executors — schedule as a unit
EOF
```

---

## Comparative

**YARN vs. Kubernetes scheduling** (see [`storage/hadoop.md`](../storage/hadoop.md) for YARN's own architecture in full): both are container/resource schedulers with a central decision-maker and per-node agents, but the *unit* each reasons about differs in a load-bearing way — YARN's ResourceManager talks to a per-job ApplicationMaster that requests a job's full container set as part of its own negotiation protocol, which is *why* gang-scheduling-like behavior falls out of YARN's design almost incidentally; `kube-scheduler` was designed around the pod as the atomic scheduling unit, with no job-level concept at the core layer at all, which is exactly why gang scheduling needed Kueue/Volcano/coscheduling bolted on years later as a separate ecosystem rather than shipping as a core feature.

**Mesos and Nomad — the honest 2026 status**: **Apache Mesos is retired and unmaintained as of October 2025** — there is no actively maintained path for new Mesos deployments, and the project's own migration guidance points existing users toward Kubernetes or Nomad. (Mesos's commercial backer, Mesosphere/D2iQ, had already ended DC/OS support back in 2021; a community fork, Clusterd, exists but has limited contributor activity and isn't a real drop-in replacement.) **HashiCorp Nomad** remains actively maintained and is one of the two recommended Mesos migration targets, but its own Spark integration (`nomad-spark`) is specifically deprecated and no longer maintained by HashiCorp — meaning Nomad is a reasonable general-purpose scheduler alternative today, but not a live option for Spark workloads specifically. For any Spark-on-a-scheduler decision in 2026, this narrows to Kubernetes or YARN as the two real choices — Mesos is off the table, and Nomad's Spark path is dead.

---

## Key Gotchas

- **Requests set too low silently defeats the scheduler's own bin-packing accuracy**: the scheduler places pods based on requests, but if requests are set far below actual usage, nodes that *look* comfortably packed to the scheduler can hit real memory/CPU pressure at runtime — this shows up as OOM kills or CPU throttling that has nothing to do with a scheduling decision being "wrong," because the scheduler never had accurate numbers to work with in the first place.
- **The gang-scheduling gap causes silent partial-deployment deadlocks, not loud failures**: without Kueue/Volcano/coscheduling, a job needing N pods simultaneously can end up with some pods Running and others stuck Pending indefinitely — nothing errors out, the job just never makes progress, and this is exactly the failure mode [`compute/spark-on-eks.md`](../compute/spark-on-eks.md) warns about for Spark driver+executor placement under cluster autoscaling pressure.
- **Taint/toleration misconfiguration produces a confusing, silent Pending state**: a pod that doesn't tolerate a node's taint simply never schedules there — `kubectl describe pod` will show a Filter-phase rejection reason, but teams unfamiliar with taints often chase the wrong root cause (assuming a resource shortage) before checking tolerations.
- **Topology spread constraints default to soft (`ScheduleAnyway`), not hard**: `whenUnsatisfiable: ScheduleAnyway` means the scheduler *tries* to spread pods across the given topology key but will still schedule a pod even if that means violating the spread constraint, if no better option exists — teams assuming spread is guaranteed (and only discovering it isn't during an actual zone outage) need `whenUnsatisfiable: DoNotSchedule` for a real hard guarantee, at the cost of pods potentially staying Pending if a spread-satisfying node genuinely isn't available.
- **GPU scheduling's device-plugin model means GPUs aren't bin-packed the way CPU/memory are** — the base scheduler treats GPUs as opaque "extended resources" advertised by a device plugin, without the same fractional/overcommit nuance CPU/memory get; fine-grained GPU sharing (multiple pods on one GPU) requires additional tooling (e.g. NVIDIA's device plugin extensions or DRA — Dynamic Resource Allocation) layered on top, not something the core scheduler does natively.

---

*Grounded against kubernetes.io scheduling documentation, the kubernetes/enhancements KEP repo (Scheduling Framework KEP-624, Gang Scheduling KEP-4671), and 2026 community writeups on Kueue/Volcano/coscheduling layering and Mesos's retirement status, as of September 2026. Mesos: confirmed retired/unmaintained as of October 2025, no supported migration path forward except to Kubernetes or Nomad. Nomad's Spark integration (`nomad-spark`) is confirmed deprecated by HashiCorp. Re-verify exact in-tree plugin counts (currently ~14 Filter / ~9 Score) against the specific Kubernetes minor version in use, since these change release to release.*
