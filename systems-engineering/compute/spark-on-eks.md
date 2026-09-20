# Spark on EKS — Production Reference

## 30-Second Intuition

Spark on EKS is just Spark's driver/executor model mapped onto Kubernetes pods — the driver is a long-lived pod that orchestrates, executors are ephemeral pods that compute. The hard parts are: shuffle data lives on executor pods (so killing them breaks dynamic allocation unless you use an External Shuffle Service), pod-to-pod networking must be flat and routable, and S3 is your durable layer for everything else. Get those three right and the rest is configuration.

---

## Table of Contents

1. [Architecture & Deployment Models](#1-architecture--deployment-models)
2. [Networking](#2-networking)
3. [Storage](#3-storage)
4. [Dynamic Allocation on EKS](#4-dynamic-allocation-on-eks)
5. [Resource Management](#5-resource-management)
6. [Performance Tuning](#6-performance-tuning)
7. [Observability](#7-observability)
8. [Operational Patterns](#8-operational-patterns)
9. [Concrete Production Example](#9-concrete-production-example)
10. [Comparison: EKS vs EMR Serverless vs Databricks](#10-comparison-eks-vs-emr-serverless-vs-databricks)
11. [Key Gotchas](#11-key-gotchas)

---

## 1. Architecture & Deployment Models

### The Mental Model

```
                    ┌─────────────────────────────────┐
                    │           EKS Cluster            │
                    │                                  │
  kubectl apply ──► │  ┌──────────────────────────┐   │
  SparkApplication  │  │   Spark Operator (CRD)   │   │
       or           │  │   watches SparkApp CRDs  │   │
  spark-submit ──►  │  └────────────┬─────────────┘   │
                    │               │ creates          │
                    │  ┌────────────▼─────────────┐   │
                    │  │      Driver Pod           │   │
                    │  │  (JVM, SparkContext)      │   │
                    │  └────────┬─────────────┬────┘   │
                    │           │requests     │        │
                    │  ┌────────▼──┐  ┌───────▼────┐  │
                    │  │Executor 1 │  │Executor 2  │  │
                    │  │(ephemeral)│  │(ephemeral) │  │
                    │  └───────────┘  └────────────┘  │
                    └─────────────────────────────────┘
```

### Spark Operator (CRD-Based)

The Spark Operator is a Kubernetes controller that watches `SparkApplication` custom resources and translates them into driver/executor pods.

**Use when:**
- You want GitOps / declarative job management
- You need retry policies, job history, status in `kubectl get sparkapplication`
- Running in a platform team context where users submit YAML, not CLI commands
- You want webhook-based admission control (e.g., inject sidecars, enforce labels)

**How it works:**
1. You `kubectl apply -f job.yaml` (SparkApplication CRD)
2. Operator reconciliation loop picks it up
3. Operator creates the driver pod with the right env vars and volume mounts
4. Driver pod starts, registers with K8s API server, requests executor pods via the K8s scheduler
5. Operator monitors pod status and updates SparkApplication `.status`

**Install:**
```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm install spark-operator spark-operator/spark-operator \
  --namespace spark-operator \
  --set webhook.enable=true \
  --set sparkJobNamespace=spark-jobs
```

### spark-submit on EKS (CLI-Based)

Directly calls the Spark binary with `--master k8s://https://<api-server>`.

**Use when:**
- Ad-hoc jobs, debugging, one-off runs
- CI/CD pipelines where you're already scripting
- You don't want the overhead of the Operator CRD

```bash
spark-submit \
  --master k8s://https://$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}') \
  --deploy-mode cluster \
  --name my-spark-job \
  --conf spark.kubernetes.container.image=123456789.dkr.ecr.us-east-1.amazonaws.com/spark:4.0.0 \
  --conf spark.kubernetes.namespace=spark-jobs \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa \
  --conf spark.executor.instances=10 \
  local:///opt/spark/jars/my-job.jar
```

**Key difference from Operator:** No CRD, no retry logic, no status tracking in K8s — just pods.

### SparkApplication CRD — Anatomy

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: my-etl-job
  namespace: spark-jobs
spec:
  type: Scala                        # or Python, Java, R
  mode: cluster                      # always cluster on K8s
  image: "123456789.dkr.ecr.us-east-1.amazonaws.com/spark:4.0.0"
  imagePullPolicy: IfNotPresent
  mainClass: com.example.MyJob
  mainApplicationFile: "s3a://my-bucket/jars/my-job.jar"
  arguments:
    - "--input-path=s3a://my-bucket/data/"

  sparkConf:
    "spark.sql.adaptive.enabled": "true"
    "spark.sql.shuffle.partitions": "200"

  driver:
    cores: 2
    coreLimit: "2000m"
    memory: "4g"
    memoryOverhead: "1g"
    labels:
      version: "4.0.0"
    serviceAccount: spark-sa
    nodeSelector:
      node-type: spark-driver
    tolerations:
      - key: "spark-driver"
        operator: "Exists"
        effect: "NoSchedule"
    volumeMounts:
      - name: spark-local-dir
        mountPath: /tmp/spark-local

  executor:
    cores: 4
    coreLimit: "4000m"
    instances: 5                     # static; use dynamicAllocation instead
    memory: "8g"
    memoryOverhead: "2g"
    labels:
      version: "4.0.0"
    nodeSelector:
      node-type: spark-executor
    volumeMounts:
      - name: spark-local-dir
        mountPath: /tmp/spark-local

  volumes:
    - name: spark-local-dir
      hostPath:
        path: /mnt/instance-store    # NVMe on i3/i3en instances

  dynamicAllocation:
    enabled: true
    initialExecutors: 2
    minExecutors: 1
    maxExecutors: 50
    shuffleTracking:
      enabled: true                  # required when no ESS

  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
```

### Driver Pod Lifecycle

```
kubectl apply SparkApplication
        │
        ▼
Operator creates Driver Pod
        │
        ▼
Driver Pod starts JVM, initializes SparkContext
        │
        ▼
SparkContext calls K8s API:
  POST /api/v1/namespaces/spark-jobs/pods  ← creates executor pods
        │
        ▼
Executors register back with driver via RPC (BlockManagerMaster)
        │
        ▼
Job runs stages/tasks
        │
        ▼
Driver deletes executor pods on completion or via dynamic allocation
        │
        ▼
Driver pod completes (Exit 0) → Operator marks SparkApplication as Completed
```

**Why the driver needs K8s API access:** It directly calls the K8s API to create/delete executor pods — not the Operator. The Operator only handles the SparkApplication lifecycle.

### Executor Pod Lifecycle

Executors are completely ephemeral:
- Created per-allocation request from driver
- Run tasks assigned by driver's DAGScheduler → TaskScheduler
- Hold shuffle data in local disk or memory (this is the fragile part)
- Killed when idle (dynamic allocation) or on job completion
- **Stateless**: they hold no permanent state — all output goes to S3

### RBAC Setup

```yaml
# ServiceAccount for Spark jobs
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark-sa
  namespace: spark-jobs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spark-role
  namespace: spark-jobs
rules:
  # Driver needs to create/list/delete executor pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "watch", "delete", "patch"]
  # For reading ConfigMaps (Spark config)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "get", "list", "watch", "delete"]
  # For services (driver headless service)
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "get", "list", "watch", "delete"]
  # For persistent volume claims (if used)
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["create", "get", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: spark-jobs
subjects:
  - kind: ServiceAccount
    name: spark-sa
    namespace: spark-jobs
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 2. Networking

### Driver ↔ Executor Communication

Two critical ports:
- **Driver port** (default 7078): TaskScheduler RPC — driver sends tasks to executors, executors report task status back
- **Block Manager port** (default 7079): shuffle data transfer — executors fetch shuffle blocks from each other

```
Driver Pod (10.0.1.5)
├── port 4040 → Spark UI
├── port 7078 → Driver RPC (tasks, scheduling)
└── port 7079 → Block Manager (shuffle read FROM driver)

Executor Pod (10.0.2.10)
├── port 7079 → Block Manager (shuffle data served here)
└── ← connects to driver:7078 on startup to register
```

**Why pod-to-pod networking matters:** Executors need to reach each other's Block Manager ports directly (not via a Service). This requires a flat pod network — no NAT between executor pods. VPC CNI (the standard EKS CNI) gives each pod a real VPC IP, so this works out of the box. Calico or Cilium also work if configured for pod CIDR routing.

**Anti-pattern:** Using host networking (`hostNetwork: true`) to avoid port conflicts. It works but breaks multi-tenant isolation.

### Headless Service for Driver

The Spark Operator (or spark-submit in cluster mode) creates a headless K8s Service for the driver so executors can find it by DNS rather than hardcoded IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-etl-job-driver-svc
  namespace: spark-jobs
spec:
  clusterIP: None        # headless — no VIP, DNS resolves to pod IP directly
  selector:
    spark-role: driver
    app-name: my-etl-job
  ports:
    - name: driver-rpc
      port: 7078
    - name: block-manager
      port: 7079
    - name: ui
      port: 4040
```

Executors connect to `my-etl-job-driver-svc.spark-jobs.svc.cluster.local:7078`.

### External Shuffle Service (ESS) vs Shuffle File Server

**Problem:** Spark's shuffle data lives on executor local disk. When an executor dies (spot eviction, OOM, dynamic allocation scale-down), its shuffle data is gone. Other executors reading that shuffle data get a `FetchFailedException` → stage retry → potentially full job retry.

| Approach | Description | Trade-offs |
|----------|-------------|------------|
| **No ESS (shuffle tracking)** | Dynamic allocation tracks which executors have shuffle data, won't kill them until data is consumed | Simple; executors live longer; data loss still possible on node failure |
| **External Shuffle Service** | DaemonSet on every node serves shuffle files; executor pods write to node-local disk, ESS survives executor death | Complex to set up; requires node-local storage; shuffle data survives executor churn |
| **Remote Shuffle Service** | Shuffle data written to remote store (Uber's RSS, Alibaba's RSS, or Gluten+Velox) | Fully decoupled; best for large-scale; significant engineering investment |

**ESS as DaemonSet:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: spark-shuffle-service
  namespace: spark-system
spec:
  selector:
    matchLabels:
      app: spark-shuffle-service
  template:
    metadata:
      labels:
        app: spark-shuffle-service
    spec:
      containers:
        - name: spark-shuffle-service
          image: apache/spark:4.0.0
          command: ["/opt/spark/sbin/start-shuffle-service.sh"]
          ports:
            - containerPort: 7337    # shuffle service port
          volumeMounts:
            - name: shuffle-dir
              mountPath: /tmp/spark-shuffle
      volumes:
        - name: shuffle-dir
          hostPath:
            path: /mnt/instance-store/spark-shuffle
      tolerations:
        - operator: "Exists"         # run on all nodes
```

Executor config:
```
spark.shuffle.service.enabled=true
spark.shuffle.service.port=7337
spark.dynamicAllocation.shuffleTracking.enabled=false  # ESS handles this
```

---

## 3. Storage

### S3A Connector Configuration

S3A is the Hadoop filesystem client for S3. Critical configs:

```
# Connection pool
spark.hadoop.fs.s3a.connection.maximum=200          # default 96; increase for large jobs
spark.hadoop.fs.s3a.connection.establish.timeout=5000
spark.hadoop.fs.s3a.connection.timeout=200000

# Threading
spark.hadoop.fs.s3a.threads.max=64
spark.hadoop.fs.s3a.multipart.size=128M
spark.hadoop.fs.s3a.fast.upload=true
spark.hadoop.fs.s3a.fast.upload.buffer=bytebuffer

# AWS credentials (prefer IRSA over hardcoded keys)
spark.hadoop.fs.s3a.aws.credentials.provider=com.amazonaws.auth.WebIdentityTokenCredentialsProvider

# Path style vs virtual hosted (use virtual hosted for new buckets)
spark.hadoop.fs.s3a.path.style.access=false
```

**IRSA (IAM Roles for Service Accounts) — the right way to auth:**
```yaml
# Annotate the K8s service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark-sa
  namespace: spark-jobs
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/spark-s3-role
```

### S3 Committer Algorithms

When Spark writes output to S3, it uses a "committer" to make the write atomic. S3 is eventually consistent for renames (it's actually a copy+delete, not a rename), so the choice matters.

| Committer | How it works | Use case |
|-----------|-------------|----------|
| **Directory committer** | Writes to temp dir, renames to final on commit | Old default; slow on S3 (rename = copy+delete); safe |
| **Magic committer** | Writes directly to final location using S3 multipart upload; manifest tracked in S3 | Fast; requires S3A magic committer support; preferred |
| **Iceberg/Delta native** | Table format handles commits via catalog | Best for table formats; bypass Spark committer entirely |

**Magic committer config:**
```
spark.hadoop.fs.s3a.committer.name=magic
spark.hadoop.fs.s3a.committer.magic.enabled=true
spark.sql.sources.commitProtocolClass=org.apache.spark.internal.io.cloud.PathOutputCommitProtocol
spark.sql.parquet.output.committer.class=org.apache.spark.internal.io.cloud.BindingParquetOutputCommitter
```

### Shuffle Storage: Instance Store vs EBS vs EFS

```
Latency (low → high):   Instance Store < EBS gp3 < EFS
Cost (low → high):      Instance Store < EBS < EFS
Availability on death:  Instance Store=gone, EBS=survives, EFS=survives
```

**Instance store (NVMe)** — best for shuffle:
- Available on i3, i3en, c5d, m5d, r5d families
- Sub-millisecond latency, 1-8 TB raw
- Formatted on node startup via DaemonSet or userdata script
- Use `hostPath` volume in executor pod pointing to `/mnt/nvme0n1/spark-tmp`

**EBS gp3** — good default:
- Persistent across pod restarts on same node
- Attach to node, not pod — you need the CSI driver and StoricClass
- More expensive than instance store for shuffle-heavy workloads
- ~3000 IOPS baseline, up to 16000 with provisioned

**EFS** — avoid for shuffle:
- Network filesystem; latency 1-3ms; not suited for random small I/O shuffle patterns
- Fine for checkpoint data shared across runs

### Shared PVC for Checkpoint Storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-checkpoint-pvc
  namespace: spark-jobs
spec:
  accessModes:
    - ReadWriteMany          # EFS supports this; EBS does not
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
```

Mount in SparkApplication:
```yaml
volumes:
  - name: checkpoint-vol
    persistentVolumeClaim:
      claimName: spark-checkpoint-pvc
driver:
  volumeMounts:
    - name: checkpoint-vol
      mountPath: /checkpoint
executor:
  volumeMounts:
    - name: checkpoint-vol
      mountPath: /checkpoint
```

### Iceberg/Delta on S3 from EKS

**Iceberg with Glue catalog:**
```
spark.sql.catalog.glue_catalog=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.glue_catalog.catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog
spark.sql.catalog.glue_catalog.io-impl=org.apache.iceberg.aws.s3.S3FileIO
spark.sql.catalog.glue_catalog.warehouse=s3a://my-lake/warehouse
spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
```

**Delta Lake:**
```
spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension
spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog
# Delta log checkpointing handles atomicity — no special S3 committer needed
```

---

## 4. Dynamic Allocation on EKS

### Why Vanilla Dynamic Allocation Breaks

In non-K8s Spark (e.g., YARN), when an executor is removed via dynamic allocation, the External Shuffle Service (a long-running daemon on each node) continues to serve that executor's shuffle files. On K8s, there's no such daemon by default.

```
Without ESS:
  Executor A writes shuffle block B → killed by dynamic allocation
  Executor C tries to read shuffle block B → FetchFailedException
  Stage B retries → Executor D re-computes → stage succeeds eventually
  (But: retry is expensive, and if no executor has the data, job fails)
```

### Two Solutions

**Solution 1: Shuffle tracking (simpler, Spark 3.x built-in)**

Spark tracks which executors have unreplicated shuffle data. Those executors are "decommissioning blocked" — they won't be killed until their shuffle data has been consumed by all downstream tasks.

```
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.shuffleTracking.enabled=true
spark.dynamicAllocation.shuffleTracking.timeout=30min   # how long to keep executor for shuffle
spark.dynamicAllocation.minExecutors=1
spark.dynamicAllocation.maxExecutors=50
spark.dynamicAllocation.executorIdleTimeout=60s
spark.dynamicAllocation.cachedExecutorIdleTimeout=300s
```

**Downside:** Executors with shuffle data live longer → less elastic. On spot eviction, data still lost.

**Solution 2: External Shuffle Service DaemonSet**

See [Networking section](#2-networking). Shuffle data on node-local disk, served by ESS pod that outlives executor pods.

```
spark.dynamicAllocation.enabled=true
spark.shuffle.service.enabled=true
spark.shuffle.service.port=7337
spark.dynamicAllocation.shuffleTracking.enabled=false
```

### Shuffle File Tracking Internals

1. Driver's `BlockManagerMaster` keeps a map: `shuffleId → Set[executorId]`
2. When an executor completes a shuffle write, it registers the block locations with `BlockManagerMaster`
3. On dynamic allocation scale-down, `ExecutorAllocationManager` asks `BlockManagerMaster`: "is this executor safe to remove?"
4. If executor has unread shuffle blocks → it's added to `decommissioning` state, not immediately killed
5. Once all shuffle blocks on that executor are read or the timeout expires → executor killed

### Dynamic Allocation + Karpenter

Karpenter is a node autoscaler that provisions EC2 nodes just-in-time based on pending pod requirements.

**API version note:** Karpenter's `NodePool`/`EC2NodeClass` CRDs graduated from beta to GA (`karpenter.sh/v1` and `karpenter.k8s.aws/v1`) with Karpenter 1.0 (mid-2024). The older `karpenter.sh/v1beta1` / `karpenter.k8s.aws/v1beta1` APIs are deprecated and no longer supported in current releases — use `v1` in all new manifests. The `nodeClassRef` field also changed shape: it now uses `group`/`kind` instead of `apiVersion`/`kind`.

```
Spark requests 20 more executor pods (pending: no nodes have capacity)
        │
        ▼
Karpenter sees pending pods, reads their node requirements
(resource requests, nodeSelector, tolerations, topology)
        │
        ▼
Karpenter launches EC2 instances matching requirements
(picks cheapest/fastest from allowed instance families + spot/on-demand)
        │
        ▼
Nodes join cluster, pods schedule
        │
        ▼
Job completes, executors removed
        │
        ▼
Karpenter consolidates/terminates empty nodes (within TTL)
```

**NodePool for Spark executors:**
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spark-executors
spec:
  template:
    metadata:
      labels:
        node-type: spark-executor
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: spark-executor-class
      requirements:
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["spot", "on-demand"]
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["r5.4xlarge", "r5.8xlarge", "r6i.4xlarge", "r6i.8xlarge"]
        - key: "topology.kubernetes.io/zone"
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      taints:
        - key: "spark-executor"
          effect: "NoSchedule"
  limits:
    cpu: 1000
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: spark-executor-class
spec:
  amiFamily: AL2
  role: KarpenterNodeRole-my-cluster
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  instanceStorePolicy: RAID0   # use NVMe instance store if available
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 200Gi
        volumeType: gp3
        iops: 6000
        throughput: 500
```

---

## 5. Resource Management

### Executor Sizing: The Memory Math

Spark executor memory has multiple layers:

```
Total container memory (what K8s sees):
├── spark.executor.memory          (JVM heap: execution + storage)
│   ├── spark.memory.fraction * heap → unified memory (execution + storage)
│   └── (1 - spark.memory.fraction) * heap → user memory (UDFs, data structures)
├── spark.executor.memoryOverhead  (off-heap: JVM overhead, NIO buffers, interned strings)
│   default: max(384MB, 0.1 * executor.memory)
└── spark.memory.offHeap.size      (optional explicit off-heap for Arrow, native)
```

**Setting K8s resource requests/limits:**
```yaml
executor:
  memory: "8g"                  # spark.executor.memory
  memoryOverhead: "2g"          # spark.executor.memoryOverhead
  # K8s container limit = memory + memoryOverhead = 10g
  cores: 4
  coreLimit: "4000m"            # K8s CPU limit = cores (set equal to avoid throttling)
```

**Rule of thumb:**
- 4-5 cores per executor (balance parallelism vs GC pressure)
- 4-8 GB heap per core (for joins/aggregations)
- memoryOverhead = 15-20% of heap, min 2GB for large heaps
- Never set CPU limit < CPU request (throttling causes GC pauses)

### Gang Scheduling

Spark's driver+executor set is exactly the "all-or-nothing" placement problem [`orchestration/cluster-schedulers.md`](../orchestration/cluster-schedulers.md) covers in full — kube-scheduler has no native concept of it, so without Kueue/Volcano/coscheduling, partial allocation across concurrent jobs can deadlock the cluster (see that doc's "Gang-Scheduling Gap" section for the mechanism and the PodGroup-based fix). The Spark-specific piece not covered there: enable it via `spark.kubernetes.scheduler.name=volcano` (or `yunikorn`) at submit time, and size `minAvailable` to `1 + numExecutors` so the driver plus every requested executor are treated as one gang.

### Spot Instances: Configuration

```yaml
executor:
  tolerations:
    - key: "karpenter.sh/capacity-type"
      operator: "Equal"
      value: "spot"
      effect: "NoSchedule"
  nodeSelector:
    karpenter.sh/capacity-type: spot
```

**Handling spot interruption (2-minute warning):**

Karpenter sends SIGTERM to pods on spot reclaim. Configure graceful decommission:
```
spark.executor.decommission.enabled=true
spark.storage.decommission.enabled=true
spark.storage.decommission.rddBlocks.enabled=true    # migrate cached RDD blocks
spark.storage.decommission.shuffleBlocks.enabled=true # migrate shuffle blocks
spark.storage.decommission.maxReplicationFailures=5
```

When SIGTERM received, executor:
1. Stops accepting new tasks
2. Migrates its shuffle blocks to surviving executors
3. Notifies driver → driver reschedules tasks from that executor
4. Exits cleanly

**Checkpoint strategy for spot resilience:**
```python
# Streaming: checkpoint to S3 (tolerates executor loss)
spark.readStream \
  .format("kafka") \
  .load() \
  .writeStream \
  .checkpointLocation("s3a://my-bucket/checkpoints/my-job") \
  .start()

# Batch: use Iceberg's ACID guarantees (partial writes don't corrupt table)
```

### Instance Family Selection

| Workload | Instance Family | Why |
|----------|----------------|-----|
| Large joins, aggregations (in-memory) | r5.4xlarge, r6i.8xlarge | 32GB+ memory/executor |
| CPU-intensive transforms, UDFs | c5.4xlarge, c5n.4xlarge | High core density, lower cost |
| Shuffle-heavy (large shuffles) | i3.4xlarge, i3en.6xlarge | NVMe local storage for shuffle |
| ML training (XGBoost, etc.) | m5.8xlarge | Balanced; GPU instances for DL |
| Streaming low-latency | c5.2xlarge (smaller, many) | Minimize task latency, fast scheduling |

---

## 6. Performance Tuning

### S3 Read Performance

```
# Parallel connections to S3
spark.hadoop.fs.s3a.connection.maximum=500

# Prefetch (Spark 3.3+ with new S3A connector)
spark.hadoop.fs.s3a.prefetch.enabled=true
spark.hadoop.fs.s3a.prefetch.block.size=8M

# Vectorized Parquet reader (avoid row-by-row deserialization)
spark.sql.parquet.enableVectorizedReader=true

# Columnar read for ORC
spark.sql.orc.enableVectorizedReader=true

# Input split size (match S3 object size for best parallelism)
spark.hadoop.mapreduce.input.fileinputformat.split.maxsize=134217728  # 128MB
```

**S3 rate limit handling:**
- S3 supports 5500 GET/sec per prefix per region
- Partition your S3 paths to spread across prefixes: `s3://bucket/year=2024/month=01/day=01/`
- Enable S3 Transfer Acceleration for cross-region

### Adaptive Query Execution (AQE)

AQE re-optimizes the query plan at runtime using actual shuffle statistics.

```
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
spark.sql.adaptive.coalescePartitions.minPartitionNum=1
spark.sql.adaptive.advisoryPartitionSizeInBytes=128MB
spark.sql.adaptive.skewJoin.enabled=true
spark.sql.adaptive.skewJoin.skewedPartitionFactor=5     # 5x median = skewed
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes=256MB
```

**AQE capabilities:**
1. **Coalesce shuffle partitions**: If `spark.sql.shuffle.partitions=200` but actual data is 20 partitions, AQE merges them → less overhead
2. **Skew join optimization**: Splits skewed partitions into smaller chunks, each processed by a separate task → no single task straggles
3. **Switch join strategies**: If one side of a join becomes small enough after filtering, AQE converts sort-merge join → broadcast hash join at runtime

```
Without AQE:
  sql.shuffle.partitions=200
  Stage 2 produces 20MB across 200 partitions → 200 tiny tasks (overhead-dominated)

With AQE:
  200 initial partitions → AQE coalesces to ~2 partitions (128MB advisory size)
  → 2 tasks, each doing meaningful work
```

### Broadcast Hash Join Threshold

```
spark.sql.autoBroadcastJoinThreshold=100MB  # default 10MB; increase if you have large dim tables
```

Spark collects statistics (from Glue/Hive metastore or `ANALYZE TABLE`) and auto-broadcasts tables smaller than this threshold.

**Force broadcast in code:**
```python
from pyspark.sql.functions import broadcast
df_large.join(broadcast(df_small), "key")
```

**When NOT to broadcast:** Table > 2GB (driver OOM); table is frequently updated (stale broadcast).

### Serialization

```
spark.serializer=org.apache.spark.serializer.KryoSerializer
spark.kryo.registrationRequired=false    # set true in prod to catch unregistered classes
spark.kryo.registrator=com.example.MyKryoRegistrator
```

**Kryo vs Java:**
- Java: ~10x larger serialized size, slower; works with any class
- Kryo: compact binary, fast; requires registration for complex class hierarchies
- Kryo savings: meaningful for RDD operations; less critical for DataFrame/SQL (Arrow columnar)

**Register custom classes:**
```scala
import org.apache.spark.serializer.KryoRegistrator
import com.esotericsoftware.kryo.Kryo

class MyKryoRegistrator extends KryoRegistrator {
  override def registerClasses(kryo: Kryo): Unit = {
    kryo.register(classOf[MyDomainObject])
    kryo.register(classOf[Array[MyDomainObject]])
  }
}
```

### GC Tuning for Large Heaps

Default GC (G1GC since Spark 3.x) is usually good. Tune for large heaps:

```
# Driver GC (smaller heap, latency-sensitive)
spark.driver.extraJavaOptions=-XX:+UseG1GC -XX:G1HeapRegionSize=32M \
  -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:ConcGCThreads=12 -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
  -Xloggc:/var/log/spark/gc-driver.log

# Executor GC (large heap, throughput-sensitive)
spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:G1HeapRegionSize=32M \
  -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:ConcGCThreads=12 -XX:MaxGCPauseMillis=500
```

**G1GC region sizing:** Set to `heap_size / 2048` rounded to power of 2. For 8GB heap: 8192/2048=4MB; for 32GB: 16MB.

**Signs of GC pressure:**
- `GC overhead limit exceeded` OOM
- Task duration variance (long pause → task slow → stage slow)
- High GC time in Spark UI → Executors tab

### Partition Sizing

```
Target: 128MB–200MB per partition (input data size / partition count)

# For shuffles:
spark.sql.shuffle.partitions=200        # default; set to 2-3x executor cores total
# e.g., 50 executors × 4 cores = 200 cores → 400-600 shuffle partitions

# Rebalance after reading small files or applying filters:
df.repartition(200)        # full shuffle, even distribution
df.coalesce(50)            # narrow transformation, may create skew but no shuffle
```

**coalesce vs repartition:**
- `coalesce(n)` where n < current: narrow dep, no shuffle, moves data to fewer partitions. Risk: if input is 1000 uneven partitions, coalesce(10) may be very skewed.
- `repartition(n)`: always shuffles, guarantees even distribution. Use when you need even partitions for downstream join/write.

---

## 7. Observability

### Spark History Server on EKS

Runs as a Deployment, reads event logs from S3.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-history-server
  namespace: spark-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spark-history-server
  template:
    spec:
      serviceAccountName: spark-sa
      containers:
        - name: spark-history-server
          image: apache/spark:4.0.0
          command:
            - /opt/spark/bin/spark-class
            - org.apache.spark.deploy.history.HistoryServer
          env:
            - name: SPARK_HISTORY_OPTS
              value: >-
                -Dspark.history.fs.logDirectory=s3a://my-bucket/spark-logs
                -Dspark.history.ui.port=18080
                -Dspark.history.fs.update.interval=10s
          ports:
            - containerPort: 18080
          resources:
            requests:
              cpu: "1"
              memory: "4Gi"
```

Enable event logging in jobs:
```
spark.eventLog.enabled=true
spark.eventLog.dir=s3a://my-bucket/spark-logs
spark.eventLog.compress=true
spark.eventLog.compression.codec=zstd
```

### Prometheus + Grafana

**Configure Spark metrics:**
```
# spark-metrics.properties (mounted as ConfigMap)
*.sink.prometheus.class=org.apache.spark.metrics.sink.PrometheusSink
*.sink.prometheus.period=15
*.sink.prometheus.unit=seconds

driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource
executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource
```

```
spark.metrics.conf=/etc/spark/metrics.properties
spark.metrics.namespace=my_etl_job
spark.ui.prometheus.enabled=true   # expose /metrics endpoint on UI port
```

**Key metrics to alert on:**
- `spark_executor_memoryUsed_bytes / spark_executor_maxMemory_bytes > 0.9` → OOM risk
- `spark_executor_failedTasks_total` spikes → task failures
- `spark_app_duration_seconds` vs SLA
- `spark_executor_gcTime_millis / spark_executor_cpuTime_millis > 0.1` → GC pressure

**Grafana dashboard query (PromQL):**
```promql
# Executor memory utilization
sum(spark_executor_memoryUsed_bytes{app_name="my-etl-job"}) by (executor_id) /
sum(spark_executor_maxMemory_bytes{app_name="my-etl-job"}) by (executor_id)
```

### Structured Streaming Monitoring

```python
from pyspark.sql.streaming import StreamingQueryListener

class MyStreamingListener(StreamingQueryListener):
    def onQueryStarted(self, event):
        print(f"Query started: {event.id}")

    def onQueryProgress(self, event):
        progress = event.progress
        print(f"Batch: {progress.batchId}, "
              f"Input rows/sec: {progress.inputRowsPerSecond:.2f}, "
              f"Processed rows/sec: {progress.processedRowsPerSecond:.2f}, "
              f"Trigger duration: {progress.triggerExecution}ms")

    def onQueryTerminated(self, event):
        if event.exception:
            print(f"Query failed: {event.exception}")

spark.streams.addListener(MyStreamingListener())
```

**Key streaming metrics:**
- `inputRowsPerSecond` vs `processedRowsPerSecond`: if processed << input, you're falling behind
- `triggerExecution.triggerExecution`: wall-clock time per batch
- `stateOperators[].numRowsUpdated`: state store growth (watch for unbounded state)

---

## 8. Operational Patterns

### Multi-Tenant EKS

```yaml
# Namespace per team
apiVersion: v1
kind: Namespace
metadata:
  name: spark-team-alpha
---
# ResourceQuota per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: spark-quota
  namespace: spark-team-alpha
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "400Gi"
    limits.cpu: "200"
    limits.memory: "800Gi"
    pods: "200"
    persistentvolumeclaims: "20"
---
# PriorityClass: high-priority jobs get preemption
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: spark-high-priority
value: 1000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
```

**Network policies for tenant isolation:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: spark-isolation
  namespace: spark-team-alpha
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: spark-team-alpha
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: spark-team-alpha
    - to: []   # allow egress to S3, external
      ports:
        - port: 443
```

### Job Queue with KEDA + SQS

Scale Spark job submissions based on SQS queue depth:

```yaml
# KEDA ScaledJob: triggers a new job pod per SQS message
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: spark-job-scaler
  namespace: spark-jobs
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    template:
      spec:
        containers:
          - name: spark-submitter
            image: my-spark-submitter:latest
            command: ["python", "/submit_spark_job.py"]
            env:
              - name: SQS_QUEUE_URL
                value: "https://sqs.us-east-1.amazonaws.com/123456789/spark-jobs"
  pollingInterval: 30
  maxReplicaCount: 10          # max concurrent Spark jobs
  scalingStrategy:
    strategy: "accurate"
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-aws-credentials
      metadata:
        queueURL: "https://sqs.us-east-1.amazonaws.com/123456789/spark-jobs"
        queueLength: "1"       # 1 job per message
        awsRegion: "us-east-1"
```

### CI/CD for Spark

**Dockerfile for Spark image:**
```dockerfile
FROM apache/spark:4.0.0

# Add custom JARs (Iceberg, Delta, custom UDFs)
COPY jars/ /opt/spark/jars/

# Add Python dependencies
COPY requirements.txt /tmp/
RUN pip install -r /tmp/requirements.txt

# Add job code
COPY jobs/ /opt/spark/jobs/

# Set proper permissions
RUN chown -R 185:185 /opt/spark
USER 185
```

**ECR push in CI:**
```bash
# In GitHub Actions / Jenkins
AWS_ACCOUNT=123456789
REGION=us-east-1
IMAGE=$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/spark-jobs

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com

docker build -t $IMAGE:$GIT_SHA .
docker tag $IMAGE:$GIT_SHA $IMAGE:latest
docker push $IMAGE:$GIT_SHA
docker push $IMAGE:latest

# Pin image version in SparkApplication YAML via kustomize overlay
```

**Version pinning strategy:** Use git SHA tags for reproducibility. Never rely on `latest` in production.

### Debugging

```bash
# Get driver pod name
kubectl get pods -n spark-jobs -l spark-role=driver

# Stream driver logs
kubectl logs -n spark-jobs spark-my-job-driver -f

# Get executor logs (they disappear on completion unless log aggregation set up)
kubectl logs -n spark-jobs spark-my-job-exec-1

# Port-forward Spark UI (4040) to localhost
kubectl port-forward -n spark-jobs spark-my-job-driver 4040:4040
# Then open http://localhost:4040

# Exec into driver for live debugging
kubectl exec -it -n spark-jobs spark-my-job-driver -- /bin/bash

# Describe a pod (events, resource limits, node assignment)
kubectl describe pod -n spark-jobs spark-my-job-driver

# Watch executor pods come and go (dynamic allocation)
kubectl get pods -n spark-jobs -l spark-role=executor -w

# Get SparkApplication status
kubectl get sparkapplication -n spark-jobs my-etl-job -o yaml | grep -A 20 status:
```

**Common failure modes:**

| Symptom | Likely cause | Debug |
|---------|-------------|-------|
| Executor pods Pending | Insufficient nodes/resources | `kubectl describe pod exec-1` → events |
| FetchFailedException | Shuffle data lost (executor died) | Enable ESS or shuffle tracking |
| Driver OOM | Collect to driver, large broadcast | Increase driver memory; avoid `collect()` |
| Task time variance | GC pauses or data skew | Check GC metrics; check AQE skew detection |
| Job hangs at 99% | Straggler task, speculative execution off | Enable `spark.speculation=true` |

---

## 9. Concrete Production Example

### Full SparkApplication YAML: Iceberg ETL Job

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: iceberg-daily-etl
  namespace: spark-jobs
  labels:
    team: data-platform
    environment: production
    pipeline: iceberg-etl
spec:
  type: Python
  mode: cluster
  image: "123456789.dkr.ecr.us-east-1.amazonaws.com/spark-iceberg:4.0.0-a1b2c3d"
  imagePullPolicy: IfNotPresent
  pythonVersion: "3"
  mainApplicationFile: "s3a://my-bucket/jobs/iceberg_etl.py"
  arguments:
    - "--date=2024-01-15"
    - "--source-table=raw.events"
    - "--target-table=curated.events_daily"

  sparkVersion: "4.0.0"

  sparkConf:
    # --- S3A ---
    "spark.hadoop.fs.s3a.impl": "org.apache.hadoop.fs.s3a.S3AFileSystem"
    "spark.hadoop.fs.s3a.aws.credentials.provider": "com.amazonaws.auth.WebIdentityTokenCredentialsProvider"
    "spark.hadoop.fs.s3a.connection.maximum": "500"
    "spark.hadoop.fs.s3a.fast.upload": "true"
    "spark.hadoop.fs.s3a.fast.upload.buffer": "bytebuffer"
    "spark.hadoop.fs.s3a.multipart.size": "128M"
    "spark.hadoop.fs.s3a.prefetch.enabled": "true"

    # --- Magic Committer ---
    "spark.hadoop.fs.s3a.committer.name": "magic"
    "spark.hadoop.fs.s3a.committer.magic.enabled": "true"
    "spark.sql.sources.commitProtocolClass": "org.apache.spark.internal.io.cloud.PathOutputCommitProtocol"
    "spark.sql.parquet.output.committer.class": "org.apache.spark.internal.io.cloud.BindingParquetOutputCommitter"

    # --- Iceberg ---
    "spark.sql.extensions": "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions"
    "spark.sql.catalog.glue_catalog": "org.apache.iceberg.spark.SparkCatalog"
    "spark.sql.catalog.glue_catalog.catalog-impl": "org.apache.iceberg.aws.glue.GlueCatalog"
    "spark.sql.catalog.glue_catalog.io-impl": "org.apache.iceberg.aws.s3.S3FileIO"
    "spark.sql.catalog.glue_catalog.warehouse": "s3a://my-lake/warehouse"

    # --- AQE ---
    "spark.sql.adaptive.enabled": "true"
    "spark.sql.adaptive.coalescePartitions.enabled": "true"
    "spark.sql.adaptive.advisoryPartitionSizeInBytes": "134217728"  # 128MB
    "spark.sql.adaptive.skewJoin.enabled": "true"
    "spark.sql.adaptive.skewJoin.skewedPartitionFactor": "5"

    # --- Dynamic Allocation ---
    "spark.dynamicAllocation.enabled": "true"
    "spark.dynamicAllocation.shuffleTracking.enabled": "true"
    "spark.dynamicAllocation.minExecutors": "2"
    "spark.dynamicAllocation.maxExecutors": "50"
    "spark.dynamicAllocation.executorIdleTimeout": "60s"
    "spark.dynamicAllocation.cachedExecutorIdleTimeout": "300s"

    # --- Serialization ---
    "spark.serializer": "org.apache.spark.serializer.KryoSerializer"

    # --- Event Logging ---
    "spark.eventLog.enabled": "true"
    "spark.eventLog.dir": "s3a://my-bucket/spark-logs"
    "spark.eventLog.compress": "true"

    # --- Metrics ---
    "spark.ui.prometheus.enabled": "true"
    "spark.metrics.namespace": "iceberg_daily_etl"

    # --- Graceful Decommission (for spot) ---
    "spark.executor.decommission.enabled": "true"
    "spark.storage.decommission.enabled": "true"
    "spark.storage.decommission.shuffleBlocks.enabled": "true"

    # --- Partitioning ---
    "spark.sql.shuffle.partitions": "400"

  driver:
    cores: 2
    coreLimit: "2000m"
    memory: "4g"
    memoryOverhead: "1g"
    serviceAccount: spark-sa
    labels:
      spark-role: driver
      pipeline: iceberg-etl
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    nodeSelector:
      node-type: spark-driver
    tolerations:
      - key: "spark-driver"
        operator: "Exists"
        effect: "NoSchedule"
    javaOptions: >-
      -XX:+UseG1GC
      -XX:G1HeapRegionSize=4M
      -XX:InitiatingHeapOccupancyPercent=35
      -XX:+PrintGCDetails
      -XX:+PrintGCDateStamps
    volumeMounts:
      - name: driver-tmp
        mountPath: /tmp/spark

  executor:
    cores: 4
    coreLimit: "4000m"
    memory: "16g"
    memoryOverhead: "3g"
    labels:
      spark-role: executor
      pipeline: iceberg-etl
    nodeSelector:
      node-type: spark-executor
    tolerations:
      # Allow spot instances
      - key: "karpenter.sh/capacity-type"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      - key: "spark-executor"
        operator: "Exists"
        effect: "NoSchedule"
    javaOptions: >-
      -XX:+UseG1GC
      -XX:G1HeapRegionSize=16M
      -XX:InitiatingHeapOccupancyPercent=35
      -XX:MaxGCPauseMillis=500
      -XX:ConcGCThreads=4
    volumeMounts:
      - name: executor-shuffle
        mountPath: /tmp/spark-shuffle

  volumes:
    - name: driver-tmp
      emptyDir: {}
    - name: executor-shuffle
      hostPath:
        path: /mnt/instance-store/spark-shuffle
        type: DirectoryOrCreate

  dynamicAllocation:
    enabled: true
    initialExecutors: 5
    minExecutors: 2
    maxExecutors: 50
    shuffleTracking:
      enabled: true
      timeout: "1800s"   # 30 min

  restartPolicy:
    type: OnFailure
    onFailureRetries: 2
    onFailureRetryInterval: 30
    onSubmissionFailureRetries: 3
    onSubmissionFailureRetryInterval: 20

  monitoring:
    exposeDriverMetrics: true
    exposeExecutorMetrics: true
    prometheus:
      jmxExporterJar: "/opt/prometheus/jmx_prometheus_javaagent.jar"
      port: 8090
```

### Walk-Through: kubectl apply to Job Completion

```
T+0s    kubectl apply -f iceberg-daily-etl.yaml
        → API server validates SparkApplication CRD
        → Operator reconciliation loop picks up new SparkApplication

T+1s    Operator creates driver pod:
        - Sets env: SPARK_DRIVER_BIND_ADDRESS, SPARK_DRIVER_PORT=7078
        - Mounts service account token for IRSA (S3 access)
        - Schedules on node with label node-type=spark-driver

T+5s    Driver pod starts, JVM initializes
        - SparkContext created, connects to K8s API
        - Operator creates headless Service for driver

T+8s    SparkContext requests 5 initial executor pods from K8s API
        → Karpenter sees 5 pending pods with nodeSelector=spark-executor
        → Karpenter launches 2x r5.4xlarge spot instances (fits 2-3 executors each)

T+45s   Nodes join cluster, executor pods schedule
        - Each executor registers with driver BlockManagerMaster
        - Driver logs: "Registered executor 1 on 10.0.2.5:7079"

T+60s   Job begins executing
        - Stage 0: scan Iceberg table from S3 (parallel S3 reads, vectorized parquet)
        - AQE monitors shuffle write statistics
        - Dynamic allocation: 5 executors → requests more as queue fills

T+90s   AQE triggers at stage boundary:
        - Original plan: sort-merge join on 200 partitions
        - AQE sees dim_table = 45MB → converts to broadcast hash join
        - Stage 1 completes 40% faster

T+5min  Dynamic allocation scales from 5 → 28 executors
        - Karpenter provisions 6 more spot instances
        - Shuffle tracking: executors with unread blocks not decommissioned

T+20min Stage 3: write output to Iceberg table
        - Magic committer: multipart uploads directly to final S3 path
        - No rename step → fast, atomic commit
        - Iceberg catalog updates table snapshot in Glue

T+22min All stages complete
        - Driver sends decommission signal to all executors
        - Executors drain tasks, exit cleanly
        - Driver pod exits 0
        - Operator marks SparkApplication as "Completed"
        - Event log written to S3, visible in History Server

T+25min Karpenter detects empty nodes → consolidates → terminates spot instances
        (within consolidateAfter=30s, so near-instant cost recovery)
```

---

## 10. Comparison: EKS vs EMR Serverless vs Databricks

| Dimension | Spark on EKS | EMR Serverless | Databricks |
|-----------|-------------|----------------|------------|
| **Control** | Full; you manage everything | Managed runtime; limited config | Managed; opinionated platform |
| **Cold start** | 30-120s (Karpenter node provisioning) | 60-180s (pre-initialized or cold) | 5-30s (pool warm-up) |
| **Cost at scale** | Lowest (spot + right-sizing) | Moderate (managed markup) | Highest (DBU pricing) |
| **Ops burden** | High (K8s, networking, images) | Low | Very low |
| **Spot support** | Native + Karpenter | Native | Instance pools (less granular) |
| **Dynamic allocation** | Manual ESS or shuffle tracking | Managed | Managed (photon-aware) |
| **Observability** | DIY (Prometheus, History Server) | CloudWatch + EMR UI | Built-in (Ganglia, Spark UI, jobs UI) |
| **ML integration** | BYO (MLflow, Ray, etc.) | Limited | Deep (MLflow native, Feature Store) |
| **Delta/Iceberg** | OSS; catalog config required | Iceberg native; Delta supported | Delta native (Databricks format); Iceberg read |
| **Multi-tenancy** | Namespace isolation, quotas | Separate applications | Workspaces, cluster policies |
| **Compliance/VPC** | Full VPC control | VPC-attached | Managed VPC or customer VPC |
| **Debugging** | `kubectl logs`, port-forward | EMR console, CloudWatch | Cluster event logs, Spark UI |
| **Version lock-in** | None (OSS) | EMR release cadence | Databricks Runtime (often 1-2 versions behind) |

**Pricing note (2026):** Databricks retired its Standard pricing tier (AWS/GCP: Oct 2025; Azure: Oct 2026, with new Standard workspaces already blocked since April 2026). Premium is now the pricing floor, which is roughly a 35%+ DBU rate increase for former Standard-tier customers — factor this into the "Highest (DBU pricing)" cost comparison above.

### When to Choose Each

**Choose Spark on EKS when:**
- You already run a K8s platform and want unified infrastructure
- Cost optimization is paramount (spot + right-sizing matters)
- You need full control over runtime, libraries, and configuration
- You have strong K8s/platform engineering expertise in-house
- You're already using Karpenter for other workloads

**Choose EMR Serverless when:**
- You want Spark without K8s overhead
- Team is AWS-native but not K8s-native
- Workloads are bursty and you want serverless billing
- You need AWS-managed Iceberg/Hudi integration
- Security/compliance requires AWS-managed service

**Choose Databricks when:**
- Unified platform for data engineering + ML + analytics is the priority
- Team productivity > infrastructure cost
- You need Delta Lake's advanced features (OPTIMIZE, ZORDER, liquid clustering)
- Collaborative notebooks + job scheduling in one tool
- Organization can absorb DBU pricing for speed of delivery

---

## 11. Key Gotchas

### Shuffle and Dynamic Allocation

**Gotcha 1:** `spark.dynamicAllocation.enabled=true` without either `shuffleTracking.enabled=true` or ESS will cause `FetchFailedException` on scale-down. Always pair dynamic allocation with one of these.

**Gotcha 2:** Shuffle tracking (`shuffleTracking.enabled=true`) doesn't protect against spot termination — it only prevents *voluntary* decommission. Enable `spark.executor.decommission.enabled=true` + block migration for spot resilience.

### Memory

**Gotcha 3:** `spark.executor.memory` sets JVM heap. K8s container limit = `executor.memory + memoryOverhead`. If you don't set `memoryOverhead` explicitly, the default is `max(384MB, 0.1 * memory)`. For large heaps (16GB+), this is often too small — add 2-4GB explicitly.

**Gotcha 4:** Off-heap memory (`spark.memory.offHeap.size`) is **in addition to** `executor.memory` and `memoryOverhead`. All three count against container memory. If you enable off-heap, add it to your container limit math.

### Networking

**Gotcha 5:** Executor pods need to reach each other's Block Manager ports directly (peer-to-peer). Security groups or network policies that restrict inter-pod traffic within the namespace will cause `BlockManagerException`. Always allow pod-to-pod traffic on the Spark port range (7000-7999 by default).

**Gotcha 6:** The driver headless Service must be created *before* executors start. The Spark Operator handles this. With raw `spark-submit`, the K8s native scheduler creates it automatically. If you're creating driver pods manually, create the Service first.

### S3

**Gotcha 7:** S3A magic committer requires S3 bucket versioning to be disabled (or the bucket must support magic committer uploads). Check with `spark.hadoop.fs.s3a.committer.staging.abort.pending.uploads=true` during testing.

**Gotcha 8:** `fs.s3a.connection.maximum` is per-executor, not per-cluster. 200 connections × 50 executors = 10,000 S3 connections. This is fine for S3 (high limits), but watch for internal S3-compatible storage with lower connection limits.

**Gotcha 9:** S3 list operations are expensive for tables with many small files. Iceberg metadata solves this (O(1) planning), but raw Parquet tables with thousands of files can cause job startup to take minutes just on file listing.

### K8s Scheduling

**Gotcha 10:** Without gang scheduling (Volcano/YuniKorn), partial-allocation deadlocks are possible in multi-job clusters. Two jobs each needing 10 executors may only get 8 each, and both stall. This is especially acute when using `spark.executor.instances` (static) rather than dynamic allocation.

**Gotcha 11:** Karpenter's consolidation (`consolidationPolicy: WhenUnderutilized`) can evict spot nodes during job execution for consolidation, even without a spot interruption. Set `karpenter.sh/do-not-disrupt: "true"` annotation on executor pods, or use `WhenEmpty` policy.

**Gotcha 12:** Pod disruption budgets (PDBs) don't prevent Karpenter node consolidation by default in newer Karpenter versions. Use the `do-not-disrupt` annotation instead.

### AQE

**Gotcha 13:** AQE's skew join optimization splits large partitions into smaller sub-partitions. This works transparently but can cause the number of tasks to be non-deterministic. If downstream code assumes `spark.sql.shuffle.partitions` partitions exactly, it may break.

**Gotcha 14:** AQE broadcast threshold is checked *after* filtering. A fact table joining a dimension may trigger a broadcast mid-query even if the dimension appears large in the catalog stats. This is usually good but can cause OOM if the dimension is larger than expected post-filter. Increase driver memory or set `spark.sql.autoBroadcastJoinThreshold=-1` to disable.

### Operations

**Gotcha 15:** Executor pod logs are gone after pod deletion unless you've set up a log aggregator (Fluent Bit → S3/CloudWatch). Always configure log aggregation before troubleshooting production issues — you won't have executor logs after the fact.

**Gotcha 16:** SparkApplication CRD retries create *new* driver pods, not restarts of the failed one. Each retry creates a new pod name. If your job has side effects (writes to Iceberg), make sure your job code is idempotent or uses Iceberg's ACID guarantees to avoid duplicate writes.

**Gotcha 17:** `imagePullPolicy: Always` on every pod means a registry roundtrip per executor pod. With 50 executors spinning up simultaneously, this can overwhelm a small ECR endpoint or cause rate limiting. Use `IfNotPresent` in production and pin image tags to immutable digests.
