# Vectorless RAG: Beyond Dense Retrieval

## 30-Second Intuition

Dense vector search excels at semantic similarity, but it's not always the right retrieval primitive. When queries are keyword-heavy (exact error codes, legal citations, function names), BM25 often beats vectors. When queries require multi-hop reasoning over relationships, graph traversal beats similarity search. When data is structured, SQL beats all of them. The best production systems combine multiple retrieval strategies.

---

## Why Vectors Aren't Always the Right Primitive

Dense embeddings encode semantics by collapsing a text into a single vector. This is lossy:

```
Query: "NullPointerException in UserService.getUserById line 142"

Dense embedding captures: "java exception user service method"
Loses: exact class name, exact method name, exact line number

BM25 term matching:
  "NullPointerException" → exact match → high score
  "UserService.getUserById" → exact match → high score
  "line 142" → partial match

Result: BM25 returns the exact stack trace.
        Dense search returns semantically similar but textually different errors.
```

**The fundamental tension**: embeddings excel at "what does this mean?" BM25 excels at "where is this exact string?" Most real queries need both.

---

## BM25 / TF-IDF: Still Beats Dense in Key Domains

### How BM25 Works

```
BM25 score for document D given query Q:

score(D, Q) = Σ IDF(qi) * (tf(qi, D) * (k1 + 1)) / (tf(qi, D) + k1 * (1 - b + b * |D|/avgdl))

Where:
  qi = each query term
  tf(qi, D) = term frequency in document
  IDF(qi) = log((N - df(qi) + 0.5) / (df(qi) + 0.5))  [inverse doc frequency]
  k1 = term frequency saturation (default 1.2-2.0)
  b = length normalization (default 0.75)
  |D| = document length, avgdl = average doc length
```

Key insight: **IDF penalizes common words** ("the", "is") and rewards rare words ("NullPointerException", "getUserById"). This is exactly right for technical search.

### When BM25 Beats Dense Retrieval

| Domain | Why BM25 Wins |
|--------|--------------|
| Legal/medical | Exact terminology required; synonyms unacceptable ("myocardial infarction" ≠ "heart attack" in a legal brief) |
| Code search | Function names, variable names, error codes need exact match |
| Product catalogs | SKU numbers, model numbers, exact product names |
| Log analysis | Error codes, stack traces, specific field values |
| Compliance/audit | "SOC 2 Type II" must match exactly, not approximately |

```python
from rank_bm25 import BM25Okapi
import re

def tokenize(text: str) -> list[str]:
    # For code: preserve identifiers, camelCase splits
    return re.findall(r'[A-Z]?[a-z]+|[A-Z]+(?=[A-Z]|$)|\d+', text.lower())

corpus = [tokenize(doc) for doc in documents]
bm25 = BM25Okapi(corpus)

query_tokens = tokenize("NullPointerException UserService getUserById")
scores = bm25.get_scores(query_tokens)
top_n = scores.argsort()[-10:][::-1]
```

### Elasticsearch / OpenSearch BM25 in Production

```json
{
  "query": {
    "multi_match": {
      "query": "connection pool exhausted",
      "fields": ["title^3", "body", "tags^2"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}
```

Field boosting (`^3`) and tie-breaking let you tune signal relative to document structure — something vector search doesn't express easily.

---

## ColBERT: Late Interaction

### The Problem with Single-Vector Bi-Encoders

Standard embedding: entire passage → single vector. Information bottleneck.

```
Passage: "The connection pool is controlled by max_pool_size.
          It defaults to 10 in HikariCP. Set it higher for high-concurrency."

Single vector: [0.34, -0.12, 0.78, ...]
                ↑ Encodes the "gist" but loses token-level detail
```

### ColBERT's Solution: Token-Level Embeddings

```
Instead of one vector per passage, generate one vector per TOKEN.

Passage: "connection pool controlled max_pool_size defaults 10 HikariCP"
ColBERT: [[v_connection], [v_pool], [v_controlled], [v_max_pool_size], ...]
         Each token gets a 128d vector

Query: "what is default pool size"
ColBERT: [[v_what], [v_is], [v_default], [v_pool], [v_size]]
```

### MaxSim Scoring

```python
def maxsim_score(query_vecs, doc_vecs):
    """
    For each query token, find maximum similarity to any doc token.
    Sum these maxima.
    """
    score = 0
    for q_vec in query_vecs:
        max_sim = max(cosine_sim(q_vec, d_vec) for d_vec in doc_vecs)
        score += max_sim
    return score

# Intuition: "pool" in query matches "pool" in doc perfectly (sim=0.99)
#            "default" matches "defaults" well (sim=0.91)
#            Each query token "finds its best match"
```

