# Apache Hadoop (HDFS + YARN)

## 30-Second Intuition

"Hadoop" today means two mostly-independent systems bundled under one project: **HDFS** (a distributed filesystem that splits files into large blocks and replicates them across machines) and **YARN** (a cluster resource manager that schedules containerized jobs onto those same machines). The original third piece, **MapReduce**, is legacy — modern workloads run Spark or Tez on top of YARN, or increasingly run on Kubernetes instead of YARN entirely, while still reading/writing HDFS or (more commonly now) S3-compatible object storage. The one fact that matters operationally: HDFS's defining decision is **replicate the data, then schedule the computation to where a replica already lives** — data locality is the point of the whole design, and it only works because compute and storage are colocated on the same physical nodes. Once you move to disaggregated object storage (S3), that locality guarantee disappears — which is exactly why the industry's shift to S3-backed lakehouses (Iceberg/Delta on S3, Spark-on-EKS) means Hadoop-the-storage-layer is far less central than Hadoop-the-scheduler-precedent (YARN's ideas heavily shaped how EMR, Kubernetes scheduling, and modern gang schedulers work).

---

## Resource-Layer Map

| Layer | Role Hadoop plays | What it's optimizing for |
|---|---|---|
| CPU | YARN schedules compute containers onto the same nodes that hold the data those containers will read | Minimize network transfer by running the computation next to the bytes, not the other way around — "move compute to data," the inverse of most other distributed systems |
| Memory | YARN NodeManagers track and enforce per-container memory limits; NameNode holds the entire filesystem namespace/block map in memory | NameNode memory is the classic HDFS scaling wall — every file, directory, and block reference lives in NameNode RAM, so small-file-heavy workloads exhaust memory long before they exhaust disk |
| Disk | Large (128MB-256MB+) blocks, sequential read/write per block, replicated 3x by default (or erasure-coded) | Optimized for a small number of very large sequential reads/writes, not many small random ones — the opposite profile from a transactional database |
| Network | Block replication (writer → 3 DataNodes) and any read that misses local data all cross the network | Data locality scheduling exists specifically to keep this row cheap — a job that reads its input from local disk instead of over the network is the entire justification for the compute-follows-data design |
| GPU | Not applicable at the HDFS/YARN layer itself | YARN can schedule GPU-aware containers for frameworks that need them, but GPU scheduling is a bolt-on capability, not something HDFS/YARN's core design is built around |

This is the sharpest contrast with S3/object storage: S3 has no concept of "run compute here because the data is here" — compute and storage are architecturally separated, and every read is a network call by design. Hadoop's entire value proposition assumed compute and storage were the same cluster; that assumption is precisely what the object-storage-based lakehouse model abandoned.

---

## The Signature Mechanism: Block Replication + Data-Locality Scheduling

HDFS splits every file into fixed-size **blocks** (128MB default, configurable) and replicates each block to multiple **DataNodes** (default replication factor 3). The **NameNode** is the single source of truth for two things: the filesystem namespace (directory tree, file→block-list mapping) and, transiently, which DataNodes currently hold which blocks (DataNodes report block locations to the NameNode via periodic heartbeats — this mapping is not persisted by the NameNode itself, it's rebuilt from DataNode reports on startup).

The mechanism that makes this pay off computationally: when YARN schedules a container to process a chunk of an HDFS file, the **ResourceManager's scheduler asks the NameNode (via the job's input-split metadata) which nodes hold the relevant blocks**, and preferentially places the container on one of those nodes (node-local), or failing that, a node in the same rack (rack-local), before falling back to an arbitrary node (off-rack, requiring a full network transfer). This preference ordering — node-local > rack-local > off-rack — is data locality scheduling, and it's the direct payoff of having replicated the block to 3 places in the first place: 3 replicas means 3 chances for the scheduler to find a node-local placement instead of pulling the block over the network.

```
Block "orders-part-0042" replicated to DataNodes: [Node A, Node D, Node G]

YARN scheduling a MapReduce/Spark task that reads this block:
  1. Check: is a NodeManager slot free on Node A, D, or G right now?
     → YES: schedule there (node-local) — read block off local disk, zero network
  2. If no slot free on any replica's node:
     check same-rack nodes as A/D/G
     → schedule there (rack-local) — one network hop within the rack switch
  3. Otherwise: schedule anywhere with a free slot (off-rack)
     → full cross-rack network transfer to read the block
```

This directly explains the resource-layer table's network row: locality-aware scheduling is the *only* reason Hadoop-era clusters could process petabytes without saturating network fabric — most reads never left the node they were scheduled on.

You can see the replica placement this scheduling decision depends on directly with `fsck`:

```bash
$ hdfs fsck /data/orders/orders.parquet -files -blocks -locations

/data/orders/orders.parquet 268435456 bytes, 2 block(s):
 OK
 0. BP-1084160542-10.0.1.10-1690000000000:blk_1073741825_1001 len=134217728 Live_repl=3
    [DatanodeInfoWithStorage[10.0.1.11:9866,DS-...,DISK],   <- rack /rack1
     DatanodeInfoWithStorage[10.0.1.24:9866,DS-...,DISK],   <- rack /rack2
     DatanodeInfoWithStorage[10.0.1.26:9866,DS-...,DISK]]   <- rack /rack2
 1. BP-1084160542-10.0.1.10-1690000000000:blk_1073741826_1002 len=134217728 Live_repl=3
    [DatanodeInfoWithStorage[10.0.1.12:9866,DS-...,DISK],   <- rack /rack1
     DatanodeInfoWithStorage[10.0.1.23:9866,DS-...,DISK],   <- rack /rack2
     DatanodeInfoWithStorage[10.0.1.27:9866,DS-...,DISK]]   <- rack /rack2

Status: HEALTHY
 Total size: 268435456 B
 Total blocks (validated): 2 (avg. block size 134217728 B)
 Minimally replicated blocks: 2 (100.0 %)
 Number of data-nodes: 6
 Number of racks: 2
```

Each block's 3 replicas straddle 2 racks (1 local rack, 1 remote) — exactly the placement policy described above. This is what gives YARN's scheduler its 3 chances at node-local placement per block.

Once a job actually runs, `yarn logs` shows whether the scheduler cashed in on that locality — the counters below are from a completed MapReduce/Spark job reading this same table:

```bash
$ yarn logs -applicationId application_1690000000000_0042 | grep -A4 "Locality"

    Data-local map tasks=118
    Rack-local map tasks=9
    Other local map tasks=1
```

118 of 128 map tasks ran node-local (read the block off local disk, zero network), 9 fell back to rack-local (one hop within the rack switch), and only 1 went fully off-rack — this ratio is the direct payoff of 3x replication: with only 1 replica, most of these tasks would have had to go off-rack instead.

**Erasure coding (HDFS 3.x+) breaks this tradeoff on purpose**: EC stores data as data+parity fragments striped across nodes (roughly 50% storage overhead instead of replication's 200% overhead for factor-3), but a file encoded this way has no single node holding a complete, locally-readable copy — every read and write to an EC-encoded file requires reconstructing from fragments across multiple nodes, which means **EC files are guaranteed to incur remote network traffic on every access**, trading away data locality entirely for storage efficiency. This is a real architectural decision per dataset, not just a config toggle: EC is the right choice for cold, rarely-read data (archival, compliance retention) where storage cost dominates; it is the wrong choice for hot data where locality-driven compute performance matters.

---

## High-to-Low Walkthrough: Client Write Path

The client-facing side of this trace is a single CLI call:

```bash
$ hdfs dfs -put orders.parquet /data/orders/
```

Everything below is what that one command triggers, level by level:

```
DistributedFileSystem.create("orders.parquet")
      │
      ▼
NameNode RPC: create file entry     NameNode checks permissions, creates the
                                     file's namespace entry (no blocks yet),
                                     returns an initial block ID + a list of
                                     DataNodes to write the first block's
                                     replicas to (chosen by the block placement
                                     policy: 1st replica on/near the writer,
                                     2nd on a different rack, 3rd on the same
                                     rack as the 2nd — this rack-awareness is
                                     itself a durability/network tradeoff:
                                     survives a rack failure, limits cross-rack
                                     replication traffic to one hop)
      │
      ▼
Client writes to a DataNode pipeline   client streams the block's bytes to the
(replication pipeline)                  FIRST DataNode in the list, which
                                         simultaneously forwards each packet to
                                         the SECOND DataNode, which forwards to
                                         the THIRD — a pipelined write, not
                                         "client sends to 3 nodes separately"
      │
      ▼
Each DataNode: local disk append    each DataNode in the pipeline writes the
                                     packet to its local block file and to an
                                     in-progress checksum file, then ACKs back
                                     up the pipeline once its own write (and
                                     all downstream ACKs) succeed
      │
      ▼
Client receives ACK chain            client only considers the write durable
                                      once ACKs have propagated back through
                                      the full pipeline (all 3 replicas
                                      confirmed) — this is the write's
                                      durability guarantee, analogous to a
                                      quorum ack in other replicated systems,
                                      except here it's "all assigned
                                      replicas," not a tunable quorum
      │
      ▼
Block finalized, NameNode updated    once the block reaches its target size
                                      (or the file is closed), the block is
                                      finalized; the NameNode's namespace
                                      metadata is updated (via edit log), and
                                      DataNodes report the finalized block on
                                      their next block report/heartbeat
```

Confirm the write landed with the placement the diagram describes:

```bash
$ hdfs fsck /data/orders/orders.parquet -files -blocks -locations
/data/orders/orders.parquet 268435456 bytes, 2 block(s):  OK
 0. blk_1073741825_1001 len=134217728 repl=3 [10.0.1.11:9866, 10.0.1.24:9866, 10.0.1.26:9866]
 1. blk_1073741826_1002 len=134217728 repl=3 [10.0.1.12:9866, 10.0.1.23:9866, 10.0.1.27:9866]
Status: HEALTHY
```

And check that the DataNodes chosen actually had room, via the cluster-wide capacity report the NameNode's placement policy consults:

```bash
$ hdfs dfsadmin -report

Configured Capacity: 48000000000000 (43.65 TB)
Present Capacity: 45360000000000 (41.26 TB)
DFS Remaining: 12100000000000 (11.01 TB)
DFS Used: 33260000000000 (30.25 TB)
DFS Used%: 73.32%
Live datanodes (6):

Name: 10.0.1.11:9866 (rack1-dn1)
Hostname: dn1.cluster.local
Decommission Status : Normal
Configured Capacity: 8000000000000 (7.28 TB)
DFS Used: 5820000000000 (5.29 TB)
DFS Remaining: 2020000000000 (1.84 TB)
DFS Used%: 72.75%
...
```

For subsequent reads of this same data, YARN's data-locality scheduling (above) is what decides whether a later job reading this file gets to read one of these 3 local replicas or has to pull it over the network.

---

## Deep Internals

### NameNode: Single Point of Metadata, Historically a Single Point of Failure

The NameNode holds the entire namespace (directory tree, permissions, file→block mapping) in memory for speed — this is precisely why NameNode memory sizing is the classic HDFS capacity-planning exercise: it scales with *number of files and blocks*, not with total data volume. A cluster storing 10PB across a million large files needs far less NameNode memory than one storing 10TB across a billion small files — the "small files problem" is fundamentally a NameNode memory problem, not a disk-space problem.

Durability of the namespace itself (separate from block data durability) comes from an edit log (a write-ahead log of namespace mutations) plus periodic checkpointed snapshots (`fsimage`), historically combined by a Secondary NameNode (a checkpointing helper, not a hot standby, despite the name being widely misread as one) or, in modern deployments, HDFS High Availability (HA) mode with an Active/Standby NameNode pair coordinated via a shared edit log (JournalNodes using a Paxos-like quorum write) and automatic failover via ZooKeeper.

The small-files problem is directly countable — `hdfs dfs -count` reports directories/files/bytes for a path, which is the quickest way to spot a namespace that's about to become a NameNode memory problem:

```bash
$ hdfs dfs -count -h /data/raw_events
           DIR_COUNT   FILE_COUNT       CONTENT_SIZE PATHNAME
                 412      8,340,112            2.1 T /data/raw_events
```

8.3M files averaging ~260KB each against a 128MB default block size — every one of those files (plus its block records) is a separate object pinned in NameNode heap, regardless of how little disk space they actually occupy. Compare against `dfsadmin -report`'s cluster-wide DFS Used% (above): a cluster can show 25% disk utilization and still be minutes from an OOM'd NameNode because of a path like this one.

Rack awareness — which the block placement policy above depends on to put the 2nd/3rd replica on a different rack — is inspectable directly:

```bash
$ hdfs dfsadmin -printTopology

Rack: /rack1
   10.0.1.11:9866 (dn1.cluster.local)
   10.0.1.12:9866 (dn2.cluster.local)
Rack: /rack2
   10.0.1.23:9866 (dn3.cluster.local)
   10.0.1.24:9866 (dn4.cluster.local)
   10.0.1.26:9866 (dn5.cluster.local)
   10.0.1.27:9866 (dn6.cluster.local)
```

If this command instead printed every node under a single `/default-rack`, that's the signature of rack-awareness never having been configured — `net.topology.script.file.name` (or the newer `net.topology.table.file.name`) unset — and the "spread replicas across racks" guarantee silently degrading to "spread replicas across the same rack." Relevant `hdfs-site.xml` keys for this whole subsection:

```xml
<property>
  <name>dfs.replication</name>
  <value>3</value>
</property>
<property>
  <name>dfs.blocksize</name>
  <value>134217728</value> <!-- 128MB -->
</property>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>/data/namenode</value> <!-- fsimage + edit log location -->
</property>
```

Once a cluster is unbalanced (new DataNodes added, or old ones filling unevenly — visible in the per-node `DFS Used%` lines from `dfsadmin -report`), the standard remedy is the balancer, which moves blocks between DataNodes without touching the namespace:

```bash
$ hdfs balancer -threshold 10
```

`-threshold 10` stops once every DataNode's utilization is within 10 percentage points of the cluster average.

### YARN: ResourceManager, NodeManager, ApplicationMaster

```
ResourceManager (cluster-wide)
   │  tracks total cluster capacity, accepts job submissions,
   │  hands each job a container to run its own ApplicationMaster
   │
   ├──▶ NodeManager (per node)          NodeManager (per node)
   │       manages containers on            manages containers on
   │       this node, enforces               this node, enforces
   │       memory/CPU limits per              memory/CPU limits per
   │       container                          container
   │
   └──▶ ApplicationMaster (per job)
           negotiates additional containers from the ResourceManager
           for this specific job's tasks, tracks task progress/failures
           — this per-job AM design is what let YARN generalize beyond
           MapReduce: Spark, Tez, and others just implement their own
           ApplicationMaster instead of MapReduce's
```

This ApplicationMaster-per-job design is YARN's actual innovation over Hadoop 1.x's JobTracker (which was both the cluster resource manager *and* the MapReduce-specific job coordinator in one process, hard-coded to MapReduce only). Separating "generic resource negotiation" (ResourceManager) from "framework-specific job logic" (per-job ApplicationMaster) is what let Spark run on Hadoop clusters at all.

The per-container memory/CPU limits NodeManagers enforce are set in `yarn-site.xml`, cluster-wide, and per-request by the submitting framework:

```xml
<property>
  <name>yarn.nodemanager.resource.memory-mb</name>
  <value>65536</value> <!-- total memory this NodeManager may hand out, 64GB -->
</property>
<property>
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>16384</value> <!-- largest single container the RM will grant, 16GB -->
</property>
<property>
  <name>yarn.nodemanager.resource.cpu-vcores</name>
  <value>16</value>
</property>
```

Live containers and their owning applications are queryable directly against the ResourceManager:

```bash
$ yarn application -list

Application-Id                  Application-Name  Application-Type    User     Queue   State     Final-State  Progress  Tracking-URL
application_1690000000000_0042  orders_etl_spark   SPARK                artagarw  default  RUNNING   UNDEFINED    45%       http://rm:8088/proxy/application_1690000000000_0042/
application_1690000000000_0041  daily_rollup_mr     MAPREDUCE            svc_etl   batch    FINISHED  SUCCEEDED    100%      http://rm:8088/proxy/application_1690000000000_0041/
```

```bash
$ yarn node -list -showDetails

Node-Id             Node-State  Running-Containers  Used-Memory  Avail-Memory  Used-VCores  Avail-VCores
dn1.cluster.local:45454  RUNNING     4                   28672MB      36864MB       6            10
dn3.cluster.local:45454  RUNNING     6                   49152MB      16384MB       9            7
```

This is the live, per-node view of exactly what the ApplicationMaster in the diagram above is negotiating for — each row is one NodeManager reporting its current container occupancy back to the ResourceManager.

### MapReduce's Legacy Status

MapReduce (the original batch execution model: map → shuffle → reduce, with every intermediate stage materialized to disk) is still present and functional in current Hadoop releases, but virtually no new workloads are authored against it directly — Spark's in-memory DAG execution model and Tez's more flexible DAG-of-stages model both run as YARN applications and outperform raw MapReduce for most workloads by avoiding MapReduce's mandatory disk materialization between every map and reduce stage. Understanding MapReduce is mostly valuable now for understanding *why* Spark's execution model (see `compute/spark-on-eks.md` in this KB) was designed the way it was — as a direct response to MapReduce's per-stage disk I/O cost.

MapReduce jobs still run the same way they always have — `hadoop jar` against the bundled examples jar, still shipped in current releases for exactly this kind of legacy verification:

```bash
$ hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar wordcount /data/text/input /data/text/output
```

And legacy MapReduce jobs (as opposed to Spark/Tez applications, which only show up under `yarn application -list`) have their own job-tracking CLI, a holdover from the JobTracker era, now implemented against YARN underneath:

```bash
$ mapred job -list

Total jobs:1
                  JobId  State  StartTime  UserName  Queue  Priority  UsedContainers  RsvdContainers  UsedMem  RsvdMem  NeededMem  AM info
job_1690000000000_0041  RUNNING  1690000012000  svc_etl  batch  NORMAL  24  0  49152M  0M  49152M  http://rm:8088/proxy/application_1690000000000_0041/
```

Note the 1:1 correspondence: `job_1690000000000_0041` here is the same underlying YARN application as `application_1690000000000_0041` in the `yarn application -list` output above — MapReduce's job IDs are just its framework-specific view onto the same YARN ApplicationMaster machinery every other framework uses.

---

## Comparative: vs. S3/Object Storage, vs. Kubernetes Scheduling

**HDFS vs. S3-style object storage**: HDFS's block-replication-plus-locality model assumes compute and storage share the same physical nodes — this is precisely what makes data-locality scheduling possible, and precisely what S3 gives up. S3 is disaggregated: any compute node reads any object over the network, with no locality concept at all, in exchange for elastic, independently-scalable storage that isn't capacity-planned alongside compute nodes. The modern lakehouse pattern (Iceberg/Delta on S3, queried by Spark/Trino/DuckDB) explicitly trades away HDFS's locality-driven network savings for the operational simplicity of not co-managing a compute cluster's disks as your storage tier — acceptable because modern data-center network bandwidth to object storage has grown enough that the locality penalty, while real, is no longer prohibitive for most workloads.

**YARN vs. Kubernetes scheduling**: both are container/resource schedulers with a central scheduler process and per-node agents (YARN: ResourceManager/NodeManager; Kubernetes: kube-scheduler/kubelet), but YARN was purpose-built around data-locality-aware placement for data-processing frameworks, while Kubernetes is a general-purpose container orchestrator with no native concept of "schedule this container near this HDFS block" (locality-aware scheduling on Kubernetes has to be bolted on via node affinity rules or accepted as a non-goal when storage is disaggregated anyway, as it typically is in cloud-native Spark-on-Kubernetes deployments — see `compute/spark-on-eks.md`). The industry direction has been to run Spark on Kubernetes against S3 rather than on YARN against HDFS specifically because once storage is disaggregated, YARN's core differentiator (locality-aware placement) has nothing to optimize for, and Kubernetes's broader ecosystem (not tied to the Hadoop stack) wins on operational grounds.

---

## Key Gotchas

- **Small files silently degrade NameNode capacity, not disk capacity**: a cluster can be nowhere near disk-full and still be effectively out of capacity because NameNode heap is exhausted tracking millions of tiny files/blocks — the fix is consolidating small files (e.g. via compaction into larger files/Parquet row groups), not adding disks.
- **Replication factor is a per-file/per-directory setting, easy to leave at a stale default**: changing the cluster-wide default doesn't retroactively change already-written files' replication factor; auditing actual replication factor across a large existing cluster is a distinct exercise from setting policy for new writes. Bumping an existing path forward has to be done explicitly, and it's a slow, bandwidth-consuming background operation, not instantaneous:

  ```bash
  $ hadoop fs -setrep -w 3 /data/orders
  Replication 3 set: /data/orders/orders.parquet
  Waiting for /data/orders/orders.parquet ... done
  ```

  (`-w` blocks until the new replication level is actually satisfied, rather than just queuing the change — useful for confirming a compliance-driven replication bump actually completed before reporting it done.)
- **Erasure-coded data has zero locality by design** — don't apply EC to hot/frequently-scanned datasets expecting the usual locality-driven read performance; every EC read is a multi-node reconstruction over the network, full stop. Enabling it is a per-directory policy assignment, not a cluster-wide flag, which is exactly why it's easy to accidentally apply (or forget to apply) to the wrong dataset:

  ```bash
  $ hdfs erasurecode -setPolicy -policy RS-6-3-1024k -path /data/cold_archive
  Set erasure coding policy RS-6-3-1024k on /data/cold_archive

  $ hdfs erasurecode -getPolicy -path /data/cold_archive
  RS-6-3-1024k
  ```

  `RS-6-3-1024k` (6 data + 3 parity, 1MB cells) costs ~1.5x storage instead of replication's 3x — but confirm with `-getPolicy` before trusting a directory's read performance profile; a directory with no explicit policy silently falls back to standard 3x replication instead.
- **The "Secondary NameNode" name is actively misleading**: it is a checkpointing helper (merges edit log into fsimage periodically), not a standby NameNode and not a failover target. Confusing it for a passive backup NameNode has led to real production availability gaps — HDFS HA mode (Active/Standby with JournalNodes + ZooKeeper failover) is the actual answer to NameNode failover, and is a distinct deployment mode you must explicitly configure.
- **Rack awareness must be explicitly configured (`net.topology.script.file.name` or equivalent) or block placement silently loses its rack-fault-tolerance guarantee** — without correct rack topology info, the NameNode can't actually place the 2nd/3rd replicas on a different rack, even though the policy assumes it can. Confirm it's actually wired up rather than assuming:

  ```bash
  $ hdfs dfsadmin -printTopology | grep -c "^Rack:"
  1
  ```

  A count of `1` here (everything under one rack, or literally `/default-rack`) on a multi-rack cluster is the tell that `core-site.xml`'s `net.topology.script.file.name` was never pointed at a real topology script — compare against the multi-rack `-printTopology` output shown earlier in Deep Internals, where the same command reports 2 distinct racks.
- **Block size is a real capacity-planning parameter, not a minor tuning knob**: too small (relative to Hadoop 1.x-era 64MB defaults on modern petabyte-scale data) multiplies NameNode metadata load and task-scheduling overhead; too large wastes space for small files and increases the granularity (and thus cost) of a single node/task failure's retry.
- **YARN container memory limits are hard kills, not soft warnings**: a container that exceeds its requested memory allocation is killed by the NodeManager outright (to protect the node from being taken down by one runaway container) — under-requesting memory for a Spark executor running as a YARN container is a common cause of mysterious executor-lost errors that look like unrelated failures downstream.

---

*Grounded against hadoop.apache.org release notes and documentation as of September 2026. Current release line: Hadoop 3.5.0 (GA April 2026) is the latest stable; Hadoop 3.4.x continues receiving patch releases (3.4.3, February 2026) for users not yet on 3.5. MapReduce, HDFS, and YARN are all still actively maintained in-tree; adoption of HDFS itself as a primary storage layer for new workloads has been declining in favor of S3-backed lakehouse architectures, while YARN's design ideas remain influential even where YARN itself has been replaced by Kubernetes. Re-verify exact patch version before citing externally.*
