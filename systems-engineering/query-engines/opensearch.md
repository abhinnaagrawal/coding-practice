# OpenSearch — Deep Technical Reference

## 30-Second Intuition

OpenSearch is a **distributed search and analytics engine** built on Lucene. Each shard is a full Lucene index: an inverted index for full-text search (BM25 scoring) plus doc values for aggregations. Starting with v2.x, it doubles as a **vector database** via k-NN plugin — enabling hybrid search that combines semantic similarity (HNSW/Faiss embeddings) with keyword relevance in a single query.

**When it clicks**: "I need sub-second search across millions of documents with relevance ranking, faceted filtering, and optionally semantic similarity — in one system."

---

## Architecture Overview

```
Client
  │  REST/HTTP JSON
  ▼
Coordinator Node (any node can route)
  │  scatter/gather
  ├──────────────────────────────────────────┐
  ▼                                          ▼
Data Node 1                            Data Node 2
┌────────────────────┐            ┌────────────────────┐
│  Shard 0 (primary) │            │  Shard 1 (primary) │
│  [Lucene Index]    │            │  [Lucene Index]     │
│  Shard 2 (replica) │            │  Shard 0 (replica) │
└────────────────────┘            └────────────────────┘

Dedicated Master Node (cluster state, not data)
```

---

## Core Search Engine

### Lucene Under the Hood

Every **shard** is a standalone **Lucene index**. OpenSearch is primarily a distributed coordinator that:
- Routes writes to the correct primary shard (by `_routing`, default: `hash(doc_id) % num_shards`)
- Scatter-gathers search requests across shards
- Merges results (sort by `_score`, pagination)

**Lucene segment**: an immutable mini-index containing:
- **Inverted index**: `term → [docID, docID, ...]` with term frequency and position info
- **Doc values**: columnar forward index `docID → field_value` (used for sorting, aggregations, scripts)
- **Stored fields**: compressed original field values for retrieval (`_source`)
- **Norms**: per-field length normalization factors for BM25
- **Point values**: BKD tree for numeric/geo range queries
- **Vector index** (k-NN): HNSW graph structure per segment

```
Lucene Segment:
  inverted_index: {"search": [3, 7, 42], "engine": [1, 3, 99], ...}
  doc_values:     {3: {date: 1704067200, category: "tech"}, 7: {...}, ...}
  stored_fields:  {3: {_source: "{...full JSON...}"}, ...}
  vector_index:   {3: [0.1, 0.4, ...768 floats], ...}  ← HNSW graph
```

### Index → Shard → Replica Relationship

```
Index "products" (logical)
  Primary Shard 0 → Lucene Index on Node A
  Primary Shard 1 → Lucene Index on Node B
  Primary Shard 2 → Lucene Index on Node C
  Replica Shard 0 → Copy of Primary 0 on Node B
  Replica Shard 1 → Copy of Primary 1 on Node C
  Replica Shard 2 → Copy of Primary 2 on Node A
```

Replicas serve reads (load balancing) and provide failover. Primary handles all writes.

---

## Write Path

```
Client → POST /products/_doc/123 {body}

1. Coordinator routes to Primary Shard (hash of doc id)
2. Primary:
   a. Write to Translog (WAL) — durable on disk immediately
   b. Write to In-memory Index Buffer
   c. Forward to replica shards (parallel)
3. Replica acknowledges → primary acknowledges to client

4. [Background: refresh every 1s by default]
   In-memory buffer → new Lucene segment (searchable)

5. [Background: flush every 30min or translog size threshold]
   Segments fsync'd to disk, translog cleared

6. [Background: merge]
   Small segments merged into larger ones (fewer segments = faster search)
```

### The Refresh Problem

After indexing a document, it is **not searchable** until the next refresh creates a new segment from the in-memory buffer. Default refresh interval: **1 second** (near-real-time, not real-time).