**Why ColBERT is more precise**: the single-vector approach must encode "pool" and "HikariCP" and "defaults" all into one vector. MaxSim lets each query token independently find its best match in the document.

### ColBERT Trade-offs

```
Advantages:
  - Higher precision than bi-encoder, approaching cross-encoder quality
  - Still fast (pre-computed doc token vectors, batched MaxSim)
  - Indexable (use FAISS for initial filtering by token vectors)

Costs:
  - Storage: 128d * avg_tokens_per_doc vectors per document (vs 1 vector)
    128 tokens/doc * 128d * 4 bytes = 65KB/doc (vs 3KB for single vector)
    1M docs: ~65GB vs ~3GB
  - Retrieval: MaxSim requires comparing all query token vectors to all doc token vectors
```

```python
# Using RAGatouille (ColBERT wrapper) — note: as of 2026, RAGatouille's
# maintenance has shifted toward a PyLate-based backend; check current
# compatibility before adopting for new projects, especially with LangChain
# integrations (older llama-index RAGatouille packs are already deprecated).
from ragatouille import RAGPretrainedModel

RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
RAG.index(collection=documents, index_name="my_index")

results = RAG.search(query="what is default connection pool size", k=10)

# PyLate (Lightonai) is the actively-developed alternative for ColBERT-style
# late-interaction retrieval and is the direction RAGatouille itself is moving toward.
```

**When ColBERT wins**: high-precision retrieval where you can afford the storage cost. Good for code search, technical documentation, legal search.

---

## SPLADE: Sparse Learned Embeddings

### The Insight

BM25 uses exact term matching with hand-tuned weights (TF-IDF). What if a neural network learned which terms to match and how to weight them?

```
SPLADE output for "database connection pool":
{
  "database":    2.3,   # high weight
  "connection":  2.1,
  "pool":        1.9,
  "JDBC":        1.4,   # model learned this is related!
  "HikariCP":    1.2,   # model learned this too
  "thread":      0.8,
  "concurrent":  0.6,
  "db":          1.1,   # abbreviation expansion
  # ... thousands of near-zero weights (sparse)
}
```

This is a **sparse vector** (most dimensions = 0) but with **learned neural weights**. It gets BM25's invertibility (fast lookup via inverted index) plus neural term expansion.

### Architecture

```
Input text → BERT encoder → token hidden states
           → linear layer + ReLU + log(1+x) per vocabulary dimension
           → max-pool over token dimension (take max weight per vocab term)
           → sparse vector over vocabulary (30,000 dimensions, ~100 non-zero)
```

### SPLADE vs BM25 vs Dense

```
Query: "why is my java app running out of memory"

BM25:   matches: "java", "app", "memory" → misses OOM-related docs that say "heap space"
Dense:  matches semantically similar → might miss exact error messages

SPLADE: learned expansion →
  "java" → also weights "JVM", "heap"
  "memory" → also weights "heap space", "OutOfMemoryError", "GC overhead"
  Result: retrieves "OutOfMemoryError: GC overhead limit exceeded" exactly
```

**SPLADE trade-offs:**
- Index size: inverted index (sparse), much smaller than ColBERT's token vectors
- Speed: same as BM25 (inverted index lookup) but with neural-quality term weights
- Latency: index-time inference required (run SPLADE on each doc at index time)
- Best available: `naver/splade-cocondenser-ensembledistil`

---

## GraphRAG: Entity Graph Traversal

### The Problem with Chunk-Based Retrieval

```
Question: "What is the relationship between the CEO's compensation policy
           and the company's revenue sharing agreements?"

Chunk retrieval:
  - Retrieves chunks about CEO compensation
  - Retrieves chunks about revenue sharing
  - Neither chunk mentions the other
  - LLM cannot synthesize the connection

Why? The connection is implicit across many documents,
     not explicit in any single chunk.
```

### GraphRAG Architecture (Microsoft Research, 2024; superseded by LazyGraphRAG)

