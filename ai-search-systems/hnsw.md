# HNSW: Hierarchical Navigable Small World

## 30-Second Intuition

HNSW builds a layered graph where higher layers have long-range edges (like highway connections between cities) and lower layers have fine-grained edges (like local streets). To find nearest neighbors, enter at the top, greedily walk toward the query, descend a layer, repeat. This achieves O(log N) search with 98-99% recall — far better than IVF's centroid approximation at the same latency budget.

---

## The Small World Graph Intuition

Before HNSW came NSW (Navigable Small World). The key insight from network theory:

**Small world property**: any two nodes can be reached in O(log N) hops if you have both:
- **Short-range edges**: connect nearby nodes (for precision)
- **Long-range edges**: connect distant nodes (for navigation)

This is the "six degrees of separation" phenomenon in social networks. HNSW applies this to vector search.

```
Layer 2 (sparsest, longest hops):
  A ─────────────────────── B

Layer 1:
  A ──── C ──── D ──── B

Layer 0 (densest, all nodes):
  A ─ E ─ F ─ C ─ G ─ D ─ H ─ B
  │                           │
  └───────────────────────────┘
```

Search: start at Layer 2, jump toward query, descend to Layer 1, refine, descend to Layer 0, find exact neighbors.

---

## Layer Structure

### How Layers Are Assigned

Each node gets a maximum layer assignment drawn from an exponential distribution:

```python
import math, random

def assign_layer(M_L: float) -> int:
    """M_L = 1/ln(M), controls layer density"""
    return int(-math.log(random.random()) * M_L)

# With M=16 → M_L ≈ 0.36
# P(layer >= 0) = 100%
# P(layer >= 1) ≈ 36%
# P(layer >= 2) ≈ 13%
# P(layer >= 3) ≈ 4.7%
# P(layer >= 4) ≈ 1.7%
```

Most nodes live only at layer 0. A few reach layer 1. Fewer still reach higher layers. The graph naturally becomes sparser at higher layers — longer-range connections dominate.

### Node Count per Layer (Expected Values)

```
Layer:  0        1        2        3
Nodes:  100%     36%      13%      5%
(for 1M nodes: 1,000,000 / 360,000 / 130,000 / 50,000)
```

---

## Construction Algorithm

### Inserting a New Node q

```python
def insert(q: Vector, M: int, ef_construction: int, layers: List[Layer]):
    l_q = assign_layer()          # new node's max layer
    ep = entry_point               # current graph entry point

    # From top layer down to l_q + 1: just find closest node
    for lc in range(top_layer, l_q, -1):
        W = search_layer(q, ep, ef=1, layer=lc)
        ep = nearest(W, q)

    # From l_q down to 0: find ef_construction neighbors, add edges
    for lc in range(min(l_q, top_layer), -1, -1):
        W = search_layer(q, ep, ef=ef_construction, layer=lc)
        neighbors = select_neighbors(q, W, M)
        add_edges(q, neighbors, layer=lc)

        # Prune existing nodes that now have too many connections
        for n in neighbors:
            if degree(n, lc) > M_max:
                prune_edges(n, M_max, lc)

        ep = nearest(W, q)

    if l_q > top_layer:
        entry_point = q             # new global entry point
```

**Key detail**: during construction, each new node is connected to its M nearest neighbors at each layer it occupies. This is expensive (calls `search_layer` which itself is O(log N)), but done once at index time.

### Neighbor Selection: Simple vs Heuristic

**Simple**: just take the M closest nodes from candidates W.

**Heuristic** (default in most implementations):
```
Select neighbors that are close to q AND diverse in direction.
Avoid selecting 3 nodes all clustered together in one direction.
This preserves "navigability" — long-range structural diversity.
```

The heuristic produces better recall at the same M but is slower to build.

---

## Search Algorithm

