# Karpenter: Just-in-Time Node Provisioning for Kubernetes

## 30-Second Intuition

Karpenter watches for unschedulable pods and provisions the exact right EC2 instance to fit them — in under 60 seconds. Unlike Cluster Autoscaler (which adjusts node group sizes), Karpenter bypasses node groups entirely: it talks directly to the EC2 API, picks the optimal instance type for the pending pods, and registers the node. When nodes become underutilized, Karpenter consolidates workloads and terminates the excess instances.

> Note: The term "CARPENTER" in some contexts refers to this AWS-native tool, which is officially named **Karpenter**.

---

## Karpenter vs Cluster Autoscaler

```
Cluster Autoscaler (CA) model:
  Pod unschedulable → CA finds a node group that fits → increments ASG desired count
  → ASG launches instance → node joins → pod scheduled
  Problems:
  - Node groups must be pre-configured with specific instance types
  - Scaling decision: "which node group?" (coarse)
  - Launch time: 3-5 min (ASG + bootstrap)
  - Bin packing: poor (same instance size for all nodes in group)
  - Consolidation: optional and slow

Karpenter model:
  Pod unschedulable → Karpenter evaluates all instance families/sizes → picks optimal fit
  → Calls EC2 CreateFleet API directly → node joins → pod scheduled
  Advantages:
  - No node groups — pod-first provisioning
  - Launch time: 30-90s (direct EC2 + fast bootstrap)
  - Bin packing: optimal (picks smallest instance that fits pods)
  - Consolidation: aggressive and automatic
```

---

## Architecture

```
                    K8s API Server
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    Karpenter         NodeClaim         NodePool
    Controller         CRD               CRD
         │                │                │
         ├─ watches unschedulable pods      │
         ├─ evaluates NodePool constraints ─┘
         ├─ calls EC2 CreateFleet API
         ├─ creates NodeClaim (tracks instance)
         └─ watches node readiness → schedules pods

    EC2 API ──── CreateFleet (spot/on-demand, multi-AZ, multi-type)
```

**Key CRDs:**

| CRD | Purpose |
|---|---|
| `NodePool` | Defines what kinds of nodes Karpenter can provision (instance families, zones, capacity types) |
| `EC2NodeClass` | AWS-specific config: AMI, security groups, IAM role, user data |
| `NodeClaim` | Represents a specific node that Karpenter provisioned (internal state tracking) |

---

## NodePool

Defines the constraints for provisioning:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        managed-by: karpenter
      annotations:
        node.kubernetes.io/exclude-from-external-load-balancers: "true"
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      requirements:
        # Instance families — Karpenter chooses cheapest fit
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [m5, m6i, m6a, m7i, c5, c6i, r5]

        # Capacity types — spot preferred
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]

        # Architecture
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]

        # Zones
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b, us-east-1c]

        # Instance size (optional — let Karpenter choose)
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: [nano, micro, small]     # exclude tiny instances

  # Disruption / consolidation policy
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s         # how long to wait before consolidating
    budgets:
      - nodes: "10%"              # max 10% of nodes disrupted at once
        schedule: "0 9 * * 1-5"  # only during business hours for this budget
        duration: 8h
      - nodes: "5%"              # off-hours: gentler budget

  limits:
    cpu: "1000"                   # max CPU across all nodes in this pool
    memory: 4000Gi                # max memory

  weight: 10                      # if multiple NodePools, higher weight = preferred
```

---

## EC2NodeClass

AWS-specific instance configuration:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # AMI selection — use AL2023 or bottlerocket
  amiFamily: AL2023

  # Or pin specific AMI (overrides amiFamily)
  # amiSelectorTerms:
  #   - id: ami-0abcdef1234567890

  # Subnet selection (by tag)
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

  # Security group selection
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

  # IAM role for the node
  role: KarpenterNodeRole-my-cluster

  # Instance store (NVMe) optimization
  instanceStorePolicy: RAID0     # aggregate NVMe disks

  # EBS root volume
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 10000
        throughput: 500
        encrypted: true

  # User data (injected into cloud-init)
  userData: |
    #!/bin/bash
    echo "KUBELET_EXTRA_ARGS=--max-pods=110" >> /etc/default/kubelet

  # Tags applied to EC2 instances
  tags:
    team: platform
    environment: production
    karpenter.sh/discovery: my-cluster
```

---

## How Karpenter Provisions: The Decision Loop

```
1. Pod scheduling fails (no node with sufficient resources)
   → kubescheduler marks pod Unschedulable

2. Karpenter controller watches for Unschedulable events

3. Simulation: for each pending pod, compute resource requirements
   (CPU + memory + extended resources like GPU, EFA)

4. Node shape selection:
   a. Filter NodePool requirements (zone, arch, family, capacity type)
   b. Filter out taints that pods don't tolerate
   c. From remaining instance types: find minimum cost that fits ALL pending pods
      (Karpenter batches: waits up to 1s for more pods before deciding)

5. EC2 CreateFleet API call:
   - Pass ordered list of instance types (primary + fallbacks)
   - Specify: LaunchTemplateId, SubnetIds, CapacityType
   - EC2 returns: InstanceId, actual instance type launched

6. Karpenter registers node:
   - Creates NodeClaim (internal tracking)
   - Registers node with K8s API
   - Adds labels: node.kubernetes.io/instance-type, topology.kubernetes.io/zone, etc.

7. Node joins cluster (kubelet registers)
   → Pending pods get scheduled → Running
```

