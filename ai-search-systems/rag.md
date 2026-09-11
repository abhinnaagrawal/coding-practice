# RAG: Retrieval-Augmented Generation

## 30-Second Intuition

RAG = retrieve relevant context from a corpus → stuff it into the LLM prompt → generate an answer grounded in that context. The LLM's parametric memory is unreliable for private/recent data; retrieval grounds it. The quality of a RAG system is bottlenecked by whichever step is weakest: chunking, embedding, retrieval, or generation.

---

## Architecture Overview

```
User Query
    │
    ▼
[Query Processing]  ← optional: rewrite, expand, HyDE
    │
    ▼
[Retrieval]         ← vector search + optional BM25 hybrid
    │
    ▼
[Reranking]         ← optional cross-encoder reranker
    │
    ▼
[Context Assembly]  ← filter, compress, deduplicate chunks
    │
    ▼
[Generation]        ← LLM with retrieved context in prompt
    │
    ▼
Answer + Sources
```

Each arrow is a potential failure point. Most production failures are in retrieval, not generation.

---

## Why Each Step Matters

### Retrieval determines the ceiling
The LLM cannot generate a correct answer if the relevant chunk was not retrieved. LLMs rarely hallucinate when given correct context; they almost always fail when given no context or wrong context.

### Chunking determines retrieval quality
A chunk that splits a key concept across two pieces will never score highly for either query. A chunk that's too large contains noise that dilutes the relevance signal.

### Generation is the floor
Even with perfect retrieval, a bad prompt template, context window overflow, or model confusion between sources will produce bad output.

---

## Naive RAG Pitfalls

```
Naive pipeline:
  chunk(fixed_size=512) → embed → store → embed_query → top_k → stuff_into_prompt → generate
```

**Problems:**

1. **Fixed-size chunking splits concepts**: "The server rejects connections when the connection pool is exhausted. This is controlled by max_pool_size." — split across chunks → neither chunk answers "why is my server rejecting connections?"

2. **Missing context in chunk**: A chunk says "This was introduced in version 3.2." — but what "this" refers to is in the previous chunk. Retrieved alone, it's useless.

3. **Retrieval-generation mismatch**: User asks a comparative question ("how does X compare to Y?"). Top-k returns 5 chunks about X and 0 about Y. The LLM has no basis for comparison.

4. **Context window stuffing**: Retrieving top-20 chunks pushes total context to 15K tokens. LLM loses focus. "Lost-in-the-middle" effect: LLMs attend to start and end of context, forget middle.

5. **Stale embeddings**: Document updated but embedding not refreshed. Old content returned.

6. **Query-document asymmetry**: User queries are short and colloquial. Documents are long and formal. Embedding space doesn't align them well without instruction prefixes or query expansion.

---

## Advanced RAG Patterns

### HyDE: Hypothetical Document Embeddings

**Problem**: user query ("why is my connection pool exhausted?") is short and query-like. Documents are long and answer-like. They live in different parts of embedding space.

**Solution**: generate a hypothetical answer first, embed that, use it as the query vector.

```python
# Step 1: Generate hypothetical answer
hypothetical = llm("Write a passage that answers: 'why is my connection pool exhausted?'")
# → "Connection pool exhaustion occurs when all available connections
#    are in use and no connections are being returned. Common causes
#    include connection leaks, insufficient pool size, slow queries
#    holding connections..."

# Step 2: Embed the hypothetical answer (now doc-like, not query-like)
query_vec = embed(hypothetical)

# Step 3: Retrieve using this vector
results = vector_db.search(query_vec, top_k=5)
```

**When HyDE wins**: document-heavy corpora where queries are much shorter than relevant passages. Measured gains of 5-15% recall.

**When HyDE fails**: short factual lookups ("what is the capital of France?"), when the LLM generates a plausible but wrong hypothetical.

### Query Expansion / Rewriting

```python
# Rewrite query for better retrieval characteristics
rewrites = llm(f"""
Generate 3 different phrasings of this question for semantic search:
"{user_query}"
Return as JSON list.
""")

# Retrieve for each, deduplicate, merge
all_results = []
for query in rewrites + [user_query]:
    all_results.extend(vector_db.search(embed(query), top_k=5))

final_results = deduplicate_by_chunk_id(all_results)
```