```python
def knn_search(q: Vector, K: int, ef: int) -> List[Vector]:
    ep = entry_point

    # Greedy descent through top layers (ef=1: just track single best)
    for lc in range(top_layer, 0, -1):
        W = search_layer(q, ep, ef=1, layer=lc)
        ep = nearest(W, q)

    # At layer 0: exhaustive search with ef candidates
    W = search_layer(q, ep, ef=ef, layer=0)

    return top_k(W, K)

def search_layer(q, ep, ef, layer) -> List[Vector]:
    visited = {ep}
    candidates = min_heap([ep])   # sorted by distance to q (ascending)
    dynamic_list = max_heap([ep]) # sorted by distance to q (descending), size ef

    while candidates:
        c = pop_nearest(candidates)   # nearest unprocessed
        f = furthest(dynamic_list)    # furthest in current result set

        if dist(c, q) > dist(f, q):
            break  # all candidates further than worst result → stop

        for n in neighbors(c, layer):
            if n not in visited:
                visited.add(n)
                f = furthest(dynamic_list)
                if dist(n, q) < dist(f, q) or len(dynamic_list) < ef:
                    candidates.push(n)
                    dynamic_list.push(n)
                    if len(dynamic_list) > ef:
                        pop_furthest(dynamic_list)

    return dynamic_list
```

**Critical path**: the `while candidates` loop is the hot loop. In practice, for a well-built graph with M=16, this terminates in ~30-50 iterations at layer 0.

---

## Key Parameters: Impact on Recall vs Speed

### M — Connections Per Node Per Layer

```
M = 4:   sparse graph, fast build, fast search, recall ~90-93%
M = 16:  default, good balance, recall ~97-98%
M = 32:  denser graph, slower build, more memory, recall ~99%
M = 64:  diminishing returns, 4x memory of M=16, recall ~99.5%
```

Higher M means each node has more edges to navigate through. Navigability improves, but memory and build time scale as O(M).

**M_max at layer 0 is typically 2*M** — layer 0 gets denser connections because it's where the final precision search happens.

### ef_construction — Build Quality

```
ef_construction = 40:  fast build, lower graph quality, recall ~94%
ef_construction = 100: standard, good quality, recall ~97%
ef_construction = 200: high quality, 2x slower build, recall ~98.5%
ef_construction = 500: very high quality, 5x slower, recall ~99%
```

This is the size of the dynamic candidate list during construction's `search_layer` at layer 0. Higher value = better neighborhood found for each new node = better graph quality.

**Rule**: ef_construction must be >= K (number of results you want). Typically set to max(100, 2*K).

### ef (Search Quality)

```
ef = 10:  fast, lower recall (~92%)
ef = 40:  standard, good recall (~97%)
ef = 100: high recall (~99%), 2.5x slower than ef=40
ef = 200: very high recall (~99.7%), expensive
```

ef is the runtime parameter — no rebuild needed to change it. Tune per query type:
- Quick autocomplete → ef=10
- Precision-critical retrieval → ef=200

**ef must be >= K**. If K=20 and ef=10, you'll only get 10 results.

---

## Why HNSW Beats IVF for Low-Latency

IVF's fundamental problem: centroid approximation error.

```
IVF search:
  1. Find nearest centroid to query
  2. Search vectors in that cluster

If query sits near a cluster boundary:
  true nearest neighbor → in adjacent cluster
  IVF search → misses it

Mitigation: increase nprobe (search more clusters)
But: nprobe=64 means 64 cluster scans → approaches brute force
```

HNSW's advantage:
```
No centroid step → no approximation error from quantization
Graph traversal naturally handles boundary cases
Recall stays high at low ef values (10-40ms latency range)
```

**Practical benchmark**: at 95% recall@10, HNSW achieves 5-10ms latency vs IVF's 15-30ms for 1M 768d vectors.

---

## Memory Cost

```
Per vector cost:
  - Raw vector: d * 4 bytes  (float32)
  - HNSW graph edges: M * 2 * 4 bytes  (int32 node IDs, bidirectional)

At layer 0: M_max = 2*M connections
At upper layers: M connections

Approximate total per node (M=16, d=768):
  Vector:      768 * 4 = 3,072 bytes
  Layer 0 edges: 32 * 4 = 128 bytes
  Upper layer edges: small fraction (most nodes don't have upper layers)
  Overhead (visited sets, heap alloc): ~128 bytes

  Total ≈ 3,328 bytes/vector ≈ 3.3 KB/vector
```

