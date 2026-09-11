# Vector Databases

## 30-Second Intuition

You have millions of embedding vectors and need to find the k most similar ones to a query vector in milliseconds. Brute-force is O(n*d) — too slow. Vector databases build approximate nearest neighbor (ANN) indexes that trade a small accuracy loss for 100-1000x faster search. The index type, distance metric, and metadata filtering strategy are the core design decisions.

---

## The Core Problem: ANN at Scale

Exact nearest neighbor search:
```
For each of N vectors, compute distance to query → sort → return top-k
Cost: O(N * d) per query, where d = dimensions
At 1M vectors, 768d, float32: 1M * 768 * 4 bytes = ~3 GB to read per query
At 10ms budget: physically impossible
```

Approximate nearest neighbor (ANN):
- Accept ε% recall loss (e.g., 95% of true nearest neighbors returned)
- Use index structures to skip most of the search space
- Target: 1-10ms p99, 99%+ recall

**Recall@k** = fraction of true top-k results that appear in returned results. This is the primary quality metric. 95% recall@10 means you get 9-10 of the true 10 nearest neighbors.

---

## Index Types: Intuition

### Flat (Brute Force)
```
No index. Scan all vectors.
Recall: 100% (exact)
Speed: O(N*d), useless at scale
Use when: N < 50K, or as ground truth for evaluating other indexes
```

### IVF (Inverted File Index)
```
Offline: cluster vectors into C centroids (k-means)
         Each vector assigned to nearest centroid
         Build inverted list: centroid → [vector_ids]

Query:   Find nearest P centroids to query
         Search only those P inverted lists
         Return top-k from candidates

Recall: ~95% with P=nprobe=10, C=1024 cells
Speed: O(P * (N/C) * d) ≈ 10x-100x faster than flat

Key param: nprobe — how many cells to search at query time
  nprobe=1  → fast, low recall
  nprobe=64 → slow, high recall
```

Problem: centroid approximation error. If your query is near a cell boundary, its nearest neighbors may be in the adjacent cell you didn't probe.

### HNSW (Hierarchical Navigable Small World)
```
Graph-based. Nodes = vectors. Edges = similar vectors.
Hierarchical layers: top layers = long-range shortcuts,
                     bottom layer = fine-grained neighbors

Query: enter at top layer → greedy walk toward query →
       descend layers → exhaustive at bottom layer

Recall: 98-99% at good ef settings
Speed: O(log N) with high practical constants
```

Best for low-latency. No centroid approximation error. Expensive memory. See `hnsw.md` for deep internals.

### LSH (Locality-Sensitive Hashing)
```
Hash vectors into buckets such that similar vectors
collide with high probability.

Query: hash query → look up same buckets → compare candidates

Recall: 80-90%, hard to tune
Speed: O(1) bucket lookup, then small linear scan

Largely superseded by HNSW for dense vectors.
Still useful for some cosine similarity problems.
```

### ScaNN (Google)
```
Two-phase: anisotropic quantization + re-scoring
  Phase 1: quantize + fast ANN with PQ
  Phase 2: exact re-score top candidates with original vectors

Recall: 98%+
Speed: state-of-the-art for billion-scale
Availability: mostly used internally by Google; available as open source
```

---

## Major Players: Honest Trade-offs

### Pinecone
- **Managed only** — no self-hosting option
- Fully serverless-first as of 2026 (legacy pod-based deployments phased out); metered pricing on read units/write units/storage rather than idle compute
- Added **Dedicated Read Nodes (DRN)** tier for sustained high-QPS predictable workloads alongside on-demand serverless
- Namespace-based multi-tenancy
- Proprietary index (reportedly HNSW-based + proprietary optimizations)
- **Gotcha**: pricing can surprise at scale; no SQL-style metadata filter expressiveness
- Good for: startups, quick prototypes, teams without infra budget