**When useful**: technical queries where synonyms matter ("OOM error" vs "out of memory" vs "memory exhaustion"). Multi-aspect queries.

### Multi-Query Retrieval

Decompose complex questions into sub-questions, retrieve for each independently:

```python
sub_questions = llm(f"""
Decompose "{user_query}" into 2-4 independent sub-questions
that together answer the original.
""")
# "Why did deployment X fail?" →
# ["What errors appeared in deployment X logs?",
#  "What changed between the last successful and failed deployment?",
#  "Were there any infrastructure events during deployment X?"]

results = {}
for sq in sub_questions:
    results[sq] = vector_db.search(embed(sq), top_k=3)
```

### Contextual Compression

After retrieval, the LLM compresses/filters each chunk to only the relevant portion:

```python
compressed_chunks = []
for chunk in retrieved_chunks:
    compressed = llm(f"""
    Question: {user_query}
    Document: {chunk}

    Extract only the sentences directly relevant to the question.
    Return empty string if nothing is relevant.
    """)
    if compressed:
        compressed_chunks.append(compressed)
```

**Benefit**: reduces noise and context window usage. **Cost**: adds LLM calls (latency + cost). Use when precision is critical and you have latency budget.

### Parent-Child Chunking

```
Index structure:
  Parent chunk (512 tokens): stores full context
    ├── Child chunk A (128 tokens): indexes for retrieval
    ├── Child chunk B (128 tokens): indexes for retrieval
    └── Child chunk C (128 tokens): indexes for retrieval

At query time:
  1. Embed query
  2. ANN search against CHILD chunk embeddings (small = precise match)
  3. When child chunk matches, return its PARENT chunk to LLM

Result: precision of small chunks + context richness of large chunks
```

```python
# LlamaIndex implementation pattern
from llama_index.node_parser import HierarchicalNodeParser

parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]  # parent → mid → child
)
nodes = parser.get_nodes_from_documents(documents)

# During retrieval: retrieve child, get parent
child_node = retrieve(query)
parent_node = index.get_node(child_node.parent_node.node_id)
context = parent_node.text
```

### Sentence Window Retrieval

Similar to parent-child but simpler:

```
Embed individual sentences.
At retrieval: return sentence + N sentences before/after.

Example: sentence_window_size=2
Retrieved sentence: "Connection pool exhausted error occurred."
Returned context: [2 before] + sentence + [2 after]
= "The API gateway returned 503 errors. Database response times exceeded 5 seconds. Connection pool exhausted error occurred. All 50 connections were in use. New requests were queued until timeout."
```

---

## Reranking: Why It Improves Precision

**Bi-encoder** (standard embedding retrieval):
```
embed(query) → vector
embed(doc)   → vector
score = cosine_sim(query_vec, doc_vec)

Fast: O(1) per doc after indexing
Problem: each encoded independently, no cross-attention
```

**Cross-encoder** (reranker):
```
input: [query, doc] concatenated
run through BERT-style model with full cross-attention
output: relevance score

Slow: O(N) per query for N candidates (must process each pair)
Advantage: sees both together → much more accurate relevance signal
```

**Strategy**: retrieve top-50 with bi-encoder, rerank with cross-encoder, return top-10.

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

# After initial retrieval
candidates = vector_db.search(query_embedding, top_k=50)

# Rerank
pairs = [(query, chunk.text) for chunk in candidates]
scores = reranker.predict(pairs)

