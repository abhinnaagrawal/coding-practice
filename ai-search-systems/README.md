# AI & Search Systems

Live site: **https://abhinnaagrawal.github.io/coding-practice/#/ai-search-systems/**
Top-level site: [Home](/README.md)

Practical notes on the retrieval layer behind RAG, search, and agentic systems. The material starts with how text becomes searchable, then builds from embeddings and ANN indexes into hybrid, graph, and agent-driven retrieval.

## Recommended Reading Order

1. [Text Processing](/ai-search-systems/text-processing.md) - tokenization, normalization, chunking, and analyzers; the foundation for both search and RAG.
2. [Embeddings](/ai-search-systems/embeddings.md) - semantic representation, similarity, model selection, and operational pitfalls.
3. [HNSW](/ai-search-systems/hnsw.md) - how the common low-latency ANN graph index works and how to tune it.
4. [Vector Databases](/ai-search-systems/vector-databases.md) - index choices, filtering, hybrid search, durability, and system selection.
5. [RAG](/ai-search-systems/rag.md) - end-to-end retrieval-augmented generation, reranking, evaluation, and agentic retrieval.
6. [Vectorless RAG](/ai-search-systems/vectorless-rag.md) - where BM25, late interaction, graph traversal, and SQL outperform dense-vector retrieval.

## Mental Model

```
Documents
    |
    v
Text processing and structure-aware chunking
    |
    +--> Lexical index: BM25 / sparse search
    |
    +--> Semantic index: embeddings / ANN
    |                     |
    |                     v
    |                 HNSW or another vector index
    |
    v
Hybrid retrieval, metadata filters, and reranking
    |
    v
Context assembly or an agent retrieval loop
    |
    v
Grounded answer with provenance
```

## What To Retain

- Retrieval recall sets the upper bound for answer quality; generation cannot repair missing evidence.
- Chunking and metadata are product decisions, not preprocessing details.
- Use hybrid retrieval when exact tokens such as identifiers, error codes, dates, or policy language matter.
- Tune recall, latency, cost, freshness, and permission filtering on representative queries rather than relying on generic benchmarks.
- Choose the retrieval primitive to match the data: vectors for semantic similarity, inverted indexes for exact terms, graphs for relationships, and SQL for structured facts.