### Weaviate
- Open source + managed cloud option
- Built-in hybrid search (BM25 + dense); shipped "Hybrid Search 2.0" with learned fusion replacing static alpha-weighting
- Weaviate Cloud added a native Embedding Service and a Query Agent (natural-language exploration in console) in early 2026
- GraphQL API + REST + gRPC
- Module system: plug in any embedding model, reranker, generative LLM
- Good horizontal scaling story
- **Gotcha**: schema management can be verbose; resource-hungry
- Good for: teams wanting full-featured open source with hybrid search built in

### Qdrant
- Open source + managed cloud
- Rust-based, very fast and memory-efficient
- Rich payload filtering with HNSW index (not post-filter — filtered HNSW)
- gRPC + REST APIs
- Sparse vector support for hybrid search
- Added GPU-accelerated HNSW index building (up to ~10x faster builds), expanded quantization (1.5-bit, 2-bit, asymmetric), and incremental HNSW for upserts
- **Gotcha**: relatively newer, smaller ecosystem
- Good for: performance-critical self-hosted deployments

### Milvus
- Open source, designed for billion-scale
- Multiple index types (HNSW, IVF_FLAT, IVF_PQ, DiskANN), plus newer variants (HNSW_SQ, HNSW_PQ, HNSW_PRQ, IVF_RABITQ, SCANN) and an AUTOINDEX mode that auto-selects index/params
- Kubernetes-native, complex architecture (etcd, MinIO, Pulsar)
- **Gotcha**: operationally complex; overkill for < 10M vectors
- Good for: large-scale enterprise deployments on-prem

### pgvector
- PostgreSQL extension — store vectors in your existing Postgres
- ivfflat and hnsw index types (still no native DiskANN as of pgvector 0.8.x in 2026); 0.8.x added iterative index scans and halfvec/sparsevec/bit types
- For scale beyond what fits in RAM, the **pgvectorscale** extension (Timescale) adds a StreamingDiskANN disk-resident index on top of pgvector — common guidance in 2026 is pgvector alone for < 10M vectors, pgvectorscale for hundreds of millions
- Full SQL: join vectors with regular tables, ACID guarantees
- **Gotcha**: recall and throughput at scale (> 5M vectors) lags behind dedicated VDBs unless paired with pgvectorscale
- Good for: you already use Postgres, data volume is moderate, want simplicity

### Newer entrants worth knowing (2026)
- **Turbopuffer**: serverless, object-storage-native, BM25 + vector hybrid search; added full-text search in 2026; used by Cursor, Notion, Linear
- **AWS S3 Vectors**: reached general availability in early 2026, storage-first vector index directly on S3 (up to 2B vectors/index, 10,000 indexes/bucket); positioned as a cheap complement to dedicated VDBs, not a full replacement
- **LanceDB**: embedded/serverless, S3-backed, popular for multimodal + lakehouse-style vector workloads

### Chroma
- Open source, Python-first, embeds locally
- Excellent for development and small-scale projects
- Not production-grade at scale; no horizontal scaling
- **Gotcha**: don't use in production for anything serious
- Good for: notebooks, prototypes, local RAG development

### OpenSearch k-NN
- HNSW + FAISS + NMSLIB backends
- Integrates with existing OpenSearch/Elasticsearch infrastructure
- Full-text + vector hybrid search in one system
- **Gotcha**: resource overhead of running JVM + HNSW in heap; tuning is complex
- Good for: teams already on OpenSearch/Elasticsearch

---

## Distance Metrics: When Each Is Appropriate

### Cosine Similarity
```
cosine_sim(a, b) = (a · b) / (|a| * |b|)
Range: [-1, 1], higher = more similar

Use when:
- Vectors have varying magnitudes (documents of different lengths)
- You care about direction, not magnitude
- Most text embedding models → this is the right default
```

### Dot Product
```
dot(a, b) = a · b
Range: unbounded

Use when:
- Vectors are pre-normalized (then dot product == cosine similarity)
- You want magnitude to matter (longer = more important)
- Some recommendation systems where "more" matters
```

### L2 (Euclidean) Distance
```
L2(a, b) = sqrt(sum((a_i - b_i)^2))
Range: [0, ∞), lower = more similar

Use when:
- Vectors live in actual geometric space (coordinate data)
- You want to penalize outliers more strongly
- Image features in some CV tasks

Gotcha: for normalized vectors, L2 and cosine are equivalent:
  L2(a,b)^2 = 2 - 2*cosine_sim(a,b)  when |a|=|b|=1
```