# Sort by cross-encoder score
reranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
top_10 = [doc for doc, _ in reranked[:10]]
```

**Popular rerankers:**
- `cross-encoder/ms-marco-MiniLM-L-6-v2` — fast, good quality, open source
- `BAAI/bge-reranker-v2-m3` — current best general-purpose open-source reranker (568M params); `bge-reranker-v2.5-gemma2-lightweight` for higher quality at higher cost
- Cohere Rerank — managed API, now on **Rerank 4** (`rerank-v4.0-pro` / `rerank-v4.0-fast`), superseding Rerank 3.5
- Jina Reranker v2 — good multilingual

**Reranking gains**: typically 5-15% improvement in NDCG@10 over bi-encoder alone. More important when first-stage retrieval is noisy (large corpus, broad embedding model).

---

## RAG Evaluation: Ragas Framework

Ragas (Retrieval Augmented Generation Assessment) pioneered these 4 core metrics; the framework has since expanded to 6-12 metrics (adding context entity recall, answer correctness, answer similarity, aspect critique, etc.), but the original 4 remain the most commonly used and are the best starting point:

### 1. Faithfulness
Does the answer make claims supported by the retrieved context?

```
Retrieved context: "HikariCP default connection pool size is 10."
Answer: "HikariCP defaults to 10 connections."     → faithful
Answer: "HikariCP defaults to 100 connections."    → not faithful (hallucinated)

