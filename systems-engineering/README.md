# Systems Engineering

Backend/distributed-systems and data-platform reference. Each file: 30-second intuition → worked examples → deep internals → gotchas. Systems-mode docs (whole technology/framework) additionally map the topic onto a CPU/memory/disk/network/GPU resource-layer table and name the one signature mechanism that makes it behave the way it does.

## Distributed Systems

| File | Topics |
|---|---|
| [consensus-raft-vs-paxos.md](/systems-engineering/distributed-systems/consensus-raft-vs-paxos.md) | Raft leader election/log replication/terms, Basic Paxos proposer-acceptor-learner, Multi-Paxos, split-vote worked example, Flexible Paxos, EPaxos |
| [consistent-hashing.md](/systems-engineering/distributed-systems/consistent-hashing.md) | Hash ring, ~1/N remapping vs `hash(key) % N`, virtual nodes for load balance, bounded-load/rendezvous hashing, vs Redis Cluster's fixed hash slots |
| [distributed-transactions.md](/systems-engineering/distributed-systems/distributed-transactions.md) | 2PC prepare/commit protocol, the blocking problem, 3PC's partition limitation, Saga pattern (orchestration vs choreography), compensating transactions |
| [consistency-models.md](/systems-engineering/distributed-systems/consistency-models.md) | CAP vs PACELC, linearizability → sequential → causal → eventual consistency, DynamoDB/Cassandra/Spanner/etcd/DNS classification |
| [clocks-and-ordering.md](/systems-engineering/distributed-systems/clocks-and-ordering.md) | Clock skew/drift, Lamport timestamps, vector clocks (concurrency detection), Hybrid Logical Clocks, Spanner TrueTime/commit-wait |
| [fallacies-of-distributed-computing.md](/systems-engineering/distributed-systems/fallacies-of-distributed-computing.md) | The 8 Deutsch/Gosling fallacies, each mapped to a remedy + tradeoff + cross-referenced doc that implements it |

## Data Formats

| File | Topics |
|---|---|
| [parquet.md](/systems-engineering/data-formats/parquet.md) | Row groups/column chunks/pages, dictionary/RLE/delta encoding, page index, bloom filters, schema evolution |
| [apache-iceberg.md](/systems-engineering/data-formats/apache-iceberg.md) | Manifest tree, snapshot isolation, schema/partition evolution, CoW vs MoR, catalogs |
| [delta-lake.md](/systems-engineering/data-formats/delta-lake.md) | `_delta_log/`, optimistic concurrency, DML internals, CDF, Deletion Vectors, Vacuum |
| [z-ordering.md](/systems-engineering/data-formats/z-ordering.md) | Morton curve, bit interleaving, file/row-group pruning, when Z wins |
| [liquid-clustering.md](/systems-engineering/data-formats/liquid-clustering.md) | Hilbert curve, incremental re-clustering, CLUSTER BY vs ZORDER BY, clustering key selection |

## Data Architecture

| File | Topics |
|---|---|
| [data-platform-architectures.md](/systems-engineering/data-architecture/data-platform-architectures.md) | Medallion, Lambda vs Kappa (2026 hybrid consensus), Data Mesh (federated governance, 52% vs 38% success rate), Data Fabric, Warehouse vs Lake vs Lakehouse, when to use what |

## Data Modeling

| File | Topics |
|---|---|
| [star-schema-dimensional-modeling.md](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md) | Kimball fact/dimension split, star vs snowflake, grain, SCD Type 0-2 worked example, OBT vs star on columnar warehouses, Data Vault comparison |

## Data Structures

| File | Topics |
|---|---|
| [lsm-trees.md](/systems-engineering/data-structures/lsm-trees.md) | Write/read/space amplification, compaction strategies, SSTable internals, tombstones, MVCC |
| [rocksdb.md](/systems-engineering/data-structures/rocksdb.md) | Column families, write stalls, compaction tuning, block cache, Flink state backend |
| [roaring-bitmaps.md](/systems-engineering/data-structures/roaring-bitmaps.md) | Array/bitmap/run containers, set ops, RBAC/ABAC bitmap filtering, ReBAC hybrid caching |

## Authorization

| File | Topics |
|---|---|
| [zanzibar.md](/systems-engineering/authz/zanzibar.md) | Relationship tuples, userset rewrites, zookies/new-enemy problem, Leopard index |
| [spicedb.md](/systems-engineering/authz/spicedb.md) | Schema DSL, CheckPermission/LookupResources, ZedTokens, vs OpenFGA/Ory Keto |

## Caching

| File | Topics |
|---|---|
| [redis-vs-memcached.md](/systems-engineering/caching/redis-vs-memcached.md) | Event loop + I/O threads vs slab allocator, RDB/AOF persistence, Redis license fork (Valkey), cache vs coordination store |

## Coordination

| File | Topics |
|---|---|
| [etcd.md](/systems-engineering/coordination/etcd.md) | Raft log as source of truth, MVCC revisions, watch mechanism, boltdb, disk-fsync sensitivity, vs ZooKeeper/Consul |

## Storage

| File | Topics |
|---|---|
| [hadoop.md](/systems-engineering/storage/hadoop.md) | HDFS block replication, data-locality scheduling, NameNode/YARN internals, erasure coding, vs S3/Kubernetes |
| [postgres.md](/systems-engineering/storage/postgres.md) | In-heap MVCC (xmin/xmax, no undo log), WAL/checkpointing, autovacuum & bloat/wraparound, B-tree vs clustering, planner cost model, vs LSM-trees/etcd MVCC |
| [s3.md](/systems-engineering/storage/s3.md) | Strong read-after-write consistency (Dec 2020), flat keyspace (no atomic rename), multipart upload internals, key-prefix request-rate partitioning, storage classes/lifecycle, vs HDFS/GCS/Azure Blob/MinIO |
| [cassandra.md](/systems-engineering/storage/cassandra.md) | Masterless ring + gossip, tunable consistency (R+W>N quorum math), partition/clustering key query-first modeling, SAI/vector search/UCS/Accord (5.0-6.0), vs DynamoDB/etcd |

