# Embeddings

## 30-Second Intuition

An embedding is a fixed-length dense vector that encodes semantic meaning. Similar concepts land near each other in vector space because the model was trained to minimize distance between semantically related pairs. You can measure "how related" two things are with a dot product or cosine similarity.

---

## Core Concept: What Is an Embedding?

A traditional one-hot encoding of 50,000 vocabulary words gives you a 50,000-dimensional sparse vector. Useless for similarity. An embedding compresses that into a dense 384–3072 dimensional vector where geometry carries meaning.

```
"database connection pool" → [0.12, -0.87, 0.34, 0.91, ...]  # 768 floats
"JDBC thread pool"         → [0.11, -0.85, 0.36, 0.89, ...]  # very close
"swimming pool"            → [0.78,  0.22, -0.41, 0.05, ...]  # far away
```

Why does this work? Training. The model learned that "database" and "JDBC" appear in similar contexts, that "connection pool" and "thread pool" solve the same problem. The geometry is a byproduct of learning to predict masked tokens or to match question-answer pairs.

---

## How Semantic Similarity Works

Training signal for embedding models typically comes from one of:

1. **Contrastive learning**: push embeddings of positive pairs together, negative pairs apart
2. **MLM (masked language model) pre-training**: forces the model to encode context
3. **Bi-encoder fine-tuning on sentence pairs**: direct similarity signal

```
Training pair: ("dog", "canine")       → should be close
Training pair: ("dog", "automobile")   → should be far

Loss = max(0, margin - sim(anchor, positive) + sim(anchor, negative))
```

After training, the model has internalized a geometry where:
- Synonyms cluster together
- Domain-specific jargon clusters by domain
- Analogical relationships form parallelograms (king - man + woman ≈ queen)

---

## Embedding Models: Trade-offs

| Model | Dims | MTEB(v1) Score | Speed | Cost | Notes |
|-------|------|-----------|-------|------|-------|
| `text-embedding-3-small` (OpenAI) | 1536 | 62.3 | Fast (API) | $0.02/1M tokens | Good default for general use; still OpenAI's current small model as of mid-2026, no successor released |
| `text-embedding-3-large` (OpenAI) | 3072 | 64.6 | Fast (API) | $0.13/1M tokens | Still OpenAI's best embedding model as of mid-2026; no `text-embedding-4` has shipped |
| `text-embedding-ada-002` (OpenAI) | 1536 | 61.0 | Fast (API) | $0.10/1M tokens | Legacy, superseded by `text-embedding-3-*`; avoid for new projects |
| `all-MiniLM-L6-v2` (sentence-transformers) | 384 | 56.3 | Very fast (local) | Free | Great for low-latency local |
| `all-mpnet-base-v2` | 768 | 57.8 | Fast (local) | Free | Better quality than MiniLM |
| `BGE-large-en-v1.5` (BAAI) | 1024 | 64.2 | Medium (local) | Free | Superseded by BGE-M3 (dense+sparse+ColBERT, 100+ languages, 8192 context) as BAAI's flagship |
| `E5-large-v2` | 1024 | 62.2 | Medium (local) | Free | E5 line has stalled since 2024 (latest is `e5-mistral-7b-instruct`); no longer a top pick |
| `nomic-embed-text-v1` | 768 | 62.4 | Fast (local) | Free | Superseded by `nomic-embed-text-v2` (MoE architecture, released early 2025) |
| `mxbai-embed-large-v1` | 1024 | 64.7 | Medium (local) | Free | Was top open-source in 2024; now clearly surpassed by newer models below |
| `Qwen3-Embedding-8B` | up to 4096 | ~70.6 (multilingual MTEB) | Medium (local, 8B params) | Free (open-weight) | 2026 open-weight leader; also ships 0.6B/4B sizes; surpasses OpenAI and older open-source models |
| `gemini-embedding-001` (Google) | up to 3072 (Matryoshka) | 68.3 (English MTEB) | Fast (API) | Paid | Tops the public English MTEB leaderboard as of 2026; supports flexible/truncatable dims |
| `voyage-3.1` / `voyage-3.1-large` (Voyage AI) | varies | Competitive with Gemini | Fast (API) | Paid | Closed-source, narrowed the gap with Gemini Embedding in 2025-2026 |

**Picking heuristics:**
- Proprietary data, don't want to self-host → OpenAI `text-embedding-3-small`, or Google `gemini-embedding-001` for top-tier quality
- Self-hosted, need fast inference → `all-MiniLM-L6-v2` (384d, tiny model)
- Self-hosted, need best quality (2026) → `Qwen3-Embedding-8B` or `BGE-M3`
- Long documents (> 512 tokens) → `nomic-embed-text-v2` or `text-embedding-3-*` with chunking
- Multilingual → `Qwen3-Embedding` or `multilingual-e5-large`