Score: fraction of answer claims that are supported by context
```

### 2. Answer Relevancy
Does the answer address the question asked?

```
Question: "What is the default HikariCP pool size?"
Answer: "HikariCP is a high-performance connection pool library written in Java."
→ not relevant (didn't answer the question)

Measured by: embed question, embed answer, cosine similarity
```

### 3. Context Precision
Of the retrieved chunks, how many are actually relevant?

```
Retrieved: [chunk_1_relevant, chunk_2_irrelevant, chunk_3_relevant, ...]
Context Precision = (relevant chunks) / (total retrieved chunks)
```

Low context precision = too much noise in the prompt.

### 4. Context Recall
Of the information needed to answer, how much was in the retrieved context?

```
Ground truth answer: "A + B + C"
Retrieved context contains: A and B (not C)
Context Recall = 2/3
```

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall
from datasets import Dataset

test_data = Dataset.from_dict({
    "question": questions,
    "answer": generated_answers,
    "contexts": retrieved_contexts,        # list of lists
    "ground_truth": reference_answers,     # for recall
})

results = evaluate(test_data, metrics=[
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
])
```

---

## RAG vs Fine-Tuning: When to Choose Each

| Factor | RAG | Fine-tuning |
|--------|-----|------------|
| Data freshness | New docs = just embed and index | Requires retraining |
| Private/proprietary data | Natural fit (data stays in VDB) | Training data exposure risk |
| Dynamic corpus | Excellent (add docs in real time) | Poor (static knowledge) |
| Specific style/format | Limited (prompt engineering) | Excellent |
| Domain-specific reasoning patterns | Limited | Excellent |
| Cost | Low (embedding + retrieval) | High (training) |
| Interpretability | High (can see retrieved context) | Low (why did it answer that?) |
| Latency | Adds retrieval latency (50-200ms) | No retrieval overhead |

**Use RAG when**: you have a large, dynamic corpus of private documents, and the primary challenge is "knowing" the right facts.

**Use fine-tuning when**: you need the model to behave differently (style, format, reasoning pattern), not just know different facts. Example: fine-tune to always output structured JSON, always follow a specific reasoning chain, or handle domain-specific language patterns.

**Use both**: fine-tune for behavior, RAG for knowledge. This is the production pattern at serious deployments.

---

## Agentic RAG

Instead of a fixed retrieve-then-generate pipeline, retrieval becomes a tool call in an agent loop:

```python
tools = [
    Tool("search_docs", "Search the documentation for relevant information"),
    Tool("search_code", "Search the codebase for relevant code examples"),
    Tool("search_metrics", "Query observability system for metrics"),
]

# Agent decides when and what to retrieve
agent_loop:
    plan = llm("What do I need to answer this question?")

    while not satisfied:
        action = llm("What tool should I call next?")
        result = execute_tool(action)
        context += result

        if llm("Do I have enough to answer?"):
            break

    return llm("Generate final answer from context")
```

**Benefits**:
- Agent can do multi-hop reasoning: retrieve → realize need more → retrieve again
- Can use different retrieval tools for different sub-questions
- Can verify retrieved information by cross-checking

**Costs**:
- Latency: each LLM call adds 500ms-2s
- Cost: many LLM calls per query
- Reliability: agent loops can get stuck or go off-track

**When agentic RAG makes sense**: complex analytical questions that require multiple retrieval steps, questions requiring information synthesis across heterogeneous sources.

**2026 note**: "Agentic RAG" is now the industry-standard framing for this pattern (retrieval as a tool call inside an agent loop, deciding whether/what/when to retrieve), not a niche variant. Some vendors are also describing a further evolution — "context architecture" / "agentic search" — where an agent with tool access (file search, code search, DB query) replaces a fixed vector-DB pipeline entirely, especially for codebases and structured corpora. Anthropic has published on using agentic search instead of embedding-based RAG for Claude Code's own codebase retrieval. Treat this as the current end of the spectrum that started with naive RAG → hybrid RAG → agentic RAG.

---

## Concrete Pipeline: "Why Did Deployment X Fail?"

```
User query: "Why did deployment X fail at 2:30pm yesterday?"

Step 1: Query rewriting
  Rewrites:
    - "deployment X failure root cause 2024-01-15 14:30"
    - "errors deployment X yesterday afternoon"
    - "what went wrong deployment X"

Step 2: Multi-source retrieval
  Source A: deployment logs index (BM25 + vector hybrid)
    → chunks from deployment X log files around 14:30
  Source B: incident documentation
    → any past incidents related to similar patterns
  Source C: runbook index
    → relevant troubleshooting procedures

Step 3: Reranking
  Cross-encoder on top-30 results → top-10 most relevant

Step 4: Context assembly
  Sort by timestamp (for logs, temporal order matters)
  Deduplicate overlapping chunks
  Truncate to fit 4K context window

Step 5: Generation prompt
  System: "You are an SRE assistant. Answer based only on provided context."
  Context: [top-10 chunks in temporal order]
  Question: "Why did deployment X fail at 2:30pm yesterday?"

Step 6: Answer with citations
  "The deployment failed due to a database migration timeout [source: deploy_log_line_847].
   The migration was attempting to add an index to a 50M row table [source: migration_script_v3.sql].
   This caused connection pool exhaustion [source: app_error_log_14:31:22]."
```

---

## Key Failure Modes

1. **Hallucination despite retrieval**: LLM ignores retrieved context and uses parametric memory. Mitigation: explicit prompt instruction, lower temperature, add "ONLY use the provided context" to system prompt.

2. **Context window stuffing**: retrieving top-20 at 500 tokens each = 10K tokens of context. LLM performance degrades in the middle. Mitigation: rerank + compress, use parent-child chunking to get denser context.

3. **Stale embeddings**: document updated but not re-indexed. Mitigation: track document hash + embedding timestamp, trigger re-embedding on document update.

4. **Off-topic retrieval poisons answer**: a vaguely similar but wrong chunk gets retrieved and confuses the LLM. Mitigation: apply similarity threshold (don't include chunks below 0.6 cosine similarity), use reranking to filter noise.

5. **Query reformulation error**: HyDE or query expansion generates a misleading reformulation that retrieves wrong documents. Mitigation: include original query in retrieval alongside reformulations.

6. **Chunk boundary kills answer**: the exact answer spans two adjacent chunks, neither of which scores highly alone. Mitigation: overlapping chunking (50-100 token overlap), or sentence window retrieval.

---

## Key Gotchas

- **Retrieval recall is the primary metric to optimize**. Faithfulness is downstream of recall. If the right chunk never comes back, nothing else matters.
- **Chunk size is a hyperparameter**: there's no universal right answer. 256 tokens for QA over dense technical docs, 512+ for narrative text.
- **Reranking is almost always worth the latency**: 100ms cross-encoder call that improves precision by 15% is worth it.
- **Metadata filtering before retrieval saves latency and improves quality**: filter by date range, document type, or user permissions before ANN search.
- **Citation hallucination**: LLMs sometimes cite the right source for the wrong claim, or swap which source said what. Validate citations programmatically in high-stakes use cases.
- **Temperature for RAG generation should be low (0.0-0.3)**: you want faithful extraction, not creative interpretation.