**Why it's fast:** Karpenter doesn't wait for ASG lifecycle hooks. It calls CreateFleet directly and monitors the instance until kubelet is ready.

---

## NodePool Constraints: Instance Families, Zones, Capacity Types

### Spot vs On-Demand Priority

```yaml
# Two NodePools: spot-preferred with on-demand fallback
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-workers
spec:
  weight: 100          # higher weight = Karpenter tries this first
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [m5, m6i, c5, c6i, r5, r6i]
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: on-demand-fallback
spec:
  weight: 1            # lower weight = fallback
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [m5, m6i]
```

### GPU NodePool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  template:
    spec:
      nodeClassRef:
        name: gpu-node-class
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule    # only GPU-requesting pods land here
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [p3, p4, g4dn, g5]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
  disruption:
    consolidationPolicy: WhenEmpty   # don't consolidate GPU nodes unless empty
    consolidateAfter: 5m
  limits:
    "nvidia.com/gpu": "50"
```

---

## Consolidation

Karpenter continuously evaluates whether running nodes are underutilized and can be consolidated:

```
Consolidation policies:
  WhenEmpty:               remove nodes with no pods (except daemonsets)
  WhenEmptyOrUnderutilized: also reschedule pods to pack them onto fewer nodes

Algorithm (WhenEmptyOrUnderutilized):
  1. Find nodes where total pod resource requests << node capacity
  2. Simulate: can ALL pods on this node fit on other existing nodes?
  3. If yes: drain the node (evict pods with PodDisruptionBudgets respected)
  4. Terminate EC2 instance
  5. Pods reschedule on existing nodes (no new node needed)

Example:
  Node A: 8 vCPU, 4 pods using 1 vCPU each = 50% utilized
  Node B: 8 vCPU, 6 pods using 1 vCPU each = 75% utilized
  
  Karpenter simulates: can Node A's 4 pods fit on Node B?
  Node B has 2 vCPU free → only 2 more pods fit → NO consolidation
  
  Node C added with 2 pods × 1 vCPU:
  Can Node C's 2 pods fit on Node B? Node B has 2 vCPU free → YES
  → Evict Node C's pods → they reschedule to Node B → terminate Node C
```

**Disruption budgets** prevent consolidation from taking down too much at once:
```yaml
disruption:
  budgets:
    - nodes: "10%"      # max 10% of nodes disrupted simultaneously
    - nodes: "5"        # or absolute number: max 5 nodes at once
```

---

## Disruption Budgets

Control when and how aggressively Karpenter can disrupt nodes:

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m

  budgets:
    # Allow up to 20% disruption during business hours
    - nodes: "20%"
      schedule: "0 8 * * 1-5"    # Monday-Friday 8am
      duration: 9h                 # for 9 hours

    # Very conservative off-hours
    - nodes: "5%"

  # Block consolidation if pod budget would be violated
  # (Karpenter respects PodDisruptionBudgets automatically)
```

Karpenter also respects:
- `do-not-disrupt: "true"` annotation on pods → pod blocks node consolidation.
- `do-not-disrupt: "true"` annotation on nodes → node won't be consolidated.
- `PodDisruptionBudgets` → Karpenter checks PDB before draining.

---

## KEDA + Karpenter: The Ideal Combination

```
Queue fills up
  → KEDA detects queue depth
  → KEDA scales Deployment: 0 → 50 replicas
  → 50 pods pending (no nodes available)
  → Karpenter sees 50 unschedulable pods
  → Karpenter calculates: need ~10 × m6i.2xlarge nodes
  → EC2 CreateFleet: launches 10 nodes in parallel
  → Nodes ready in 60-90s
  → 50 pods schedule and run

Queue drains
  → KEDA scales Deployment: 50 → 0
  → All pods terminate
  → 10 nodes become empty
  → Karpenter consolidation: WhenEmpty → terminates 10 nodes
  → Nodes terminated in ~2 min (configurable consolidateAfter)

Total cost: pay only for compute while processing
```

---

## Concrete Example: Spark on EKS with Karpenter

Dynamic executor provisioning for Apache Spark jobs:

```yaml
# spark-node-class.yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: spark-nodes
spec:
  amiFamily: AL2023
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-eks-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-eks-cluster
  role: KarpenterNodeRole-my-eks-cluster
  instanceStorePolicy: RAID0    # use NVMe for Spark shuffle
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 200Gi
        volumeType: gp3
  tags:
    karpenter.sh/discovery: my-eks-cluster
    workload: spark

---
# spark-nodepool.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spark-executor-pool
spec:
  weight: 50
  template:
    metadata:
      labels:
        workload: spark-executor
    spec:
      nodeClassRef:
        name: spark-nodes
      taints:
        - key: workload
          value: spark
          effect: NoSchedule
      requirements:
        # Memory-optimized and compute-optimized for Spark
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [r5, r6i, r6a, r5d, m5, m6i]
        # Instance sizes with enough memory for typical executors
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: [xlarge, 2xlarge, 4xlarge, 8xlarge]
        # Spot first (Spark handles task retries)
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        # All AZs for spot availability
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b, us-east-1c]
      # Spread across AZs to reduce spot interruption impact
      topologySpreadConstraints:
        - maxSkew: 2
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
  disruption:
    consolidationPolicy: WhenEmpty       # don't rebalance mid-job
    consolidateAfter: 30s
  limits:
    cpu: "2000"
    memory: 8000Gi

---
# spark-driver-pool.yaml (on-demand for the driver)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spark-driver-pool
spec:
  template:
    spec:
      nodeClassRef:
        name: spark-nodes
      taints:
        - key: workload
          value: spark-driver
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]        # driver must not be interrupted
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [m6i, m5]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: [xlarge, 2xlarge]
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 1m
```

```python
# SparkApplication (using spark-on-k8s-operator)
# Driver pod has toleration for spark-driver taint
# Executor pods have toleration for spark taint

spark_conf = {
    "spark.kubernetes.driver.node.selector.workload": "spark-driver",
    "spark.kubernetes.executor.node.selector.workload": "spark-executor",
    "spark.kubernetes.driver.tolerations": "workload=spark-driver:NoSchedule",
    "spark.kubernetes.executor.tolerations": "workload=spark:NoSchedule",
    "spark.executor.instances": "20",
    "spark.executor.memory": "14g",
    "spark.executor.cores": "4",
    # Karpenter will provision r6i.xlarge (4 vCPU, 32GB) for each executor
}
```

**Execution trace:**
```
1. Spark job submitted: 1 driver + 20 executors requested
2. Driver pod: pending (no spark-driver taint node)
   → Karpenter provisions m6i.xlarge → driver schedules

3. 20 executor pods: pending (no spark-executor taint nodes)
   → Karpenter sees 20 pods, each requesting 4 CPU/14GB
   → Calculates: r6i.xlarge (4 vCPU, 32GB) fits 2 executors
   → Need 10 × r6i.xlarge nodes
   → EC2 CreateFleet: [r6i.xlarge, r6i.2xlarge, r5.xlarge, ...] (fallbacks)
   → 10 nodes launched in parallel (~60s)
   → 20 executors schedule

4. Job completes, executors exit
   → 10 executor nodes empty → Karpenter consolidates after 30s
   → 10 nodes terminated
   → Driver node empty → terminated after 1m
```

---

## When to Use / When NOT to Use Karpenter

**Use Karpenter when:**
- EKS on AWS (Karpenter is AWS-native; AKS/GKE have different solutions).
- Workloads with highly variable scale (batch jobs, ML training, data pipelines).
- Mixed spot/on-demand requirements.
- Multiple different instance type needs in one cluster.
- You want aggressive cost optimization via consolidation.

**Do NOT use Karpenter when:**
- You're on GKE or AKS (use GKE Autopilot / AKS VMSS instead).
- Strict compliance requires exact instance type control.
- You rely on Cluster Autoscaler features like per-node-group scale-to-zero (use CA for those specific node groups, Karpenter for the rest).
- You have workloads with `do-not-disrupt` requirements — they'll block consolidation and negate the cost savings.

---

## Key Gotchas

1. **Karpenter manages node lifecycle — don't also run Cluster Autoscaler on the same node groups.** They'll conflict. Use either/or, or assign Karpenter to handle specific node pools and CA for others.

2. **Spot interruptions require fault-tolerant workloads** — Spot instances can be reclaimed with a 2-minute warning. Spark, Flink, and Argo Workflows with retries handle this; stateful services (databases) should use on-demand NodePools.

3. **NodePool `limits` are global, not per-instance** — `cpu: "1000"` means total CPUs across ALL nodes in the pool, not per node. A pool can have 100 nodes with 10 vCPU each = 1000 total.

4. **Consolidation disrupts pods** — `WhenEmptyOrUnderutilized` will evict pods to consolidate. Use `PodDisruptionBudgets` and `do-not-disrupt` annotations on critical pods.

5. **`consolidateAfter` too short causes flapping** — if a node goes empty briefly (between tasks) and Karpenter terminates it, the next task has to wait for a new node. Set consolidateAfter based on your job inter-arrival time.

6. **`karpenter.sh/discovery` tags are critical** — subnets and security groups must be tagged. Missing tags = Karpenter can't launch nodes.

7. **IAM role permissions** — Karpenter controller needs `ec2:CreateFleet`, `ec2:RunInstances`, `ec2:TerminateInstances`, and IAM PassRole. Use the official IAM policy template.

8. **Topology spread constraints interact with spot** — if you require `maxSkew: 1` across 3 AZs but one AZ has no spot capacity, provisioning fails. Use `whenUnsatisfiable: ScheduleAnyway` for spot workloads.

9. **Karpenter v1beta1 → v1 migration** — API versions changed. NodePool replaces Provisioner; EC2NodeClass replaces AWSNodeTemplate. Old CRDs were removed in Karpenter 1.0.