```
Phase 1: Graph Construction (offline)
  ├── Extract entities: [CEO, CompensationPolicy, RevenueSharing, Board]
  ├── Extract relationships: [CEO -governs→ CompensationPolicy,
  │                           CompensationPolicy -linkedTo→ RevenueSharing]
  ├── Build knowledge graph
  └── Run community detection (Leiden algorithm)
       → Community 1: {CEO, CompensationPolicy, BoardDecision}
       → Community 2: {RevenueSharing, Shareholders, Dividends}
       → Generate community summaries with LLM

Phase 2: Query (online)
  Option A: Local search
    → Embed query, find relevant entities in graph
    → Traverse: entity → relationships → connected entities
    → Gather entity-linked text chunks

  Option B: Global search (for synthesis questions)
    → Map: score each community summary for relevance to query
    → Reduce: synthesize across top-scored community summaries
```

### When GraphRAG Wins

```
Wins:
  - "What are the main themes in this corpus?" (global synthesis)
  - "How are these organizations connected?" (relationship queries)
  - Multi-hop reasoning: A→B→C connections across documents
  - When the answer requires connecting multiple entities not co-located

Loses:
  - Simple factual lookup: "What is the CEO's salary?" → chunk retrieval is faster
  - High-velocity corpora: graph rebuilding is expensive
  - Corpora without clear entities: generic text without named entities
```

```python
# Using Microsoft's graphrag library
import graphrag

# Offline indexing
graphrag index --root ./my_project

# Query
graphrag query --root ./my_project --method global \
  "What are the relationships between compensation and revenue sharing?"
```

**Cost**: Original GraphRAG requires many LLM calls for entity extraction and community summarization. Indexing a 1000-document corpus can cost $5-50 in LLM API calls. Not suitable for real-time indexing.

**2026 update — LazyGraphRAG**: Microsoft's successor avoids full upfront entity/relationship extraction and community summarization. It defers most LLM work to query time, using lightweight NLP (noun-phrase extraction) for indexing instead. Reported results: indexing cost roughly on par with plain vector RAG (a small fraction of original GraphRAG's cost) while matching or beating GraphRAG Global Search quality on synthesis-style questions, at a fraction of the query cost. It's being folded into Microsoft's open-source `graphrag` library as the default path — check the `microsoft/graphrag` repo for current integration status before assuming the original (expensive) indexing flow is still the recommended default.

---

## Recursive Document Retrieval: Tree-Based Summarization

### Architecture (RAPTOR pattern)

```
Original documents (leaf nodes)
    ↓ cluster semantically similar chunks
    ↓ summarize each cluster with LLM
Mid-level summaries
    ↓ cluster again
    ↓ summarize
Higher-level summaries
    ↓
Root summary (entire corpus)

All levels embedded and indexed.
```

### Query Routing

```python
# Determine query scope first
scope = llm("Is this question about: (a) a specific detail, (b) a broad topic?")

if scope == "specific":
    # Search leaf nodes (original chunks) → high precision
    results = index.search(query, level="leaves")
else:
    # Search summary nodes → high recall for broad questions
    results = index.search(query, level="summaries")
    # Then drill down for supporting details
    for summary in results:
        details = index.get_children(summary)
```

**When to use**: large corpora where some questions need synthesis ("summarize all the incidents from Q3") and others need specifics ("what was the root cause of incident #47"). RAPTOR handles both with the same index.

---

## Text-to-SQL: When Structured Data Beats Vector Search

### The Problem Vector Search Can't Solve

```
Question: "How many users signed up last week from California?"

Vector search: retrieves documents about users, signups, California
LLM: tries to synthesize a count from document text → will hallucinate

SQL:
  SELECT COUNT(*) FROM users
  WHERE state = 'CA' AND created_at >= CURRENT_DATE - INTERVAL '7 days'
  → Exact, deterministic answer
```

### Text-to-SQL Pipeline

```python
def text_to_sql_rag(user_question: str, db_schema: str) -> dict:
    # Step 1: Generate SQL
    sql = llm(f"""
    Database schema:
    {db_schema}

    Question: {user_question}

    Generate a SQL query. Return only the SQL, no explanation.
    """)

    # Step 2: Validate (safety check)
    if any(keyword in sql.upper() for keyword in ["DROP", "DELETE", "UPDATE", "INSERT"]):
        raise ValueError("Only SELECT queries allowed")

    # Step 3: Execute
    results = db.execute(sql)

    # Step 4: Explain results
    answer = llm(f"""
    Question: {user_question}
    SQL: {sql}
    Results: {results[:10]}  # first 10 rows

    Answer the question based on the query results.
    """)

    return {"answer": answer, "sql": sql, "raw_results": results}
```

### When Text-to-SQL Wins

- Exact aggregations: COUNT, SUM, AVG over structured data
- Time-range queries with specific dates
- Filtering on exact values (user IDs, status codes, enum values)
- Multi-table joins that vector search can't express