```json
PUT /products/_settings
{
  "index.refresh_interval": "30s"   // Reduce for bulk indexing (less overhead)
}

// Force immediate refresh (for testing):
POST /products/_refresh

// Real-time GET by ID (reads from translog, bypasses refresh):
GET /products/_doc/123?realtime=true   // Always returns current version
```

**Performance implication**: frequent refreshes create many small segments, causing segment merge pressure. For bulk loads, set `refresh_interval: -1`, bulk load, then reset to `1s`.

### Segment Merges

Merges are expensive (CPU + I/O) but necessary: fewer segments = faster searches (fewer segment-level inverted index lookups to merge at query time). Lucene uses a tiered merge policy by default.

```json
PUT /products/_settings
{
  "index.merge.policy.max_merge_at_once": 10,
  "index.merge.scheduler.max_thread_count": 1  // Limit merge I/O on hot nodes
}
```

---

## BM25 Scoring

BM25 (Best Match 25) is the default relevance scoring algorithm, replacing classic TF-IDF.

### The Formula
```
BM25(q, d) = Σ IDF(t) × [TF(t,d) × (k1 + 1)] / [TF(t,d) + k1 × (1 - b + b × |d|/avgdl)]
```

**Where**:
- `TF(t,d)`: term frequency of term t in document d
- `IDF(t)`: inverse document frequency — rare terms get higher weight: `log(1 + (N - df + 0.5) / (df + 0.5))`
- `|d|`: document length (in terms)
- `avgdl`: average document length in corpus
- `k1` (default 1.2): term frequency saturation. High k1 = TF matters more. Low k1 → quickly saturates.
- `b` (default 0.75): length normalization. b=0 disables length norm; b=1 full normalization.

**Intuition**: BM25 fixes TF-IDF's problem where a term appearing 1000× gets 1000× score. BM25 **saturates** — after some frequency, additional occurrences barely increase score. The `k1` parameter controls saturation speed.

```json
PUT /products
{
  "settings": {
    "similarity": {
      "my_bm25": {
        "type": "BM25",
        "k1": 1.5,   // Higher: reward TF more, useful for longer docs
        "b": 0.5     // Lower: less length normalization (for structured data)
      }
    }
  },
  "mappings": {
    "properties": {
      "description": { "type": "text", "similarity": "my_bm25" }
    }
  }
}
```

---

## Mappings: The Schema

### Dynamic vs Explicit

Dynamic mapping: OpenSearch infers types on first document. **Dangerous in production** — a stray `"price": "free"` when you expected `"price": 1.99` corrupts the numeric type.

```json
PUT /products/_mapping
{
  "dynamic": "strict",   // "strict": error on unknown fields
                         // "false": ignore unknown fields
                         // true (default): auto-map
  "properties": {
    "title": { "type": "text" },
    "price": { "type": "float" },
    "category": { "type": "keyword" },
    "created_at": { "type": "date", "format": "strict_date_optional_time" }
  }
}
```

### The `keyword` vs `text` Gotcha

- **`text`**: tokenized, analyzed, stored in inverted index. Used for full-text search. `"Hello World"` → tokens `["hello", "world"]`. Cannot sort/aggregate on text fields (use `keyword` subfield).
- **`keyword`**: not analyzed, stored as-is. Used for exact match, sorting, aggregations (terms aggregation), filtering. `"Hello World"` stored as `"Hello World"`.

**Classic trap**: mapping a status field as `text` then trying to do `terms` aggregation on it — you get tokenized garbage (`"active_user"` → `["active", "user"]`).

```json
// Pattern: dual mapping for search + aggregation
"category": {
  "type": "text",           // Analyzed for full-text search
  "fields": {
    "raw": { "type": "keyword" }  // Exact for aggregations
  }
}

// Query text field:   "match": { "category": "electronics" }
// Aggregate keyword:  "terms": { "field": "category.raw" }
```

### Important Field Types