**Rule of thumb**: use cosine for text embeddings. Most embedding models are trained with cosine as the similarity function.

---

## Metadata Filtering: The Hard Problem

Every real RAG system needs filtering: "find similar documents WHERE tenant_id='acme' AND date > 2024-01-01".

### Post-Filter
```
1. ANN search: retrieve top-1000 candidates
2. Apply metadata filter
3. Return top-k from filtered set

Problem: if filter is selective (1% of corpus matches),
         you waste 99% of ANN work.
         If all 1000 results fail the filter, you return 0.
```

### Pre-Filter
```
1. Build filtered list of valid IDs from metadata index
2. Run ANN search restricted to those IDs

Problem: HNSW struggles with small filtered sets
         (can't navigate graph if most edges lead to filtered-out nodes)
         Effective recall degrades.
```

### Filtered HNSW (Qdrant's approach)
```
During HNSW traversal, skip nodes that don't match filter
but still allow traversal through them.
Maintains recall even with selective filters.
This is state-of-the-art for production systems.
```

### Segmented / Partitioned Indexes
```
Build separate HNSW index per tenant / per time partition.
Route query to correct index.
Simple, effective, scales well for known filter categories.
Common in multi-tenant SaaS.
```

**Decision rule:**
- Filter selectivity > 20% → post-filter is fine
- Filter selectivity 1-20% → use segmented indexes or filtered HNSW (Qdrant)
- Filter selectivity < 1% → build a partitioned index per filter value

---

## Hybrid Search: Dense + Sparse

Pure dense retrieval (vectors) misses keyword-heavy queries: exact product names, version numbers, error codes, proper nouns. BM25 nails these but misses paraphrase/synonym queries.

Hybrid = combine both signals.

### Reciprocal Rank Fusion (RRF)
```python
def rrf(dense_results, sparse_results, k=60):
    scores = {}
    for rank, doc_id in enumerate(dense_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    for rank, doc_id in enumerate(sparse_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

RRF is robust, parameter-free, and typically outperforms linear combination of scores.

### Linear Score Combination
```
hybrid_score = α * dense_score + (1 - α) * sparse_score
# α is tunable; 0.7 is a common starting point
```

Requires score normalization (dense and sparse scores live on different scales).

**When hybrid search wins over pure dense:**
- Acronyms and product names ("LSTM", "GPT-4", "HikariCP")
- Error codes and log patterns
- Names (people, companies)
- Any domain with specific terminology not well-represented in embedding training data

---

## Namespaces / Collections / Multi-Tenancy

| System | Primitive | Notes |
|--------|-----------|-------|
| Pinecone | Namespace | Logical partition within an index; separate HNSW per namespace |
| Weaviate | Class (Collection) | Separate schema + index per class |
| Qdrant | Collection | Separate HNSW per collection; payload-based filtering within |
| Milvus | Collection | Full schema isolation |
| pgvector | Table | Just use separate tables or a tenant_id column |

**Tenant isolation patterns:**
1. **Separate collection per tenant**: full isolation, operationally complex at 1000s of tenants
2. **Shared collection + tenant_id filter**: simple, but filtered search degrades with many tenants
3. **Namespace per tenant**: good middle ground (Pinecone model)

---

## Persistence, WAL, Replication

- Most dedicated VDBs (Qdrant, Milvus, Weaviate) use WAL + periodic snapshotting
- HNSW graphs are expensive to rebuild from scratch on restart → load from snapshot
- **Replication**: Qdrant supports leader + follower replicas; Milvus uses Pulsar for log replication
- **Durability trade-off**: async WAL flush = faster writes, risk of data loss on crash; sync = slower
- pgvector inherits PostgreSQL's WAL — battle-tested durability

---

## Batch vs Real-Time Indexing

### Batch (Offline) Indexing
```
- Build HNSW graph once on full dataset
- Better quality (sees full neighbor distribution)
- Can tune M, ef_construction without query pressure
- Not queryable until complete
- Good for: initial load, periodic re-indexing
```

### Real-Time Indexing
```
- Insert vectors as they arrive
- HNSW supports online insert natively
- Quality may degrade if insertion order is adversarial
- Deletes are expensive (see hnsw.md)
- Good for: user-generated content, event streams
```

**Hybrid pattern in production**: batch build nightly, stream inserts to a small "delta" index, merge periodically.

---

## pgvector: When It's Good Enough vs When You Need a Dedicated VDB

### ivfflat index
```sql
CREATE INDEX ON items USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- At query time:
SET ivfflat.probes = 10;  -- tune for recall vs speed
SELECT * FROM items ORDER BY embedding <=> query_embedding LIMIT 10;
```

- `lists` = number of clusters (analog to IVF's C). Rule: `sqrt(N)` to `N/1000`
- `probes` = how many cells to search (analog to nprobe)
- Recall at `lists=100, probes=10`: ~95% for many workloads

### hnsw index
```sql
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