**Hard challenges**:
- Schema complexity (100+ table joins)
- Ambiguous column names
- Domain-specific business logic embedded in query semantics
- LLMs generating syntactically valid but semantically wrong SQL

**Mitigations**: few-shot examples of query patterns, chain-of-thought SQL generation, SQL validation before execution, read-only database user.

---

## Lexical + Semantic Hybrid: When to Use BM25 as Primary

### The Standard Hybrid (symmetric)
Most implementations treat BM25 and dense equally, combine with RRF.

### Asymmetric Hybrid: BM25 Primary, Dense as Reranker

```
Use case: technical support search, code search

Pipeline:
  1. BM25 → top-100 candidates (fast, keyword-exact)
  2. Dense rerank → top-10 from candidates (semantic quality)

Why BM25 first:
  - Preserves exact keyword matches (error codes, function names)
  - Inverted index is much faster than ANN for large corpora
  - Dense retrieval can't rank what it never retrieved

Why dense second:
  - "SSL handshake failure" and "TLS certificate error" → same issue, different terms
  - Cross-encoder adds precision over BM25 ranking
```

```python
def hybrid_search_bm25_primary(query: str, top_k: int = 10):
    # Stage 1: BM25 candidates
    bm25_candidates = bm25_index.search(query, top_n=100)

    # Stage 2: Dense rerank
    reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
    pairs = [(query, doc.text) for doc in bm25_candidates]
    scores = reranker.predict(pairs)

    reranked = sorted(zip(bm25_candidates, scores),
                      key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in reranked[:top_k]]
```

---

## The Needle in a Haystack Problem

### Why Long-Context LLMs Still Don't Fully Replace RAG (Updated for 2026 Context Windows)

```
As of 2026, context windows have grown far past 128K:
  - Claude Opus/Sonnet: 1M tokens (GA since March 2026), 200K on older models
  - Gemini 1.5/2.5 Pro: 1M-2M tokens
  - Long-context surcharges are also shrinking/disappearing (Anthropic dropped
    its 2x above-200K pricing premium on current models).

Can I just dump all my docs and skip retrieval entirely? Better than 2024, but still no for most production systems:

1. Cost: even with surcharges gone, 1M tokens of input per query is not free.
   Re-sending a huge context on every query is drastically more expensive than
   retrieving ~2K tokens of relevant chunks, especially at scale/high QPS.
   Prompt caching narrows this gap for repeated queries against the same
   corpus, but doesn't eliminate it for large or frequently-changing corpora.

2. Latency: multi-hundred-K to 1M token inputs still cost meaningful TTFT
   (seconds, not the 10-30s of 2023-era 128K models, but not free) vs
   RAG's 1-2 seconds for a few thousand tokens.

3. "Lost in the Middle" effect persists at longer lengths, though newer
   models handle it better than 2023-era GPT-4:
   Benchmark: Liu et al. 2023 — accuracy drops from 80% to 55% when the
   answer is in the middle of long context vs at the start. Later needle-
   in-a-haystack evals (2024-2026) show meaningfully improved retrieval
   fidelity at 1M-token scale for frontier models, but degradation with
   multiple distractors/needles or reasoning-over-scattered-facts is still
   measurable — it hasn't been fully solved by scaling context alone.

4. No persistence: without RAG, you must re-inject relevant context every
   query (mitigated, not eliminated, by prompt caching)
   vs RAG: corpus embedded once, reused indefinitely, and trivially
   updated incrementally as documents change.

5. Precision/citation: long-context stuffing makes it harder to know which
   part of a huge context produced an answer; RAG's explicit retrieval step
   gives natural provenance/citations.
```

**Updated take**: long-context models have genuinely eaten into RAG's advantage for small-to-medium, mostly-static corpora that fit comfortably in a single context window — for those, "stuff it all in" is increasingly viable and often simpler to operate. RAG remains the better architecture as corpus size grows past what fits economically in context, when the corpus changes frequently, when you need per-query cost/latency control, or when citation/provenance matters. The two approaches are also commonly combined: retrieve a smaller candidate set with RAG, then let a long-context model reason over that larger-than-usual retrieved set.

---

## When Vectorless Approaches Win