| Type | Use case |
|------|----------|
| `text` | Full-text search |
| `keyword` | Exact match, sort, aggregation |
| `integer/float/double` | Numeric range, sort |
| `date` | Date range, date histogram agg |
| `boolean` | Filter |
| `nested` | Array of objects where you need object-scoped queries |
| `geo_point` | Geographic queries |
| `knn_vector` | Dense vector for semantic similarity |

---

## Query DSL: bool/must/should/filter

```
bool query
  ├── must:    [queries]   → ALL must match + contributes to score
  ├── should:  [queries]   → AT LEAST ONE should match (OR), boosts score
  ├── filter:  [queries]   → ALL must match, NO score contribution
  └── must_not:[queries]   → NONE must match, NO score contribution
```

**Filter context vs Query context** — this is the most important performance concept:

- **Query context** (must/should): computes relevance score. Every document goes through BM25 scoring. **Slower** for large result sets.
- **Filter context** (filter/must_not): binary yes/no. No score computation. Results are **cached** in the filter cache (bitset per segment). **Fast** for structured filters.

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "running shoes" } }  // ← BM25 scored, expensive
      ],
      "filter": [                                    // ← cached, free after first use
        { "term": { "category.raw": "footwear" } },
        { "range": { "price": { "gte": 50, "lte": 200 } } },
        { "term": { "in_stock": true } }
      ],
      "should": [
        { "term": { "brand.raw": "Nike" } }         // Boost Nike results
      ],
      "minimum_should_match": 0                      // should is optional here
    }
  }
}
```

**Rule**: put everything that doesn't need scoring into `filter`. Especially: date ranges, category filters, boolean flags, numeric ranges. Only put full-text search in `must`.

---

## Aggregations

Aggregations use **doc values** (columnar forward index), not the inverted index. Doc values are pre-sorted on disk, making aggregations fast without loading `_source`.

### Terms Aggregation (GROUP BY equivalent)
```json
{
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.raw",   // Must be keyword
        "size": 10,
        "order": { "_count": "desc" }
      },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "revenue": { "sum": { "field": "price" } }
      }
    }
  }
}
```

**Gotcha**: `terms` aggregation is approximate for high-cardinality fields across shards. Each shard returns top N, coordinator merges. A term ranked 11th on each shard might be 1st globally. Increase `shard_size` to reduce error:
```json
"terms": { "field": "user_id", "size": 100, "shard_size": 1000 }
```

### Date Histogram
```json
{
  "aggs": {
    "events_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "1d",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

### Nested Aggregations
For arrays of objects, use `nested` mapping + `nested` aggregation context. Without `nested`, object arrays are "flattened" — cross-object queries return false positives.

```json
// Mapping
"line_items": {
  "type": "nested",
  "properties": {
    "product_id": { "type": "keyword" },
    "quantity": { "type": "integer" }
  }
}

// Query: find orders with product X qty > 5
{
  "query": {
    "nested": {
      "path": "line_items",
      "query": {
        "bool": {
          "must": [
            { "term": { "line_items.product_id": "prod-123" } },
            { "range": { "line_items.quantity": { "gt": 5 } } }
          ]
        }
      }
    }
  }
}
```

---

## Index Lifecycle Management (ILM)

For time-series data (logs, events), ILM automates the hot/warm/cold/delete lifecycle.

```json
PUT _plugins/_ism/policies/logs_policy
{
  "policy": {
    "description": "Log rotation policy",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          { "rollover": { "min_size": "50gb", "min_index_age": "1d" } }
        ],
        "transitions": [{ "state_name": "warm", "conditions": { "min_index_age": "3d" } }]
      },
      {
        "name": "warm",
        "actions": [
          { "replica_count": { "number_of_replicas": 0 } },
          { "force_merge": { "max_num_segments": 1 } }  // Compact to 1 segment
        ],
        "transitions": [{ "state_name": "cold", "conditions": { "min_index_age": "30d" } }]
      },
      {
        "name": "cold",
        "actions": [
          { "index_priority": { "priority": 0 } }
          // Optionally: snapshot to S3, then remove from cluster
        ],
        "transitions": [{ "state_name": "delete", "conditions": { "min_index_age": "90d" } }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }]
      }
    ]
  }
}
```

**Node attributes** let you assign indices to "hot" (SSD) vs "warm" (HDD) nodes:
```json
// Hot node config: node.attr.box_type: hot
// Warm node config: node.attr.box_type: warm

// Index routing on warm:
PUT /logs-2024.01/_settings
{ "index.routing.allocation.require.box_type": "warm" }
```

---

## Vector Search (k-NN Plugin)

### The knn_vector Field

```json
PUT /documents
{
  "settings": {
    "index.knn": true                    // Enable k-NN for this index
  },
  "mappings": {
    "properties": {
      "text": { "type": "text" },
      "embedding": {
        "type": "knn_vector",
        "dimension": 768,                // Must match your embedding model output
        "space_type": "cosinesimil",     // l2, cosinesimil, innerproduct
        "method": {
          "name": "hnsw",
          "engine": "faiss",             // faiss, nmslib, lucene
          "parameters": {
            "m": 16,                     // HNSW connections per node
            "ef_construction": 128       // Build-time beam width
          }
        }
      }
    }
  }
}
```

### HNSW Engine Comparison

| Engine | Best for | Notes |
|--------|----------|-------|
| `faiss` | Large-scale workloads, GPU support, IVF, high indexing throughput | Default engine when unspecified; recommended for large-scale/billion-vector use cases |
| `lucene` | Smaller deployments (up to a few million vectors) | Native Lucene HNSW; often better latency/recall than Faiss at smaller scale; benefits from Lucene's automatic filtering strategy selection; converging with Faiss via "Lucene-on-Faiss" work for memory-efficient search |
| `nmslib` | Deprecated — do not use for new indexes | Deprecated as of OpenSearch 2.19; blocked for new indexes starting in 3.0 |

**Faiss IVF (Inverted File Index)**: clusters vectors into `nlist` cells at index build time. At query time, only searches `nprobe` nearest cluster centroids. Dramatically faster than pure HNSW for billion-scale vectors at cost of recall.

```json
"method": {
  "name": "ivf",
  "engine": "faiss",
  "parameters": {
    "nlist": 4096,    // Number of clusters (sqrt(N) is a heuristic)
    "nprobe": 128     // Clusters to search at query time (recall vs speed trade-off)
  }
}
```

### HNSW Parameters Explained

```
m (default 16):
  - Edges per node in the HNSW graph
  - Higher m → better recall, more memory, slower build
  - Range: 4-64; 16 is good default

ef_construction (default 128):
  - Beam width during index build
  - Higher → better quality graph, slower indexing
  - Range: 64-512

ef_search (at query time, default 512):
  - Beam width during query traversal
  - Higher → better recall, slower query
  - Can tune at query time without rebuilding index
```

### Approximate k-NN Query

```json
GET /documents/_search
{
  "size": 10,
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.1, 0.4, 0.2, ...],   // Your query embedding (768 floats)
        "k": 10                             // Return top 10 nearest neighbors
      }
    }
  }
}
```

### Approximate vs Exact k-NN

| | Approximate (HNSW) | Exact (script_score) |
|-|-------------------|----------------------|
| Speed | Sub-millisecond (O(log N)) | O(N) — slow for large corpora |
| Recall | 95-99% (tunable) | 100% |
| Memory | High (HNSW graph in RAM) | None extra |
| With filters | Tricky (see pre-filtering) | Works perfectly |
| Use case | Production similarity search | Small corpora (<100K), evaluation |

```json
// Exact k-NN via script_score (brute force, 100% recall)
{
  "query": {
    "script_score": {
      "query": { "match_all": {} },
      "script": {
        "source": "knn_score",
        "lang": "knn",
        "params": {
          "field": "embedding",
          "query_value": [0.1, 0.4, ...],
          "space_type": "cosinesimil"
        }
      }
    }
  }
}
```

---

## Pre-Filtering: The Recall Problem

This is the most important concept for production vector search with metadata filters.

### The Problem with `post_filter`

```json
// WRONG: post_filter loses recall
{
  "query": { "knn": { "embedding": { "vector": [...], "k": 10 } } },
  "post_filter": { "term": { "category": "tech" } }
}
// k-NN returns top 10 globally → post_filter may eliminate most → you get 0-3 results
```

### Why HNSW + Pre-Filter is Hard

HNSW doesn't natively support filtered search — it traverses the graph without knowing which nodes pass your filter. Two approaches:

**1. Pre-filter then exact k-NN** (good for highly selective filters):
```json
{
  "query": {
    "script_score": {
      "query": {                              // pre-filter first
        "bool": {
          "filter": [
            { "term": { "category": "tech" } },
            { "range": { "date": { "gte": "2024-01-01" } } }
          ]
        }
      },
      "script": {                             // then exact k-NN over filtered set
        "source": "knn_score",
        "lang": "knn",
        "params": {
          "field": "embedding",
          "query_value": [...],
          "space_type": "cosinesimil"
        }
      }
    }
  }
}
```
This applies the boolean filter first (fast, cached), then does brute-force similarity over the filtered candidates. Works well when filter is selective (<1% of docs).

**2. Efficient filter via `k-NN` with `filter` parameter** (OpenSearch 2.4+):
```json
{
  "query": {
    "knn": {
      "embedding": {
        "vector": [...],
        "k": 10,
        "filter": {                   // Native filtered HNSW (re-traverses if needed)
          "term": { "category": "tech" }
        }
      }
    }
  }
}
```
OpenSearch's filtered k-NN does approximate search and re-traverses the graph if filtered results are fewer than k. More recall-aware than post_filter.

---

## Hybrid Search: BM25 + k-NN

OpenSearch 2.x+ supports combining keyword (BM25) and vector (k-NN) search scores in a single query.

```json
GET /documents/_search
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "match": {
            "title": {
              "query": "running shoes lightweight",
              "boost": 1.0
            }
          }
        },
        {
          "knn": {
            "embedding": {
              "vector": [0.1, 0.4, ...],
              "k": 100             // Fetch more candidates for merging
            }
          }
        }
      ]
    }
  },
  "search_pipeline": {
    "phase_results_processors": [
      {
        "normalization-processor": {
          "normalization": { "technique": "min_max" },  // or l2
          "combination": {
            "technique": "arithmetic_mean",
            "parameters": { "weights": [0.3, 0.7] }    // 30% BM25, 70% vector
          }
        }
      }
    ]
  }
}
```

**Score normalization is critical** — BM25 scores and cosine similarity scores are on different scales. Without normalization, one would dominate. Use `min_max` or `l2` normalization before combining.

```json
// Define pipeline once, apply to index
PUT /_search/pipeline/hybrid_pipeline
{
  "phase_results_processors": [{
    "normalization-processor": {
      "normalization": { "technique": "min_max" },
      "combination": {
        "technique": "harmonic_mean",
        "parameters": { "weights": [0.4, 0.6] }
      }
    }
  }]
}

// Use pipeline on query
GET /documents/_search?search_pipeline=hybrid_pipeline
{ "query": { "hybrid": { ... } } }
```

---

## Neural Search Pipeline

Built-in text-to-embedding without external preprocessing:

```json
// 1. Upload ML model (sentence-transformers/all-MiniLM-L6-v2 available via model hub)
POST /_plugins/_ml/models/_register
{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_format": "TORCH_SCRIPT"
}
// Returns model_id: "abc123"

// 2. Deploy model
POST /_plugins/_ml/models/abc123/_deploy

// 3. Create ingest pipeline (auto-embed on index)
PUT /_ingest/pipeline/nlp-pipeline
{
  "processors": [{
    "text_embedding": {
      "model_id": "abc123",
      "field_map": {
        "title": "title_embedding",
        "body": "body_embedding"
      }
    }
  }]
}

// 4. Index documents (embeddings generated automatically)
PUT /documents/_settings { "index.default_pipeline": "nlp-pipeline" }

POST /documents/_doc
{ "title": "Running shoes review", "body": "..." }
// title_embedding and body_embedding auto-populated
```

---

## Concrete Vector Search Example

Document similarity search with metadata filter:

```python
from opensearchpy import OpenSearch
import numpy as np

client = OpenSearch(hosts=["localhost:9200"])

# Index a document with embedding
client.index(
    index="articles",
    body={
        "title": "Neural networks in production",
        "author": "Alice",
        "tags": ["ml", "infrastructure"],
        "published_at": "2024-03-15",
        "embedding": model.encode("Neural networks in production").tolist()
    }
)

# Hybrid search: semantic + filter
query_vec = model.encode("machine learning deployment").tolist()

response = client.search(
    index="articles",
    body={
        "size": 5,
        "query": {
            "script_score": {
                "query": {
                    "bool": {
                        "filter": [
                            {"term": {"tags": "ml"}},
                            {"range": {"published_at": {"gte": "2024-01-01"}}}
                        ]
                    }
                },
                "script": {
                    "source": "knn_score",
                    "lang": "knn",
                    "params": {
                        "field": "embedding",
                        "query_value": query_vec,
                        "space_type": "cosinesimil"
                    }
                }
            }
        },
        "_source": ["title", "author", "published_at", "tags"]
    }
)

for hit in response["hits"]["hits"]:
    print(f"{hit['_score']:.4f}  {hit['_source']['title']}")
```

---

## Operational: Cluster Sizing

### Node Roles

```
Dedicated Master Nodes (3, odd number):
  - Manages cluster state: shard allocation, index creation, membership
  - No data, no queries
  - Small instance (2-4 vCPU, 8GB RAM)
  - 3 nodes for quorum (prevents split-brain: floor(3/2)+1 = 2)

Data Nodes:
  - Store shards, execute queries
  - Size based on data volume and query load
  - Rule of thumb: 30-50GB data per vCPU, 50% RAM for heap

Coordinator Nodes (dedicated, optional for large clusters):
  - Route requests, scatter-gather, aggregate results
  - No shards, high memory for large aggregations
  - Add when coordinators become CPU/memory bottleneck

Ingest Nodes:
  - Run ingest pipelines (transformations, enrichment)
  - Useful when pipeline processing is heavy (ML embeddings)
```

### Shard Sizing Rules

The #1 operational mistake: **oversharding**. OpenSearch has overhead per shard:
- Each shard = a Lucene index = JVM file handles, heap metadata
- 1000 shards = OpenSearch spending 30% of time on shard management overhead

```
Rules of thumb:
  - Target 10-50GB per shard (never <1GB, never >50GB)
  - Max ~20 shards per GB of heap (e.g., 30GB heap → max 600 shards per node)
  - Total cluster shards = data_nodes × shards_per_node target
  
For 1TB of data:
  - 20-30 primary shards of ~40GB each is reasonable
  - 1 replica = 40-60 shards total per node (for 2 data nodes)
  
Common mistake: creating index with 10 shards/1 replica for a 1GB index
  → 20 shards, each ~50MB — pure overhead
```

```json
// Fix: shrink oversharded index (must make read-only first)
POST /my-index/_shrink/my-index-shrunk
{
  "settings": {
    "index.number_of_shards": 2,
    "index.number_of_replicas": 1
  }
}
```

### Heap and Memory

```
Heap sizing:
  - Set to 50% of RAM, max 32GB (above 32GB, JVM loses compressed OOPs)
  - Remaining 50% goes to OS page cache (Lucene uses this heavily for segment files)
  - Example: 64GB node → 31GB heap, 33GB OS cache

Heap is used for:
  - Aggregation buffers (terms agg with high cardinality = OOM risk)
  - k-NN HNSW graphs (loaded into native memory, NOT heap — monitor separately)
  - Field data cache (avoid text field aggregations without fielddata=true)
  - Segment metadata

k-NN native memory:
  - HNSW graphs live in native (off-heap) memory via Faiss
  - Monitor: GET /_cat/nodes?v&h=name,ram.current,ram.percent
  - Set: knn.memory.circuit_breaker.limit: 60%  (% of JVM heap as proxy)
```

---

## Oversharding Problem Deep Dive

```
Symptom: high CPU, slow queries, cluster state updates slow

Why oversharding kills performance:
1. Each query fans out to ALL shards, even empty ones
2. Cluster state (shard table) grows → slower cluster state updates
3. Each shard has fixed per-segment overhead
4. Too many small segments across many shards → many Lucene merge threads competing

Diagnosis:
GET /_cat/shards?v  // See shard count and sizes
GET /_stats         // See index-level stats

Fix for time-series:
- Use data streams with proper rollover (max_size=50gb)
- ISM policy to force_merge warm/cold indices to 1 segment

Fix for static indices:
- Shrink API (halves shards: must be divisible)
- Reindex with correct shard count
```

---

## Cross-Cluster Search (CCS)

Query multiple clusters from one coordinator:

```json
// opensearch.yml on coordinator:
cluster.remote.cluster_a.seeds: ["node-a1:9300", "node-a2:9300"]
cluster.remote.cluster_b.seeds: ["node-b1:9300"]

// Query across clusters:
GET /cluster_a:products,cluster_b:products/_search
{ "query": { "match": { "title": "running shoes" } } }

// Or with index pattern:
GET /cluster_*:products/_search
```

CCS adds network latency (coordinator fetches from remote coordinator). Use for:
- Multi-region search with data locality
- Separate dev/staging clusters queryable from one endpoint
- Compliance: data stays in region, queries cross-region

---

## OpenSearch vs Elasticsearch

OpenSearch was forked from Elasticsearch 7.10 (2021) when AWS open-sourced it under Apache 2.0 (vs Elastic's SSPL).

| Dimension | OpenSearch | Elasticsearch |
|-----------|------------|---------------|
| License | Apache 2.0 | Elastic License 2.0 (non-OSS) |
| Hosted | Amazon OpenSearch Service | Elastic Cloud |
| k-NN plugin | Mature, Faiss (default) + Lucene HNSW + IVF built-in; nmslib deprecated (2.19+) / removed for new indexes (3.0+) | Integrated in 8.x |
| ML features | ML Commons plugin | Elastic ML (proprietary) |
| Dashboards | OpenSearch Dashboards (Kibana fork) | Kibana (proprietary features) |
| API compat | Compatible through v2.x | Diverged significantly in 8.x |
| Security | Free, built-in | Free tier + paid X-Pack |
| Performance | Comparable | Comparable |

**Practical difference**: if you're on AWS and want managed service + Apache license, OpenSearch. If you need Elastic's ML stack or the latest Elasticsearch 8.x features (ELSER semantic search, etc.), use Elasticsearch. Core search semantics are essentially identical for most use cases.

---

## Deep Internals

### How BM25 is Computed at Query Time

1. **TermQuery execution**: look up `"shoes"` in inverted index → get posting list `[docID, freq, positions]`
2. **Norm lookup**: for each docID, fetch the pre-computed length norm from norms file
3. **IDF**: pre-computed per-segment or computed at query time from segment-level doc frequency stats
4. **Score computation**: SIMD-optimized inner loop over posting list
5. **TopN collection**: min-heap of size N, discard lower-scoring docs (WAND optimization)

### WAND (Weak AND) Optimization

For top-N retrieval, Lucene uses WAND to skip low-scoring documents without scoring them. Each term maintains an upper bound score estimate. If sum of upper bounds for remaining terms < current Nth best score, entire document can be skipped without computing actual score. This makes `size=10` queries dramatically faster than full table scans even on million-document indices.

### Segment-Level Parallelism

Within a shard, Lucene can search multiple segments in parallel. OpenSearch spawns threads per segment (thread pool size = CPU cores). For heavily segmented indices (many refreshes, few merges), this helps — but more threads also means more context switching. Force-merging to 1 segment eliminates per-segment overhead at cost of large merge operation.

### The `_source` vs Doc Values Trade-off

- **`_source`**: compressed JSON blob for the original document. Stored as-is. Retrieved for display. Expensive to deserialize for aggregations.
- **Doc values**: columnar, pre-sorted, on-disk format optimized for aggregation/sorting. Low overhead per field access.
- **Stored fields**: individual fields stored for retrieval without full `_source` deserialization.

```json
// Disable _source for pure aggregation indices (save storage, lose doc retrieval):
"mappings": {
  "_source": { "enabled": false }  // Can't retrieve original docs, only aggregations
}

// Or store only specific fields:
"_source": { "includes": ["id", "title"], "excludes": ["embedding"] }
```

---

## Key Gotchas

1. **`text` field aggregations fail** unless you enable `fielddata: true` (loads entire column into heap — dangerous). Always use `.keyword` subfield for aggregations.

2. **HNSW index loads into memory at startup** — if you have 100GB of vector indices, startup takes minutes and requires that much native memory free. Monitor `knn.memory.circuit_breaker.triggered`.

3. **`cosinesimil` vs `innerproduct`**: if your embeddings are L2-normalized (unit vectors), `innerproduct` == `cosinesimil` but faster. Many embedding models (OpenAI `text-embedding-ada-002`, Sentence Transformers) output unit-normalized vectors by default.

4. **k-NN index is immutable at dimension level**: you cannot change the `dimension` field after index creation. If your embedding model changes output dimension (e.g., 768→1536), you must reindex.

5. **Oversharding is invisible until it hurts**: monitor `GET /_cat/shards?v` regularly. Target <500 shards per node.

6. **`terms` aggregation is approximate**: for exact counts, use `cardinality` aggregation (HyperLogLog, ~5% error) or ensure `shard_size` is large enough.

7. **Dynamic mapping + nested arrays**: without `nested` type, `{"items": [{"id":1,"price":10}, {"id":2,"price":5}]}` stores as `items.id: [1,2]` and `items.price: [10,5]` — a query for `id=1 AND price=5` would match this document (false positive). Use `nested` type to preserve object correlations.

8. **`post_filter` after k-NN = bad recall**: always use script_score with pre-filter or native filtered k-NN.

9. **Refresh overhead in bulk indexing**: disable refresh (`-1`) during bulk loads. Each refresh creates a new segment that requires I/O and merge capacity.

10. **Split-brain prevention**: always deploy 3 dedicated master nodes. With 2, a network partition → both nodes think they're master → data corruption risk.

---

## When to Use / When NOT to Use

### Use OpenSearch when:
- Full-text search with relevance ranking is required
- Faceted search (e-commerce filters, log aggregations)
- Log/event analytics (ELK-style: ingest logs, search + dashboard)
- Semantic/hybrid search combining text and embeddings
- Time-series data with ILM (logs, metrics, traces)
- You need both BM25 keyword search AND vector similarity in one system
- AWS-native deployment with managed service

### Do NOT use OpenSearch when:
- Primary use case is relational queries/JOINs (use Postgres/DuckDB)
- OLAP-style analytics on structured data (use DuckDB/Trino/Redshift)
- Pure vector database at billion scale (use Pinecone, Weaviate, or pgvector for simplicity)
- Low-latency OLTP point lookups by primary key (use DynamoDB/Redis)
- Strict ACID transactions required
- Very small datasets (<100K docs): Postgres full-text search is simpler
- Real-time streaming processing (use Kafka + Flink, then sink to OpenSearch)