## Streaming

| File | Topics |
|---|---|
| [kafka.md](/systems-engineering/streaming/kafka.md) | Zero-copy sendfile, resource-layer map, KRaft, log segments, tiered storage, vs Pulsar/Redpanda |
| [debezium.md](/systems-engineering/streaming/debezium.md) | Log-based CDC via WAL/binlog, Postgres replication slots & logical decoding, incremental snapshotting, outbox pattern, SMTs, vs polling/JDBC source |

## Query Engines

| File | Topics |
|---|---|
| [duckdb.md](/systems-engineering/query-engines/duckdb.md) | Resource-layer map, vectorized+morsel-driven execution (no shuffle), HTTPFS pushdown, Iceberg/Delta writes, vs Spark/Polars |
| [opensearch.md](/systems-engineering/query-engines/opensearch.md) | Lucene internals, BM25, k-NN plugin, hybrid search, ILM, sharding |
| [clickhouse.md](/systems-engineering/query-engines/clickhouse.md) | MergeTree parts/sparse index, columnar codecs, materialized views, replication, Keeper |
| [trino.md](/systems-engineering/query-engines/trino.md) | Connector SPI/federation, in-memory no-durability exchange, fault-tolerant execution, CBO/statistics, dynamic filtering, vs Spark/DuckDB |

## Orchestration

| File | Topics |
|---|---|
| [airflow-internals.md](/systems-engineering/orchestration/airflow-internals.md) | Scheduler loop, all executors, KubernetesExecutor, XCom, deferrable operators, HA |
| [argo-workflows.md](/systems-engineering/orchestration/argo-workflows.md) | CRD types, template types, artifact passing, emissary executor, DAG vs Steps |
| [argo-rollouts.md](/systems-engineering/orchestration/argo-rollouts.md) | Blue-green, canary steps, AnalysisTemplate, Istio weight patching, Experiment CRD |
| [argo-events.md](/systems-engineering/orchestration/argo-events.md) | EventSource/EventBus/Sensor, NATS backbone, multi-source triggers, payload extraction |
| [keda.md](/systems-engineering/orchestration/keda.md) | Scale-to-zero, ScaledObject/ScaledJob, Kafka/SQS/Prometheus scalers, IRSA auth |
| [karpenter.md](/systems-engineering/orchestration/karpenter.md) | NodePool/EC2NodeClass, vs Cluster Autoscaler, consolidation, KEDA+Karpenter pattern |
| [cluster-schedulers.md](/systems-engineering/orchestration/cluster-schedulers.md) | kube-scheduler filter/score plugin framework, requests vs limits, taints/topology spread, gang-scheduling gap (Kueue/Volcano/coscheduling), vs YARN, Mesos retirement (Oct 2025) |

## Compute

| File | Topics |
|---|---|
| [spark.md](/systems-engineering/compute/spark.md) | Catalyst/Tungsten, whole-stage codegen, shuffle mechanics, AQE, native execution (Photon/Gluten/Comet) |
| [spark-on-eks.md](/systems-engineering/compute/spark-on-eks.md) | Operator vs submit, dynamic allocation, ESS, S3A tuning, AQE, gang scheduling, spot |
| [flink.md](/systems-engineering/compute/flink.md) | Checkpoint barriers (Chandy-Lamport snapshots), watermarks/event-time/allowed lateness, windowing, two-phase-commit sinks, ForSt disaggregated state, vs Spark Structured Streaming |
| [spark-structured-streaming.md](/systems-engineering/compute/spark-structured-streaming.md) | Micro-batch as repeated batch query, Spark 4.1 Real-Time Mode/transformWithState, watermarks, RocksDB state store, checkpointing/exactly-once, vs Flink/Kafka Streams/ksqlDB |

## Networking

| File | Topics |
|---|---|
| [networking-layers.md](/systems-engineering/networking/networking-layers.md) | TCP/IP model (link/network/transport/application), encapsulation with header sizes, TCP three-way handshake, flow control, slow start/AIMD congestion control, TCP vs UDP, HTTP/1.1 vs HTTP/2 vs HTTP/3-QUIC head-of-line blocking, TLS 1.3 handshake, DNS resolution |
| [grpc.md](/systems-engineering/networking/grpc.md) | HTTP/2 stream multiplexing + Protobuf binary wire format, unary/streaming RPC types, protoc codegen + grpcurl walkthrough, deadline/cancellation propagation, status codes, Protobuf Editions, gRPC-Web/HTTP-3, vs REST/JSON and GraphQL |

## Reliability

| File | Topics |
|---|---|
| [multi-az-dr-and-failure-patterns.md](/systems-engineering/reliability/multi-az-dr-and-failure-patterns.md) | AWS DR 4-strategy taxonomy (RTO/RPO), static stability vs bimodal behavior, timeouts/retries/backoff+jitter, circuit breaker/bulkhead, hedged/tied requests (Tail at Scale), cascading failure — quotes primary sources throughout |

## Performance

| File | Topics |
|---|---|
| [resource-layer-crux-guide.md](/systems-engineering/performance/resource-layer-crux-guide.md) | Cross-cutting index (OS/CPU/memory/file systems/disks/network/cloud): syscall/context-switch tax, event loops vs io_uring, cache locality, mmap/TLB, page cache, LSM sequential-write trick, Firecracker microVMs — cross-referenced to every doc that demonstrates each |