| Approach | Best Domains |
|----------|-------------|
| BM25 | Error logs, legal/medical exact terminology, product SKUs |
| ColBERT | Code search, technical docs requiring token-level precision |
| SPLADE | Mix of keyword and semantic needs; good general-purpose upgrade over BM25 |
| GraphRAG | Relationship queries, cross-document synthesis, entity networks |
| Text-to-SQL | Structured/tabular data, aggregations, time-series |
| Recursive summaries | Multi-scale questions over large document collections |
| Hybrid (BM25 primary) | Any domain where exact terminology coexists with paraphrasing |

---

## Concrete Example: Codebase Q&A

**Why BM25 + AST Chunking Beats Pure Embedding Search**

Scenario: developer asks "where is the connection pool initialized in the auth service?"

### Problem with Pure Embedding Search

```python
# Query embedded as concept: "connection pool initialization auth service"
# Embedding captures: pooling, initialization, auth

# Retrieved chunks (embedding-based):
# - Documentation about connection pools (high semantic similarity)
# - Test files that mock connection pools (moderate similarity)
# - The actual initialization code (often LOWER similarity because code != prose)

# Why code gets low similarity:
# "HikariConfig config = new HikariConfig();"
# → embedding space doesn't strongly link this to "connection pool initialization"
# → sparse function names, few natural language tokens
```

### Better Approach: AST Chunking + BM25

```python
import ast
import libcst as cst  # for Python, use tree-sitter for multi-language

def chunk_by_ast(file_path: str) -> list[dict]:
    """Split code into function/class/method chunks with rich metadata."""
    with open(file_path) as f:
        source = f.read()

    tree = ast.parse(source)
    chunks = []

    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.ClassDef)):
            chunk_text = ast.get_source_segment(source, node)
            chunks.append({
                "text": chunk_text,
                "name": node.name,
                "type": type(node).__name__,
                "file": file_path,
                "line": node.lineno,
                # BM25-searchable metadata
                "identifiers": extract_identifiers(node),
            })

    return chunks

def extract_identifiers(node) -> list[str]:
    """Extract all variable names, function calls, imports."""
    names = []
    for child in ast.walk(node):
        if isinstance(child, ast.Name):
            names.append(child.id)
        elif isinstance(child, ast.Attribute):
            names.append(child.attr)
    return names
```

BM25 over `text + identifiers` will rank the actual `initConnectionPool()` method highly for "connection pool initialized" because:
- "connection", "pool" appear in variable names (`connectionPool`, `poolConfig`)
- "init" or "initialize" appears in the function name
- BM25's IDF correctly weights these rare-in-codebase terms highly

### Hybrid Code Search Strategy

```
Index:
  - AST-chunked code with identifier extraction
  - BM25 over: function names + variable names + comments + docstrings
  - Dense embedding over: docstrings + comments (natural language parts)
  - Metadata: file path, class hierarchy, language, module name

Query: "connection pool initialization auth service"
  Step 1: BM25 → find functions with "pool", "init", "connection" in identifiers
  Step 2: Filter by file path matching "auth" or module metadata = "auth"
  Step 3: Dense rerank remaining candidates for semantic alignment
  Step 4: Return function with source + surrounding context (±20 lines)
```

This outperforms pure embedding search because:
1. Code token names are the primary signal — BM25 is designed for token matching
2. AST chunking respects code structure — no function split across chunks
3. Metadata filter (auth service) is exact — vector search would blur module boundaries

---

## Key Gotchas

- **Never deploy pure dense retrieval for code or legal text**: you will miss exact matches that BM25 would find trivially.
- **GraphRAG indexing cost is high**: budget LLM API calls carefully; run entity extraction on a sample first.
- **Text-to-SQL needs a read-only DB user**: LLMs can generate valid DROP/DELETE statements. Always enforce permissions at the database level.
- **ColBERT's storage cost surprises teams**: 65KB/doc vs 3KB/doc means 20x more disk space. Plan ahead.
- **SPLADE requires SPLADE on both sides**: you must encode queries with SPLADE and documents with SPLADE. Mixing with BM25 scores produces incomparable values.
- **Long-context LLM as complement, and increasingly a partial substitute**: with 1M-2M token windows now common (Claude, Gemini) and long-context surcharges dropping, "retrieve fewer, coarser chunks + let long context do more synthesis work" is a valid 2026 pattern for corpora that fit in-window. For corpora that don't fit, or where cost/latency/freshness/citation matter, RAG (or agentic retrieval) is still the better default. Re-evaluate this tradeoff periodically — context windows and pricing are still moving fast.
- **Hybrid search weight tuning is data-dependent**: the RRF k=60 parameter and BM25/dense weights should be validated on held-out queries from your domain, not left at defaults.