For 1M vectors at 768d:
```
Raw vectors:  768 * 4 * 1M = 3.07 GB
HNSW graph:   ~0.5 GB (M=16)
Total:        ~3.6 GB  (in RAM — HNSW must be memory-resident for performance)
```

For 1M vectors at 1536d:
```
Raw vectors:  6.14 GB
HNSW graph:   ~0.5 GB
Total:        ~6.6 GB
```

**This is why HNSW is memory-bound**: the entire index must fit in RAM. Disk-based ANN (DiskANN, FAISS IVF with mmap) trades latency for memory savings.

---

## Deletions: The Hard Problem

HNSW doesn't support true deletions efficiently.

```
Problem: node B is connected to 5 nodes.
Delete B → what happens to its neighbors?
Their paths through B are now broken.

Option 1: Tombstoning
  Mark B as deleted. During search, skip deleted nodes.
  Problem: deleted nodes still take memory and pollute traversal.
  After many deletes, recall degrades as graph becomes fragmented.

Option 2: Full rebuild
  Expensive. O(N log N).

Option 3: Soft delete + periodic consolidation (most production systems)
  Tombstone deletes. Periodically rebuild the index from scratch.
  Run rebuild offline, swap atomically.
```

**Qdrant's approach**: tracks deleted payload, marks tombstones, triggers compaction when deletion ratio exceeds threshold.

**Design implication**: if your use case has frequent deletes (user-deleted documents, expired records), factor in rebuild cost. Consider time-partitioned indexes (new index per week) where old partitions are dropped wholesale.

---

## hnswlib vs FAISS HNSW vs Native Implementations

| Implementation | Notes |
|---------------|-------|
| **hnswlib** | Original C++ reference impl by Malkov (the author). Single-file header. Fast. Limited batch support. Python bindings. Still the widely-embedded reference impl as of 2026, but maintainer activity has slowed — stable/dormant rather than actively evolving. Good for small-scale custom use. |
| **FAISS HNSW** | Meta's implementation. Integrates with FAISS quantization (PQ, SQ8). Better batch build throughput. As of Faiss 1.10 (2025), integrates NVIDIA cuVS/CAGRA — GPU-built graphs convert to CPU-compatible HNSW format, up to ~12x faster builds and ~4.7x faster search vs CPU HNSW. Native GPU *search* on HNSW itself still isn't a thing; CAGRA is the GPU-native graph algorithm. |
| **Qdrant HNSW** | Rust reimplementation. Filtered HNSW (search with metadata filters without recall degradation); added the ACORN algorithm in 2026 for strict/highly selective filters. Shipped GPU-accelerated HNSW index building (v1.13, 2025) — ~10x faster ingestion, NVIDIA/AMD/Intel GPUs supported, now also in Qdrant Cloud. Production-grade. Best option for full VDB. |
| **pgvector HNSW** | C implementation inside Postgres. IVFFlat also available. Reasonably fast; good for < 5M vectors — beyond that, recall/memory degrade (HNSW is RAM-bound). The **pgvectorscale** extension (Timescale) adds a disk-based StreamingDiskANN index for scaling to hundreds of millions of vectors without pgvector's RAM ceiling. |
| **Weaviate HNSW** | Go implementation. Supports async indexing (opt-in via `ASYNC_INDEXING=true`, decouples ingestion from graph build via a queue). Well-integrated with Weaviate's pipeline. |
| **nmslib** | Deprecated in practice, though not formally archived (still on GitHub/PyPI). OpenSearch dropped its NMSLIB engine as of 3.0 in favor of Faiss/Lucene. Treat as legacy — don't pick it for new systems. |
| **NVIDIA cuVS / CAGRA** | GPU-native graph ANN algorithm (not HNSW, but a direct competitor). Up to ~12x faster index builds and ~4.7x faster search vs CPU HNSW; adopted by Faiss and Elasticsearch. Worth evaluating for GPU-available, large-scale deployments. NVIDIA/Microsoft are also porting DiskANN/Vamana to GPU. |