SELECT * FROM items ORDER BY embedding <=> query_embedding LIMIT 10;
```

- Better recall than ivfflat at similar speed
- Higher memory usage
- No equivalent to nprobe — ef at search time is set via `SET hnsw.ef_search = 40`
- pgvector 0.8.x added `hnsw.iterative_scan` (and `ivfflat.iterative_scan`) to improve recall when combined with restrictive metadata filters, partially addressing the pre-filter/post-filter problem described below

### When pgvector is good enough:
- N < 5M vectors
- Query throughput < 100 QPS
- Need ACID joins between vectors and relational data
- Team doesn't want to operate another system
- Recall > 95% achievable with hnsw index

### When you need a dedicated VDB:
- N > 10M vectors
- Query throughput > 1000 QPS
- Need < 5ms p99 latency
- Complex metadata filtering (especially with high selectivity)
- Multi-tenant isolation with thousands of tenants

---

## Concrete Example: Semantic Doc Search

**Scenario**: 2M internal documents, 50 QPS search, metadata filter by department (20 departments), 95% recall target, < 20ms p99.

**Decision tree:**
```
N = 2M  → pgvector might work, dedicated VDB safer
Filtering = department (low cardinality) → segment by department
Recall target = 95% → HNSW over IVF (less parameter tuning)
Latency = 20ms → HNSW at 10ms leaves margin for network + reranking
```

**Recommended setup:**
```
System: Qdrant (filtered HNSW, good Rust performance)
Index: HNSW, M=16, ef_construction=100
Query ef: 50 (tune until recall@10 ≥ 95% on holdout set)
Filtering: department as payload filter (Qdrant handles filtered HNSW)
Embedding model: BGE-M3 or Qwen3-Embedding (2026 open-source quality leaders; BGE-large-en-v1.5 also still workable, 1024d)
Hybrid search: add BM25 for exact term matching, RRF fusion

For 2M * 1024d * 4 bytes = 8GB raw vectors
Plus HNSW graph overhead (M=16): ~8GB additional
Total RAM: ~16-20GB → fits in a single instance
```

---

## Key Gotchas

- **Don't confuse recall and precision**: ANN recall measures "did I get the true nearest neighbors?" Downstream retrieval quality depends on whether your chunks were good in the first place.
- **ef_search at query time beats ef_construction at index time**: build with high ef_construction once, tune ef_search per query for latency/recall trade-off.
- **Deletions degrade HNSW recall**: tombstoned nodes pollute graph traversal. Schedule periodic re-indexing.
- **Metadata cardinality matters for filtering**: 1000s of unique filter values → consider segmented indexes.
- **Score thresholds are model-specific**: a 0.8 cosine similarity with ada-002 means something different than with MiniLM. Never hardcode thresholds across model changes.
- **Cold start problem**: HNSW indexes loaded from disk have warm-up latency on first queries. Use readiness probes.
- **Quantization and recall**: PQ reduces memory 4-8x but can drop recall by 2-5%. Always benchmark on your data.