**Critical**: MTEB (Massive Text Embedding Benchmark) scores tell you retrieval quality on standard benchmarks. Your domain may differ — always validate on your own data. **Note**: the scores above for older models are legacy MTEB v1 averages, kept here for relative comparison — MTEB introduced a v2 scoring methodology in 2025-2026 that is not directly comparable, and current leaderboard-topping models (Qwen3-Embedding, Gemini Embedding, Jina v4/v5, NV-Embed-v2) score well above these older numbers. Always check the live leaderboard rather than trusting static scores in any doc, including this one.

---

## Dimensions: Quality vs Cost vs Storage

Higher dimensions = more expressive geometry, but diminishing returns.

```
384d:  float32 → 1536 bytes/vector  →  1M vectors = 1.5 GB
768d:  float32 → 3072 bytes/vector  →  1M vectors = 3.1 GB
1536d: float32 → 6144 bytes/vector  →  1M vectors = 6.1 GB
3072d: float32 → 12288 bytes/vector →  1M vectors = 12.3 GB
```

Quality difference between 768d and 1536d is often < 2% on most tasks. The jump from 384d to 768d is more meaningful. Beyond 1536d, you're paying real money for marginal gains.

OpenAI `text-embedding-3-*` supports **dimension reduction** via the `dimensions` parameter — they trained with Matryoshka Representation Learning, so truncating to 256d or 512d still gives reasonable quality.

---

## Tokenization: Why Word Forms Matter

Embedding models tokenize input before encoding. Most use WordPiece or BPE tokenization:

```
"running"   → ["run", "##ning"]
"runs"      → ["run", "##s"]
"runner"    → ["run", "##ner"]
```

All three share the "run" subword token. So their embeddings will be close — the model has seen this root in similar contexts.

But:
```
"HTTP"       → ["HTTP"]        # single token, common in tech text
"http"       → ["http"]        # might be different token, slightly different embedding
"H", "T", "T", "P" → rare, might be split differently in some tokenizers
```

Practical implication: **case normalization matters**. If your corpus has inconsistent casing, embeddings may be noisier than expected. Most production systems lowercase before embedding.

**Context window limits:**
- Most models: 512 tokens (~380 words)
- `nomic-embed-text-v1`: 8192 tokens
- OpenAI `text-embedding-3-*`: 8192 tokens

Text beyond the limit is **truncated silently** in most libraries. This is a silent quality killer for long documents.

---

## Chunking Strategies

The biggest practical lever on RAG quality. You embed chunks, not whole documents.

### Fixed-Size Chunking
```python
chunks = [text[i:i+512] for i in range(0, len(text), 512)]
# Problem: cuts mid-sentence, mid-concept
```

Bad for anything that has structure. Fast to implement.

### Sentence-Based Chunking
```python
from nltk import sent_tokenize
sentences = sent_tokenize(text)
chunks = []
current = ""
for s in sentences:
    if len(current) + len(s) < MAX_CHARS:
        current += " " + s
    else:
        chunks.append(current)
        current = s
```

Better: preserves semantic units. Problem: variable chunk size.

### Recursive Character Splitting (LangChain default)
Split on `\n\n`, then `\n`, then `.`, then ` ` — tries to preserve structure.
```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,   # overlap prevents losing context at boundaries
    separators=["\n\n", "\n", ".", " "]
)
```

**Overlap is critical**: without overlap, a sentence split across chunks is lost to retrieval.

### Semantic Chunking
Embed each sentence, split where cosine similarity drops sharply between adjacent sentences. Expensive (requires embedding every sentence twice), but produces the most coherent chunks for retrieval.

### Document-Specific Chunking
- **Markdown**: split on `##` headers
- **Code**: split on function/class boundaries using AST
- **PDF**: split on page breaks or paragraph detection
- **Tables**: keep tables as single chunks; never split mid-row

### Parent-Child Chunking
Chunk into small child chunks (128 tokens) for retrieval precision, store parent chunk (512 tokens) for context. Retrieve child, return parent to LLM.

---

## Normalization: L2 Normalize Before Cosine Similarity

Cosine similarity = dot product of L2-normalized vectors.

```
cosine_sim(a, b) = (a · b) / (|a| * |b|)

# If you pre-normalize:
a_norm = a / |a|
b_norm = b / |b|
cosine_sim(a, b) = a_norm · b_norm  # just a dot product now
```

**Why normalize?** Without it, vectors from long documents (which tend to have higher magnitude) would dominate similarity scores even if semantically unrelated.

Most embedding models output pre-normalized vectors (check the docs). If you're using them in a vector database that uses dot product as its metric, pre-normalized vectors give you cosine similarity for free and queries are faster (SIMD dot product is faster than computing norms at query time).

```python
import numpy as np

def normalize(v: np.ndarray) -> np.ndarray:
    return v / np.linalg.norm(v)

# Or batch:
def normalize_batch(vecs: np.ndarray) -> np.ndarray:
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / norms
```

---

## Embedding Drift

Same model, same text → same vector. But:

- **Model version changes**: `text-embedding-ada-002` vs `text-embedding-3-small` produce incomparable vectors. Never mix.
- **Fine-tuning**: if you fine-tune an embedding model, your new model is a different embedding space. You must re-embed your entire corpus.
- **Quantization**: float32 vs int8 quantized models produce slightly different vectors. Mixing them degrades similarity quality.

**Operational rule**: store which model + version produced each embedding. When you upgrade models, re-embed everything and swap atomically.

---

## Multimodal Embeddings (Brief)

Models like **CLIP** (OpenAI), **ImageBind** (Meta), and **Nomic Embed Vision** create a **shared embedding space** for text and images. An embedding of the text "a dog running in the park" lands near an embedding of an actual photo of that scene.

```
CLIP:
  "golden retriever"  → [0.34, -0.12, ...]
  [image of golden retriever] → [0.33, -0.11, ...]  # very close
  [image of cat]              → [0.10, 0.45, ...]   # far
```

Used for: image search, cross-modal retrieval, content moderation. Architecture: dual encoder trained with contrastive loss on image-caption pairs.

---

## Deep Internals: How Transformer Embeddings Are Generated

For a sentence-transformer model:

```
Input: "database connection pool"
   ↓
Tokenize: [CLS] database connection pool [SEP]
   ↓
Token embeddings + positional embeddings
   ↓
N transformer layers (attention + FFN)
   ↓
Hidden states: shape (seq_len, hidden_dim)
   ↓
Pooling strategy:
  - Mean pooling: average all token hidden states  ← most common
  - CLS pooling: use only [CLS] token
  - Max pooling: element-wise max
   ↓
Single vector: shape (hidden_dim,)
   ↓
Optional: linear projection to output dim
   ↓
L2 normalize
   ↓
Final embedding: shape (output_dim,)
```

**Mean pooling vs CLS**: Mean pooling tends to outperform CLS on retrieval tasks because it incorporates signal from all tokens.

---

## Concrete Example: "database connection pool"

Why is it close to "JDBC thread pool" but far from "swimming pool"?

Training corpus statistics:
- "database connection pool" appears near: "max connections", "pool size", "connection leak", "JDBC", "thread pool", "HikariCP", "datasource"
- "JDBC thread pool" appears near: same technical context
- "swimming pool" appears near: "chlorine", "diving board", "swim lane", "outdoor", "backyard"

The model learned different **contextual neighborhoods** for each usage of "pool." The homograph disambiguation happens because the surrounding words in training examples were consistently different. The geometry reflects co-occurrence statistics, not string similarity.

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("all-mpnet-base-v2")

vecs = model.encode([
    "database connection pool",
    "JDBC thread pool",
    "swimming pool",
    "connection pooling in HikariCP",
])

# Cosine similarities (vectors are pre-normalized by sentence-transformers)
sims = vecs @ vecs.T
# database connection pool vs JDBC thread pool   → ~0.82
# database connection pool vs swimming pool      → ~0.18
# database connection pool vs HikariCP pooling   → ~0.88
```

---

## Picking an Embedding Model for a New Project

```
1. What's my deployment constraint?
   - Can't call external APIs → sentence-transformers locally
   - Fine with managed API → OpenAI or Cohere

2. What's my corpus domain?
   - General English text → any of the top models
   - Code → CodeBERT, unixcoder, or nomic-embed-code
   - Multi-language → multilingual-e5-large
   - Medical/legal → domain-specific models (BioLinkBERT, legal-bert)

3. What's my document length?
   - < 512 tokens → any model
   - > 512 tokens → nomic-embed-text-v2 or text-embedding-3-* with chunking

4. What's my latency/throughput constraint?
   - < 5ms embedding time → MiniLM-L6 (384d, tiny)
   - Batch offline → BGE-M3 or Qwen3-Embedding
   - Real-time API → text-embedding-3-small or gemini-embedding-001

5. Validate on your data:
   - Take 100 sample queries + known relevant docs
   - Compute recall@10 for each candidate model
   - Pick the model that maximizes recall, not MTEB score
```

---

## Key Gotchas

- **Truncation is silent**: text longer than model's context window is truncated with no warning in most libraries. Always chunk first.
- **Don't mix model versions**: incompatible embedding spaces. Always track model name + version per vector.
- **Instruction prefixes**: E5 and BGE models require prefixes like `"query: "` and `"passage: "` for asymmetric retrieval. Omitting them tanks recall.
- **Batch size matters for throughput**: embedding one sentence at a time vs batches of 64 can be 20x slower.
- **Float32 vs float16**: float16 cuts storage in half with < 0.1% quality loss on most models. Consider it for production.
- **Cosine similarity ≠ relevance threshold**: a score of 0.7 might mean "highly relevant" for one model and "loosely related" for another. Calibrate thresholds per model empirically.
- **Empty strings / very short inputs**: embedding a single word produces a noisy vector. Set minimum chunk size.