---

## Quantization: Compress Vectors Without Killing Recall

### Scalar Quantization (SQ8)
```
float32 → int8: 4x memory reduction
Map per-dimension float to [-128, 127] range

Quality loss: ~0.5-1% recall
Speed gain: 2-4x (int8 SIMD vs float32)

Implementation:
  scale = (max_val - min_val) / 255
  quantized = int8((val - min_val) / scale)
```

### Product Quantization (PQ)
```
Split 768d vector into 8 subvectors of 96d each.
Quantize each subspace independently with k=256 centroids.
Store 8 bytes per vector (1 byte = centroid ID per subspace).

Memory: 768d * 4 bytes = 3072 bytes → 8 bytes (384x compression!)
Recall loss: 3-8% depending on PQ segments and training data quality

Distance computation: precompute lookup tables, then table lookups (fast).
```

### PQ with Re-scoring
```
1. Fast ANN search using PQ-compressed vectors: retrieve top-N (N=500)
2. Re-score top-N using original float32 vectors
3. Return top-k from re-scored results

Result: near-full recall, compressed memory, small latency overhead for re-scoring
```

This pattern (PQ search + exact re-score) is used in production by major systems to handle billion-scale with limited RAM.

---

## Concrete Benchmark: 1M Vectors at 768d, 10ms p99 Target

**Hardware**: single machine, 32GB RAM, 8 cores

**Target**: 10ms p99, recall@10 ≥ 97%, K=10

```python
import hnswlib
import numpy as np

dim = 768
num_elements = 1_000_000

# Build index
p = hnswlib.Index(space='cosine', dim=dim)
p.init_index(max_elements=num_elements, ef_construction=200, M=16)

# Insert in batches (hnswlib supports parallel insertion)
p.set_num_threads(8)
p.add_items(data, num_threads=8)

# Query settings
p.set_ef(50)  # ef >= K; start at 50, tune down if latency too high

# Benchmark: measure p99 at ef=50
# Typical result:
# - QPS: ~800 single-threaded, ~5000 multi-threaded
# - Recall@10: ~98%
# - p50: 3ms, p95: 7ms, p99: 10ms  ← hits target
```

**Parameter tuning recipe:**
```
1. Build once with M=16, ef_construction=200
2. Benchmark recall at ef=10, 20, 40, 80, 100, 200
3. Find lowest ef that achieves recall target (97%)  → ef=50 typically
4. Benchmark latency at that ef with expected QPS
5. If latency too high: add more replicas or increase M_max (rebuild needed)
6. If memory too high: use SQ8 quantization (4x savings, ~1% recall loss)
```

**Memory budget:**
```
1M * 768d * float32 = 3.07 GB (vectors)
HNSW graph (M=16): ~0.5 GB
Total index: ~3.6 GB → fits easily in 32GB
For 10M vectors: ~36 GB → need dedicated box or quantize
```

---

## Key Gotchas

- **ef must be >= K**: if ef < K, you'll get fewer than K results. Many silent bugs come from this.
- **ef_construction must be set before insert**: you can't change it after building. Rebuild required.
- **M changes require full rebuild**: you cannot add edges to an existing HNSW graph.
- **Thread safety**: hnswlib's `add_items` with `num_threads > 1` is safe; concurrent reads + writes need locking in most implementations.
- **Entry point matters**: the single global entry point at the top layer is a hot path. If that node is deleted, search degrades badly. Some implementations handle this with entry point rotation.
- **Warm-up queries**: first queries after loading index from disk are slow (page cache cold). Run 100 warm-up queries before serving production traffic.
- **Layered structure is probabilistic**: two builds with the same data and different random seeds produce different graphs with similar (but not identical) recall.
- **Don't compare HNSW recall across implementations**: hnswlib, FAISS, and Qdrant use slightly different neighbor selection heuristics. Benchmark on your data.
