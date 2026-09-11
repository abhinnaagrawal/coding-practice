# Text Processing for AI/ML and Search Systems

> **Audience**: Senior backend/distributed systems engineers learning AI/ML and search internals.
> **Goal**: Fast recall + deep intuition. Production reference.

---

## 30-Second Intuition

Raw text is chaos. Every downstream system — BM25, embeddings, LLMs — fails or degrades when given raw chaos. Text processing is the discipline of converting that chaos into a consistent, queryable representation.

The mental model:

```
Raw Text
  │
  ▼
[CharFilters]          ← HTML strip, char mapping
  │
  ▼
[Tokenizer]            ← split into tokens (words, subwords, chars)
  │
  ▼
[TokenFilters]         ← lowercase, stop words, stem, synonyms
  │
  ▼
Clean Token Stream
  │
  ├──▶ Inverted Index (BM25)     ← lexical retrieval
  ├──▶ Embedding Model           ← vector retrieval
  └──▶ LLM Context               ← generation
```

**The key insight**: BM25 and embeddings need *very different* preprocessing. BM25 needs stemming, stop word removal, aggressive normalization. Embeddings need minimal preprocessing — the model handles morphology internally. Mixing these up is the #1 production mistake.

---

## 1. Why Text Processing Matters

### The Noise Problem

Raw text contains variations that should be semantically equivalent:

```
"Running shoes"    # capital R, present participle
"running shoes"    # lowercase
"RUNNING SHOES"    # all caps
"ran in shoes"     # past tense, different word
"shoe for running" # word order changed
"run shoe"         # stemmed form
```

A naive string-match system sees these as 6 different things. A well-processed system collapses them appropriately.

### Downstream Impact

| System | What breaks without preprocessing |
|--------|----------------------------------|
| BM25 | "running" ≠ "run" → missed recall |
| TF-IDF | "the" gets high TF → noisy scores |
| Embeddings | Truncated input → information loss |
| LLMs | Malformed HTML in prompt → wasted tokens |
| Autocomplete | No edge n-grams → no prefix matching |

### The Pipeline

```
Raw Text
  │
  ├── For BM25: tokenize → normalize → stem → remove stop words → index
  │
  ├── For Embeddings: chunk → (minimal normalize) → embed → store vector
  │
  └── For LLMs: chunk → enrich with metadata → build prompt
```

The pipeline choices cascade: a bad chunking decision for embeddings cannot be fixed by tuning retrieval.

---

## 2. Tokenization

Tokenization is the act of splitting a text stream into discrete units (tokens) that can be processed, indexed, or fed to a model.

### Levels of Tokenization

```
Character:  ["h","e","l","l","o"," ","w","o","r","l","d"]
Word:       ["hello", "world"]
Subword:    ["hell", "##o", "world"]          ← WordPiece style
Byte-level: [b"hel", b"lo", b" wo", b"rld"]  ← BPE on bytes
```

### Whitespace Tokenization (Naive)

```python
text = "I don't like New York's weather"
tokens = text.split()
# ["I", "don't", "like", "New", "York's", "weather"]
```

Problems:
- `"don't"` stays as one token — downstream systems may not know it's `"do"` + `"not"`
- `"New York"` splits — loses the named entity
- Punctuation sticks to words: `"weather."` ≠ `"weather"`

### Word Tokenization (NLTK)

```python
from nltk.tokenize import word_tokenize
tokens = word_tokenize("I don't like New York's weather.")
# ["I", "do", "n't", "like", "New", "York", "'s", "weather", "."]
```

Better: handles contractions and punctuation. Still splits multi-word expressions.

### Subword Tokenization

This is the dominant approach for LLMs. The core problem it solves:

**OOV (Out-of-Vocabulary) Problem**: Word-level vocabularies can't handle:
- New words (`"GPT-4o"`)
- Misspellings (`"accomodation"`)
- Rare words (`"defenestration"`)
- Morphological variants (`"unbelievableness"`)

Subword tokenization decomposes unknown words into known pieces.

---

#### BPE (Byte Pair Encoding)

Used by: GPT-2, GPT-3, GPT-4, LLaMA, RoBERTa

**Algorithm**: Start with individual characters. Iteratively merge the most frequent adjacent pair.

**Step-by-step example** — vocabulary starts from: `["lower", "lowest", "newer", "wider"]`

```
Step 0 (character vocab, space-separated, _ = end of word):
  l o w e r _
  l o w e s t _
  n e w e r _
  w i d e r _

Count all adjacent pairs:
  (l,o): 2   (o,w): 2   (w,e): 3   (e,r): 3   (r,_): 3
  (e,s): 1   (s,t): 1   (t,_): 1   (n,e): 1   (w,i): 1
  (i,d): 1   (d,e): 1

Step 1: Merge most frequent pair → (e,r) → "er"
  l o w er _
  l o w e s t _
  n e w er _
  w i d er _

Step 2: Merge → (er,_) → "er_"
  l o w er_
  l o w e s t _
  n e w er_
  w i d er_

Step 3: Merge → (w,e) → "we" (count: 3 before, now 2 after step 2 reduction)
  Actually merge (l,o) → "lo" (count: 2)
  lo w er_
  lo w e s t _
  n e w er_
  w i d er_

... continue until vocabulary size reached
```

Final vocabulary might include: `["lo", "w", "er_", "e", "s", "t_", "n", "we", "wi", "d", ...]`

At inference: unknown word `"lower"` → look up `["lo", "w", "er_"]` (all known).

**Key property**: BPE uses frequency. Common substrings become single tokens. Rare words decompose further.

---

#### WordPiece (BERT)

Used by: BERT, DistilBERT, ALBERT

Similar to BPE but uses **likelihood** instead of frequency. At each step, merge the pair that maximizes the language model likelihood of the training data.

```python
# WordPiece output for BERT:
from transformers import BertTokenizer
tok = BertTokenizer.from_pretrained("bert-base-uncased")
tok.tokenize("transformer-based RAG systems")
# ["transformer", "-", "based", "rag", "systems"]

tok.tokenize("unbelievableness")
# ["un", "##believe", "##able", "##ness"]
# "##" prefix means: this subword continues a previous token (no space before it)
```

The `##` notation distinguishes `"##ing"` (suffix) from `"ing"` (standalone word).

---

#### SentencePiece

Used by: T5, LLaMA, ALBERT, XLNet, mT5

**Key difference**: language-agnostic. Treats the raw Unicode byte stream including whitespace as input. No pre-tokenization step needed.

```python
import sentencepiece as spm
# Whitespace is treated as a character: "▁" (U+2581) marks word starts
# "Hello world" → ["▁Hello", "▁world"]
# "Hello  world" → ["▁Hello", "▁", "▁world"]  # double space preserved
```

This matters for:
- Languages without spaces (Chinese, Japanese, Thai)
- Code (whitespace is semantically significant)
- Documents where spacing is structural

---

#### Unigram Language Model

Used by: SentencePiece (alternative training mode), XLNet

Instead of building up from merges, start with a large vocabulary and prune. Each token gets a log-probability. Tokenization = find the segmentation that maximizes `sum(log P(token))`.

```
"unhappiness" might segment as:
  ["un", "happiness"]    score: -2.1 + -0.8 = -2.9   ← chosen
  ["unhappy", "ness"]    score: -3.5 + -1.2 = -4.7
  ["u", "n", "happy", "ness"]  score: much worse
```

**Key property**: Multiple valid tokenizations exist; the model picks the most probable one during training. At inference, Viterbi decoding finds the optimal segmentation.

---

#### Token IDs

Every tokenizer maps tokens → integer IDs via a vocabulary file:

```python
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained("gpt2")
ids = tok.encode("Hello world")
# [15496, 995]
tok.decode([15496, 995])
# "Hello world"

# Special tokens:
# [CLS] = 101 (BERT), <s> = 1 (LLaMA), <|endoftext|> = 50256 (GPT-2)
# [PAD] = 0, [SEP] = 102, [MASK] = 103 (BERT)
```

---

#### The Tokenization Gotcha: Position-Dependent Tokenization

```python
tok = BertTokenizer.from_pretrained("bert-base-uncased")

# "running" at start of sentence (after [CLS]):
tok.tokenize("Running is good")
# ["running", "is", "good"]

# "running" after punctuation or in different context:
tok.tokenize("I love running")
# ["i", "love", "running"]

# GPT-2 (BPE on bytes) — space changes the token ID:
gpt2_tok = AutoTokenizer.from_pretrained("gpt2")
gpt2_tok.encode("dog")     # [9703]
gpt2_tok.encode(" dog")    # [3290]  ← DIFFERENT TOKEN!
# This is because " dog" (with leading space) is a different BPE merge
```

**Implication for RAG**: When you split a document into chunks, the first token of a chunk might tokenize differently than it would in context. This is usually negligible but matters at very fine-grained analysis.

---

### Tokenization in Search (Lucene/OpenSearch)

In search systems, "tokenizer" means something different from ML tokenizers:

```
Char Stream → [Tokenizer] → Token Stream → [TokenFilters] → Final Tokens
                                │
                        (word boundaries,
                         position info,
                         start/end offsets)
```

Each token carries:
- `term`: the string value
- `position`: word position in document (for phrase queries)
- `startOffset/endOffset`: byte offsets in original text (for highlighting)
- `type`: `<ALPHANUM>`, `<NUM>`, etc.

```
Input: "Transformer-based RAG systems"
Standard Tokenizer output:
  Token 1: term="Transformer", position=0, start=0, end=11
  Token 2: term="based", position=1, start=12, end=17
  Token 3: term="RAG", position=2, start=18, end=21
  Token 4: term="systems", position=3, start=22, end=29
```

---

### BPE Step-by-Step: "transformer-based RAG systems"

Using GPT-2 tokenizer (BPE):

```python
from transformers import GPT2Tokenizer
tok = GPT2Tokenizer.from_pretrained("gpt2")
tokens = tok.tokenize("transformer-based RAG systems")
# ['transform', 'er', '-', 'based', ' RAG', ' systems']
# IDs: [35636, 263, 12, 3021, 32735, 3341]

# Note: "transformer" → ["transform", "er"] (not a single token in GPT-2!)
# " RAG" (with space) is ONE token
# Token count: 6 tokens for 4 "words"
```

Why "transformer" splits: GPT-2 trained on pre-2019 text; "transformer" wasn't common enough to be one BPE unit. GPT-4's tokenizer would likely have it as one token.

---

## 3. Stemming vs Lemmatization

Both address **morphological normalization**: collapsing inflected forms to a base form.

```
Inflected forms of "run":
  run, runs, ran, running, runner, runners

Inflected forms of "good":
  good, better, best  ← suppletive (completely different stems!)
```

### Stemming: Rule-Based Suffix Stripping

Fast, crude, produces stems that may not be real words.

#### Porter Stemmer (The Classic)

5-phase cascade of rewrite rules:

```
Phase 1a: Plurals and past participles
  SSES → SS:   "caresses" → "caress"
  IES  → I:    "ponies"   → "poni"
  SS   → SS:   "caress"   → "caress"
  S    →  :    "cats"     → "cat"

Phase 1b: -eed, -ing, -ed
  ATING → ATE: "relating" → "relate" → "relat"
  ...

Phase 2: -ational, -tional, -enci, -anci, ...
  "relational" → "relate"
  "conditional" → "condition"

Phase 3: -icate, -ative, -alize, ...
  "electriciti" → "electric"

Phase 4: -ement, -ment, -ence, -ance, -ism, ...
  "allowance" → "allow"

Phase 5a: Final -e removal
  "probate" → "probat"
  "rate" → "rate" (protected by rule)

Phase 5b: Double consonant → single
  "controlling" → "control"
```

```python
from nltk.stem import PorterStemmer
ps = PorterStemmer()

ps.stem("running")    # "run"
ps.stem("studies")    # "studi"    ← NOT a real word!
ps.stem("generously") # "generous"
ps.stem("maximum")    # "maximum"  (some words don't change)
ps.stem("presumably") # "presum"
```

**Why non-real-word stems are fine for search**:
```
Index: "studies" → stem → "studi" → stored in inverted index
Query: "study"   → stem → "studi" → looks up "studi" → MATCH

The stem is just a key. Nobody reads it. It just needs to be consistent.
```

#### Snowball (Porter2)

Improved Porter. Better handling of edge cases, more languages:

```python
from nltk.stem.snowball import SnowballStemmer
ss = SnowballStemmer("english")
ss.stem("running")    # "run"
ss.stem("generously") # "generous"  ← better than Porter

# Also supports:
# "danish", "dutch", "finnish", "french", "german",
# "hungarian", "italian", "norwegian", "portuguese",
# "romanian", "russian", "spanish", "swedish"
```

#### Lancaster (Aggressive)

```python
from nltk.stem import LancasterStemmer
ls = LancasterStemmer()
ls.stem("generously") # "gen"    ← very aggressive!
ls.stem("maximum")    # "maxim"
ls.stem("running")    # "run"
```

Lancaster achieves higher conflation (more words map to same stem) but risks over-stemming: `"general"` and `"generous"` might both → `"gen"`, destroying distinction.

---

### Lemmatization: Dictionary-Based Morphological Analysis

Returns the **canonical dictionary form** (lemma). Requires knowing the part-of-speech.

```python
import spacy
nlp = spacy.load("en_core_web_sm")

doc = nlp("The meeting was better than running in the rain")
for token in doc:
    print(f"{token.text:15} POS:{token.pos_:6} lemma:{token.lemma_}")

# The             POS:DET    lemma:the
# meeting         POS:NOUN   lemma:meeting   ← noun stays "meeting"
# was             POS:AUX    lemma:be        ← knows "was" = past of "be"
# better          POS:ADJ    lemma:good      ← knows "better" = comparative of "good"!
# than            POS:ADP    lemma:than
# running         POS:VERB   lemma:run       ← verb "running" → "run"
# in              POS:ADP    lemma:in
# the             POS:DET    lemma:the
# rain            POS:NOUN   lemma:rain
```

**Why POS tagging matters**:
```
"meeting" as NOUN → lemma: "meeting"   (a meeting = a gathering)
"meeting" as VERB → lemma: "meet"      (I am meeting you)

"saw" as NOUN     → lemma: "saw"       (the tool)
"saw" as VERB     → lemma: "see"       (past tense of see)
```

Without POS tagging, lemmatizers use heuristics and get these wrong.

#### WordNet Lemmatizer (NLTK)

```python
from nltk.stem import WordNetLemmatizer
from nltk.corpus import wordnet

wnl = WordNetLemmatizer()
# Must specify POS:
wnl.lemmatize("running", pos=wordnet.VERB)  # "run"
wnl.lemmatize("running", pos=wordnet.NOUN)  # "running"
wnl.lemmatize("better",  pos=wordnet.ADJ)   # "good"
wnl.lemmatize("better")                     # "better" (default=NOUN, wrong!)
```

---

### Stemming vs Lemmatization: The Trade-Off

| Property | Stemming | Lemmatization |
|----------|----------|---------------|
| Speed | Very fast (~microseconds) | Slow (requires NLP pipeline) |
| Output | May not be real word | Always real word |
| Accuracy | Lower | Higher |
| OOV handling | Works (rule-based) | Fails (not in dictionary) |
| Multilingual | Language-specific rules | Language-specific dicts |
| Search recall | High | High |
| Search precision | Lower (over-conflates) | Higher |
| Best for | High-throughput indexing | Quality-critical systems |

**Concrete recall example**:

```
Query: "running shoes"

Without stemming:
  Index contains: "run", "runner", "runners", "shoe", "shoes"
  Query tokens: ["running", "shoes"]
  BM25 lookup: no match for "running" → MISS

With stemming (Porter):
  Index: "run_", "runner_" → stem → "run", "run"
         "shoe_", "shoes_" → stem → "shoe", "shoe"
  Query: "running" → "run", "shoes" → "shoe"
  BM25 lookup: MATCH → "running shoes" found!
```

### Stemming/Lemmatization for RAG/Embeddings

**You don't need either.** The embedding model handles this internally:

```python
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")
vecs = model.encode(["running shoes", "run shoe", "shoe for running"])

# Cosine similarities:
# running shoes ↔ run shoe:      0.94  ← model understands morphology
# running shoes ↔ shoe for running: 0.89

# The model already "knows" these are equivalent.
# Stemming would destroy the semantic signal the model uses.
```

Stemming/lemmatization is **only for lexical (BM25) search**. In hybrid search, apply it only to the BM25 branch.

---

## 4. Stop Words

Stop words are high-frequency tokens with low information content:

```
English stop words: the, a, an, is, are, was, were, be, been, being,
                    have, has, had, do, does, did, will, would, could,
                    should, may, might, shall, to, of, in, on, at, by,
                    for, with, about, against, between, into, through,
                    during, before, after, above, below, up, down...
```

### Why Remove Stop Words

**TF-IDF/BM25 scoring example**:

```
Document: "The quick brown fox jumps over the lazy dog"

TF("the") = 2/9 = 0.22   (high TF)
IDF("the") ≈ 0.0          (appears in every document → log(N/df) ≈ 0)
TF-IDF("the") ≈ 0.0       (correctly downweighted by IDF)

TF("fox") = 1/9 = 0.11   (lower TF)
IDF("fox") ≈ 3.7          (appears in few documents)
TF-IDF("fox") ≈ 0.41      (high relevance signal)
```

IDF **naturally suppresses** stop words in BM25. This means in modern search engines, explicit stop word removal is less critical than it was in TF-IDF era systems.

### When NOT to Remove Stop Words

```python
# "to be or not to be" — Hamlet query
# Without stop words: [] (empty! all words are stop words)
# Query fails completely.

# "The Who" — band name
# Without stop words: [] (both "the" and "who" are stop words)

# Phrase query: "man in the moon"
# After stop word removal: "man moon"
# phrase distance preserved? NO — "man" and "moon" are now adjacent in index
# but they weren't in the original. Phrase query breaks.

# Negation: "not guilty"
# Remove "not" → "guilty" → completely inverts meaning
```

### Custom Domain Stop Words

```python
# Medical corpus: "patient" is NOT a stop word
# (it's the most important entity)

# Legal corpus: "shall", "herein", "thereof" might be stop words
# (they appear in every document but carry little discriminating info)

# Code search: "def", "class", "return" might be over-weighted
# (appear in every Python file)

custom_stop_words = {
    "medical": ["the", "a", "an", "of", "in", "and", "or",
                "with", "for", "is", "was", "were", "been"],
    # Note: "patient" is NOT in this list
}
```

### OpenSearch/Lucene Configuration

```yaml
settings:
  analysis:
    filter:
      english_stop:
        type: stop
        stopwords: _english_    # built-in list
        # or:
        stopwords: ["the", "a", "an", "is"]  # custom list
        # or:
        stopwords_path: "analysis/stopwords_en.txt"  # file
    analyzer:
      my_analyzer:
        tokenizer: standard
        filter: [lowercase, english_stop, porter_stem]
```

---

## 5. Normalization

Normalization ensures that semantically equivalent text maps to the same representation.

### Case Folding

```python
# Simple ASCII lowercasing
"Apple Inc." → "apple inc."
"HTTP" → "http"

# Problem: loses meaning sometimes
"Apple" (company) → "apple" (fruit) — same in index
"US" (United States) → "us" (pronoun)
"IT" (Information Technology) → "it" (pronoun)

# Unicode case folding (more correct):
import unicodedata
# Turkish: "I" lowercases to "ı" (dotless i), not "i"
# Standard Python lower() is locale-aware on some platforms
"İstanbul".lower()  # "i̇stanbul" (Python's Unicode-aware lower)
```

### Unicode Normalization

The same visual character can have multiple byte representations:

```
"café" — 4 characters, but:
  NFC:  c-a-f-é     where é = U+00E9 (precomposed: 1 codepoint)
  NFD:  c-a-f-e-◌́   where é = e (U+0065) + combining acute (U+0301) (2 codepoints)

They look identical but:
  NFC bytes: 63 61 66 C3 A9        (5 bytes)
  NFD bytes: 63 61 66 65 CC 81     (6 bytes)

String comparison fails unless you normalize first!
```

| Form | Description | Use Case |
|------|-------------|----------|
| NFC | Composed (precomposed characters) | General text, storage |
| NFD | Decomposed (base + combining marks) | String analysis |
| NFKC | Compatibility composed | **Search** — maps ﬁ→fi, ²→2, ½→1/2 |
| NFKD | Compatibility decomposed | Aggressive normalization |

```python
import unicodedata

text = "ﬁle²"  # ﬁ is a ligature, ² is superscript
unicodedata.normalize("NFKC", text)  # "file2"  ← search-friendly

# For search: use NFKC
# It handles: ligatures, superscripts, subscripts, compatibility symbols
```

### Accent Folding

```python
def strip_accents(text):
    import unicodedata
    # NFD decomposes: é → e + ́
    # Then remove combining characters (category 'Mn')
    return "".join(
        c for c in unicodedata.normalize("NFD", text)
        if unicodedata.category(c) != "Mn"
    )

strip_accents("résumé")   # "resume"
strip_accents("naïve")    # "naive"
strip_accents("Björk")    # "Bjork"
```

**Trade-off**: Improves recall (user searching "resume" finds "résumé") but reduces precision (might also find unrelated "resuming").

**OpenSearch ASCIIFoldingFilter** handles this automatically:
```yaml
filter:
  ascii_fold:
    type: asciifolding
    preserve_original: true  # index both "résumé" and "resume"
```

### Punctuation Handling

```
Remove punctuation naively:
  "C++"     → "C"        ← WRONG: the language name is gone
  "e-mail"  → "email"    ← OK
  "U.S.A."  → "USA"      ← OK
  "3.14"    → "314"      ← WRONG: number changed
  "Mr."     → "Mr"       ← OK
  "don't"   → "dont"     ← debatable
```

Lucene's StandardTokenizer is smarter — it handles many of these edge cases by rule.

### Synonym Expansion

Two approaches with different trade-offs:

**Index-time expansion** (expand at write time):
```
Index: "automobile" → stores tokens: ["automobile", "car", "vehicle"]
Query: "car" → matches "automobile" documents
Pro: Query is simple, one lookup
Con: Index 3x larger, cannot change synonyms without reindexing
```

**Query-time expansion** (expand at read time):
```
Query: "car" → expand → ["car", "automobile", "vehicle"] → OR query
Pro: Update synonyms without reindexing
Con: Slower query, complex query tree
```

**Lucene SynonymGraphFilter** (OpenSearch):
```yaml
filter:
  synonym_filter:
    type: synonym_graph          # use synonym_graph, not synonym (deprecated)
    synonyms:
      - "car, automobile, vehicle"
      - "US, USA, United States, America"
      - "ML => machine learning"  # one-directional
    tokenizer: keyword            # for parsing synonyms file
```

### HTML Stripping

```python
from bs4 import BeautifulSoup

html = "<p>This is <b>bold</b> and <a href='...'>linked</a> text.</p>"
BeautifulSoup(html, "html.parser").get_text()
# "This is bold and linked text."

# OpenSearch: HTMLStripCharFilter (runs before tokenization)
# Strips tags, decodes HTML entities: &amp; → &, &lt; → <
```

### Language Detection

Different languages need different analyzer pipelines:

```python
from langdetect import detect
lang = detect("Das ist ein Satz auf Deutsch")  # "de"
lang = detect("This is an English sentence")   # "en"

# Route to appropriate analyzer:
analyzers = {
    "en": "english_analyzer",    # Porter stemmer
    "de": "german_analyzer",     # German stemmer (Snowball)
    "fr": "french_analyzer",     # French light stemmer
    "zh": "ik_max_word",         # IK Analyzer for Chinese (no spaces!)
    "ja": "kuromoji",            # Kuromoji for Japanese
}
```

---

## 6. Chunking Strategies (for RAG)

Chunking is the most impactful pre-processing decision for RAG quality. It determines what the retrieval system retrieves and what the LLM sees.

### Why Chunking Matters

```
Problem Space:
                    ┌──────────────────────────┐
                    │    Full Document          │
  Too large: ───▶   │  (100k tokens)           │   ← dilutes relevance signal
                    │                          │      wastes context window
                    └──────────────────────────┘

                    ┌────┐ ┌────┐ ┌────┐ ┌────┐
  Too small: ───▶   │ 50t│ │ 50t│ │ 50t│ │ 50t│   ← orphaned sentences
                    └────┘ └────┘ └────┘ └────┘      no surrounding context

                    ┌──────────────┐
  Sweet spot: ──▶   │  256-512     │             ← enough context
                    │  tokens      │                retrieval precision
                    └──────────────┘
```

**The fundamental tension**: Smaller chunks → better retrieval precision. Larger chunks → better context for the LLM. Parent-child chunking resolves this.

---

### Fixed-Size Chunking

Split every N characters or N tokens, with M overlap.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,          # tokens or chars depending on length_function
    chunk_overlap=50,        # overlap prevents info loss at boundaries
    length_function=len,     # or: use tiktoken for token counting
)

chunks = splitter.split_text(document_text)

# With token-accurate counting:
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    length_function=lambda text: len(enc.encode(text)),
)
```

**Overlap example**:
```
Chunk 1: "...PostgreSQL uses MVCC for concurrency control. Each"
                                               ┌────────────┐
Chunk 2:                  "Each transaction sees a consistent snapshot of"
         ├─── overlap ───┤
```

**When to use**: Baseline, uniform documents, speed-critical indexing pipelines, when document structure is unknown or inconsistent.

---

### Sentence-Based Chunking

```python
import spacy
nlp = spacy.load("en_core_web_sm")

def sentence_chunk(text, sentences_per_chunk=3):
    doc = nlp(text)
    sentences = [sent.text for sent in doc.sents]
    chunks = []
    for i in range(0, len(sentences), sentences_per_chunk):
        chunk = " ".join(sentences[i:i+sentences_per_chunk])
        chunks.append(chunk)
    return chunks

# NLTK alternative (faster, no full NLP pipeline):
from nltk.tokenize import sent_tokenize
sentences = sent_tokenize(text)  # Punkt tokenizer handles "Dr. Smith" correctly
```

**Punkt tokenizer handles abbreviations**:
```
Text: "Dr. Smith works at St. Mary's Hospital. He treats patients."
Punkt: ["Dr. Smith works at St. Mary's Hospital.", "He treats patients."]
Naive split on ".": ["Dr", " Smith works at St", " Mary's Hospital", ...]  WRONG
```

**Problem**: Sentence length varies enormously (10 words to 100 words). Fixed sentence count → variable token count.

---

### Recursive Character Splitting

LangChain's `RecursiveCharacterTextSplitter` is the most practical choice for prose:

```python
# Tries separators in order until chunks are small enough:
separators = [
    "\n\n",    # Paragraph breaks (try first — preserves most structure)
    "\n",      # Line breaks
    ". ",      # Sentence boundaries
    " ",       # Word boundaries
    "",        # Character (last resort)
]

# Algorithm:
# 1. Try splitting on "\n\n"
# 2. If resulting chunks are within chunk_size → done
# 3. If chunks are too large, recursively apply next separator
```

```
Input (400 chars):
"PostgreSQL uses MVCC for concurrency.

MVCC stands for Multi-Version Concurrency Control.
It allows multiple transactions to read data..."

Split on \n\n first:
  Chunk A: "PostgreSQL uses MVCC for concurrency." (37 chars) → under limit
  Chunk B: "MVCC stands for Multi-Version Concurrency Control.\nIt allows multiple transactions..." (200 chars) → under limit
```

---

### Semantic Chunking

Split based on topic boundaries detected via embedding similarity.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai.embeddings import OpenAIEmbeddings

chunker = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",   # or "standard_deviation", "interquartile"
    breakpoint_threshold_amount=95,           # split at 95th percentile similarity drops
)

# Algorithm (Greg Kamradt's approach):
# 1. Split text into sentences
# 2. Create sliding windows of sentences (e.g., 3-sentence windows)
# 3. Embed each window
# 4. Compute cosine similarity between adjacent windows
# 5. Where similarity drops sharply → topic boundary → split there
```

```
Similarity curve:
  0.95 ──────────────╮    ╭──────────────────╮
  0.85               │    │                  │
  0.75               ╰────╯                  │
  0.65                                       ╰──────────
        [para 1]   [topic  [para 3]  [para 4] [topic
                   shift]                     shift]
                     ↑ split here               ↑ split here
```

**Cost**: O(N) embedding calls where N = number of sentences. At $0.0001/1K tokens, 1000 sentences ≈ $0.05.

**Problem**: Non-deterministic (depends on embedding model). Harder to reproduce. Topic detection is imperfect.

---

### Document-Structure-Aware Chunking

```python
# Markdown: split on headers
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "h1"),
    ("##", "h2"),
    ("###", "h3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
chunks = splitter.split_text(markdown_text)

# Each chunk carries header metadata:
# chunk.metadata = {"h1": "Installation", "h2": "Requirements"}
# Use for: prefix chunk content with breadcrumb trail
```

```python
# HTML: split on structural elements
from bs4 import BeautifulSoup

def html_chunk(html):
    soup = BeautifulSoup(html, "html.parser")
    chunks = []
    for section in soup.find_all(["section", "article", "div"], class_="content"):
        text = section.get_text(separator=" ", strip=True)
        if text:
            chunks.append({"text": text, "id": section.get("id", "")})
    return chunks
```

```python
# PDF: structure extraction is hard
import pymupdf  # PyMuPDF (fitz)

doc = pymupdf.open("document.pdf")
text_with_structure = []
for page_num, page in enumerate(doc):
    blocks = page.get_text("dict")["blocks"]
    for block in blocks:
        if block["type"] == 0:  # text block
            text_with_structure.append({
                "text": " ".join(span["text"] for line in block["lines"]
                                 for span in line["spans"]),
                "page": page_num + 1,
                "font_size": block["lines"][0]["spans"][0]["size"] if block["lines"] else 0,
                "bbox": block["bbox"],
            })

# Use font_size to infer headers (larger = heading)
# Use bbox to detect columns, captions, footnotes
```

**PDF gotcha**: PDF is a visual format, not a semantic one. Text order in the file ≠ reading order. Multi-column layouts, footnotes, headers, and tables all require special handling.

---

### Parent-Child Chunking (Multi-Granularity)

The best of both worlds: small chunks for retrieval precision, large chunks for LLM context.

```
Document (full text)
  │
  ├── Parent Chunk A (1024 tokens)  ← stored in doc store
  │     ├── Child Chunk A1 (128 tokens)  ← stored in vector DB
  │     ├── Child Chunk A2 (128 tokens)
  │     └── Child Chunk A3 (128 tokens)
  │
  ├── Parent Chunk B (1024 tokens)
  │     ├── Child Chunk B1 (128 tokens)
  │     └── Child Chunk B2 (128 tokens)
  │
  └── ...

Retrieval flow:
  Query → embed → find similar child chunks (e.g., A2, B1)
        → look up parent_id → fetch Parent A, Parent B
        → return Parent A and Parent B to LLM
```

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_chroma import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Child splitter: small, precise
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)

# Parent splitter: large, contextual
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

vectorstore = Chroma(embedding_function=embeddings)
docstore = InMemoryStore()  # use Redis/DynamoDB in production

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(documents)

# Query returns parent chunks, not child chunks:
results = retriever.invoke("how does MVCC work?")
```

---

### AST-Based Code Chunking

Naive text splitting destroys code semantics. Use the AST.

```python
import tree_sitter_python as tspython
from tree_sitter import Language, Parser

PY_LANGUAGE = Language(tspython.language())
parser = Parser(PY_LANGUAGE)

def extract_functions(source_code: str) -> list[dict]:
    tree = parser.parse(source_code.encode())
    chunks = []

    def visit(node, class_name=None):
        if node.type == "class_definition":
            # Extract class name
            name_node = node.child_by_field_name("name")
            current_class = name_node.text.decode() if name_node else None
            for child in node.children:
                visit(child, class_name=current_class)

        elif node.type in ("function_definition", "async_function_definition"):
            name_node = node.child_by_field_name("name")
            func_name = name_node.text.decode() if name_node else "unknown"
            text = source_code[node.start_byte:node.end_byte]

            # Prefix with class context for methods
            full_name = f"{class_name}.{func_name}" if class_name else func_name
            chunks.append({
                "text": text,
                "name": full_name,
                "start_line": node.start_point[0],
                "end_line": node.end_point[0],
                "type": "method" if class_name else "function",
            })
        else:
            for child in node.children:
                visit(child, class_name=class_name)

    visit(tree.root_node)
    return chunks

# Tree-sitter supports: Python, JavaScript, TypeScript, Go, Rust, Java,
# C, C++, Ruby, PHP, C#, Kotlin, Swift, Scala, and 30+ more
```

**Include class name as context prefix**:
```python
# Without context:
chunk = "def connect(self): ..."  # connect to WHAT?

# With context:
chunk = "# Class: DatabasePool\ndef connect(self): ..."  # clear!
```

---

### Contextual Chunk Enrichment (Anthropic, 2024 — still a recommended technique in 2026)

**The problem**: A chunk extracted from a document loses its document-level context.

```
Document: "PostgreSQL Internals: WAL and Checkpoints"
...section 3...
Chunk: "The frequency is controlled by the checkpoint_completion_target
        parameter, which defaults to 0.9..."

Without context: What is the frequency of? What checkpoint? What parameter?
```

**The solution**: Prefix each chunk with a generated context sentence.

```python
import anthropic

client = anthropic.Anthropic()

def enrich_chunk(document: str, chunk: str) -> str:
    # Use prompt caching for the (large) document — only charge once
    response = client.messages.create(
        model="claude-haiku-4-5",  # fast + cheap for this task; claude-3-5-haiku-20241022 was retired Feb 2026 — always check platform.claude.com for the current model ID
        max_tokens=100,
        system="Generate a concise context sentence for a chunk.",
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": f"<document>{document}</document>",
                    "cache_control": {"type": "ephemeral"},  # cache the doc
                },
                {
                    "type": "text",
                    "text": f"Chunk to contextualize:\n<chunk>{chunk}</chunk>\n\n"
                             "Provide a single sentence that situates this chunk "
                             "within the document context. Be specific.",
                }
            ]
        }]
    )
    context = response.content[0].text
    return f"{context}\n\n{chunk}"

# Example output:
# "This chunk is from a document about PostgreSQL WAL internals,
#  specifically discussing checkpoint frequency configuration."
#
# The frequency is controlled by the checkpoint_completion_target
# parameter, which defaults to 0.9...
```

**Results** (from Anthropic's 2024 paper):
- BM25 retrieval: 49% improvement
- Embedding retrieval: 35% improvement
- Combined hybrid: 67% improvement on some benchmarks

**2026 status**: Contextual retrieval is now a well-established, widely-adopted technique (integrated into vector DB docs like Milvus, referenced across RAG tooling) — treat it as current best practice, not a novel research idea. That said, newer research argues the LLM-per-chunk-contextualization approach is expensive at indexing time relative to alternatives (e.g., encoder-based contextualization like ModernBERT + InSeNT), so for very large or fast-changing corpora, evaluate cheaper contextualization methods before defaulting to an LLM call per chunk. The core insight — chunks lose meaning without document context, so inject it — remains valid regardless of which method generates the context.

**Cost optimization**: With prompt caching, the document is charged once per context window TTL (5 minutes, or longer with extended/1-hour caching now available on some models). For a 100-chunk document, you pay for the document once + 100 small chunk queries. Also note: with 1M-token context windows now standard on frontier models, the "large cached document" can itself be much bigger than in 2024 without hitting context limits.

---

### Chunk Metadata

Always attach metadata. It's nearly free and highly valuable.

```python
chunk = {
    "text": "...",
    "metadata": {
        "source_url": "https://docs.example.com/pg/wal",
        "document_title": "PostgreSQL WAL Internals",
        "section_header": "3.2 Checkpoint Frequency",
        "chunk_index": 7,           # position within document
        "total_chunks": 23,         # total chunks in document
        "timestamp": "2024-01-15T10:30:00Z",
        "file_type": "markdown",
        "language": "en",
        "token_count": 342,
        "parent_chunk_id": "doc_abc_parent_3",  # for parent-child retrieval
    }
}
```

**Uses**:
- **Filtering**: `WHERE timestamp > 2024-01-01` (freshness)
- **Citation**: `"Source: PostgreSQL WAL Internals, §3.2"`
- **Staleness detection**: re-index old chunks
- **Debugging**: which chunk returned this answer?

---

### Chunking Strategy Selection Guide

| Strategy | Best For | Chunk Size | Pros | Cons |
|----------|----------|------------|------|------|
| Fixed-size | Baseline, uniform docs | 256-512 tokens | Fast, simple | Mid-sentence splits |
| Sentence | Prose, articles | 3-5 sentences | Natural boundaries | Variable size |
| Recursive | Most prose | 512 tokens | Structure-aware | Still paragraph-based |
| Semantic | Mixed-topic docs | Varies | Topic-coherent | Expensive, slow |
| Markdown header | Docs, wikis | Section size | Preserves structure | Requires markdown |
| AST-based | Code | Function size | Semantic units | Language-specific |
| Parent-child | Mixed needs | Parent: 2048, Child: 256 | Best of both | Complex infra |
| Contextual | High-quality RAG | 256-512 tokens | Best retrieval | LLM cost at index time |

---

## 7. The Full Text Analysis Pipeline (Lucene/OpenSearch)

```
Input Text
    │
    ▼
┌─────────────────┐
│   CharFilters   │  ← modify the character stream BEFORE tokenization
│                 │    HTMLStripCharFilter: <b>bold</b> → "bold"
│                 │    MappingCharFilter:  "æ" → "ae", "&" → "and"
└────────┬────────┘
         │
    Character Stream
         │
         ▼
┌─────────────────┐
│   Tokenizer     │  ← ONE tokenizer (splits stream into tokens)
│                 │    StandardTokenizer: Unicode word boundaries
│                 │    WhitespaceTokenizer: split on whitespace only
│                 │    NGramTokenizer: sliding window n-grams
│                 │    EdgeNGramTokenizer: prefix n-grams
│                 │    PathHierarchyTokenizer: "/a/b/c" → ["/a", "/a/b", "/a/b/c"]
└────────┬────────┘
         │
    Token Stream (with positions and offsets)
         │
         ▼
┌─────────────────┐
│  TokenFilters   │  ← ZERO or more filters, applied in order
│                 │    LowercaseFilter
│                 │    StopFilter
│                 │    PorterStemFilter / SnowballFilter
│                 │    SynonymGraphFilter
│                 │    ASCIIFoldingFilter
│                 │    NGramTokenFilter
│                 │    EdgeNGramTokenFilter
└────────┬────────┘
         │
    Final Token Stream → Inverted Index
```

### Custom Analyzer: Multi-Language Product Search

```yaml
# opensearch/elasticsearch index settings
PUT /products
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip": {
          "type": "html_strip"
        },
        "special_chars": {
          "type": "mapping",
          "mappings": [
            "& => and",
            "+ => plus",
            "@ => at"
          ]
        }
      },
      "tokenizer": {
        "standard_tokenizer": {
          "type": "standard",
          "max_token_length": 255
        }
      },
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "product_synonyms": {
          "type": "synonym_graph",
          "synonyms": [
            "laptop, notebook, computer",
            "tv, television, monitor",
            "phone => smartphone, mobile, cellphone"
          ]
        },
        "ascii_fold": {
          "type": "asciifolding",
          "preserve_original": true
        },
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      },
      "analyzer": {
        "product_search_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip", "special_chars"],
          "tokenizer": "standard_tokenizer",
          "filter": [
            "lowercase",
            "ascii_fold",
            "english_stop",
            "product_synonyms",
            "english_stemmer"
          ]
        },
        "product_autocomplete_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard_tokenizer",
          "filter": [
            "lowercase",
            "edge_ngram_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "product_search_analyzer",
        "search_analyzer": "product_search_analyzer",
        "fields": {
          "autocomplete": {
            "type": "text",
            "analyzer": "product_autocomplete_analyzer",
            "search_analyzer": "standard"  # query without edge ngram
          }
        }
      }
    }
  }
}
```

### Debugging with `_analyze` API

```bash
# See exactly what tokens your analyzer produces:
GET /products/_analyze
{
  "analyzer": "product_search_analyzer",
  "text": "Running Shoes & Sneakers"
}

# Response:
{
  "tokens": [
    {"token": "running", "start_offset": 0,  "end_offset": 7,  "position": 0},
    {"token": "run",     "start_offset": 0,  "end_offset": 7,  "position": 0},  # synonym
    {"token": "shoe",    "start_offset": 8,  "end_offset": 13, "position": 1},  # stemmed
    {"token": "sneaker", "start_offset": 17, "end_offset": 25, "position": 2}   # stemmed
  ]
}

# Test a specific tokenizer or filter:
GET /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "porter_stem"],
  "text": "Running Shoes"
}
```

---

## 8. N-Grams for Search

### Character N-Grams

```
"hello" with trigrams (n=3):
  ["hel", "ell", "llo"]

"hello" with bigrams (n=2):
  ["he", "el", "ll", "lo"]
```

**Use cases**:
- **Fuzzy matching**: "helllo" still shares "hel", "ell" with "hello"
- **Prefix-agnostic matching**: "ello" (typo missing 'h') still shares "ell", "llo"
- **Morphologically rich languages**: German, Finnish — word endings vary enormously

```
German: "Kraftfahrzeug" (car), "Kraftfahrzeugbrief" (car title), "Kraftfahrzeughaftpflichtversicherung" (car insurance)
Stemming fails → character n-grams work because the root "Kraftfahrzeug" is shared
```

### Word N-Grams

```
"machine learning model" with bigrams:
  ["machine learning", "learning model"]

Combined unigram + bigram index:
  ["machine", "learning", "model", "machine learning", "learning model"]
```

**Use cases**:
- Phrase matching without positional index overhead
- Collocations: "New York", "machine learning" should score higher together

### EdgeNGram (Autocomplete)

```
"hello" → min_gram=2, max_gram=5:
  ["he", "hel", "hell", "hello"]

Autocomplete query "hel" → matches "hello"
```

**Critical settings**:
```yaml
# WRONG: apply edge n-gram at both index AND query time
# → "hel" at query time → "he", "hel" → matches ANY prefix of any word

# CORRECT:
"title": {
  "type": "text",
  "analyzer": "autocomplete_index_analyzer",    # edge n-gram at index
  "search_analyzer": "standard"                 # NO edge n-gram at query
}
```

### N-Gram Size Trade-offs

```
min_gram=1: index explosion, massive index
  "hello" → ["h", "he", "hel", "hell", "hello"] (5 entries)
  100K documents × avg 100 tokens × avg 5 chars → 50M index entries for 1-grams

min_gram=2: practical minimum
min_gram=3: better precision, still handles most typos (1-char typo)

Index size grows O(n²) with document length for character n-grams.
```

---

## 9. Preprocessing: BM25 vs Embeddings vs LLM Input

| Preprocessing Step | BM25 / Lexical | Embeddings | LLM Prompt |
|-------------------|----------------|------------|------------|
| **Tokenization** | Lucene tokenizer (word-level) | Model internal (subword BPE/WP) | Model internal |
| **Lowercasing** | Critical | Mild benefit | No |
| **Stop words** | Optional (IDF handles it) | Do not remove | No |
| **Stemming** | High value | Do not use | No |
| **Lemmatization** | High value (slower) | Do not use | No |
| **Punctuation removal** | Usually yes | Preserve sentence structure | Preserve |
| **Accent folding** | High value | Low value | No |
| **Unicode normalization** | Critical (NFC/NFKC) | NFKC recommended | NFKC |
| **Synonym expansion** | Critical | Low value (model handles) | No |
| **HTML stripping** | Critical | Critical | Critical |
| **Chunking** | Affects recall | Critical (size = embedding quality) | Critical (token budget) |
| **Markdown/structure** | Strip | Mild benefit | **Preserve** (helps LLM parse) |
| **Chunk overlap** | Not applicable | 10-15% recommended | Managed in prompt |

**The #1 rule**: For embeddings, less is more. The model already knows that "running" and "run" are related. Your job is to:
1. Feed clean text (strip HTML, fix encoding)
2. Chunk at the right size for the model's max input
3. Not destroy information (no stemming, no stop word removal)

---

## 10. Practical Recipes

### Recipe 1: OpenSearch Custom English Product Search Analyzer

(See full YAML in Section 7 above)

Key additions for product search:
```yaml
# Brand name handling: don't lowercase brand names in a separate field
"brand": {
  "type": "keyword"  # exact match, case-sensitive
}

# SKU/product codes: keyword field, plus edge ngram for prefix
"sku": {
  "type": "keyword",
  "fields": {
    "prefix": {
      "type": "text",
      "analyzer": "product_autocomplete_analyzer"
    }
  }
}
```

---

### Recipe 2: PDF Ingestion Pipeline

```python
import pymupdf  # pip install pymupdf
from langchain.text_splitter import RecursiveCharacterTextSplitter
import tiktoken

def ingest_pdf(pdf_path: str, chunk_size: int = 512, chunk_overlap: int = 50):
    enc = tiktoken.encoding_for_model("text-embedding-3-small")

    # Step 1: Extract text with page information
    doc = pymupdf.open(pdf_path)
    pages = []
    for page_num, page in enumerate(doc):
        text = page.get_text("text")
        pages.append({
            "text": text,
            "page": page_num + 1,
            "page_count": len(doc),
        })
    doc.close()

    # Step 2: Clean extracted text
    def clean_pdf_text(text):
        import re
        # Fix common PDF extraction artifacts
        text = re.sub(r'\s*\n\s*\n\s*', '\n\n', text)  # normalize blank lines
        text = re.sub(r'(?<![.!?])\n(?=[a-z])', ' ', text)  # rejoin wrapped lines
        text = re.sub(r'-\n', '', text)                  # fix hyphenated line breaks
        text = re.sub(r'\x0c', '\n\n', text)             # form feed → paragraph break
        return text.strip()

    # Step 3: Combine pages with page markers
    full_text = ""
    page_offsets = []
    for page in pages:
        offset = len(full_text)
        page_offsets.append(offset)
        full_text += clean_pdf_text(page["text"]) + "\n\n"

    # Step 4: Chunk
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=lambda t: len(enc.encode(t)),
        separators=["\n\n", "\n", ". ", " ", ""],
    )

    chunks_raw = splitter.create_documents(
        [full_text],
        metadatas=[{"source": pdf_path}]
    )

    # Step 5: Attach page numbers to chunks
    def find_page(offset):
        for i, page_offset in enumerate(reversed(page_offsets)):
            if offset >= page_offset:
                return len(page_offsets) - i
        return 1

    chunks = []
    for i, chunk in enumerate(chunks_raw):
        # Approximate page from character offset
        chunks.append({
            "text": chunk.page_content,
            "metadata": {
                **chunk.metadata,
                "chunk_index": i,
                "total_chunks": len(chunks_raw),
                "token_count": len(enc.encode(chunk.page_content)),
            }
        })

    return chunks
```

---

### Recipe 3: AST-Based Code Chunking with Tree-sitter

```python
# pip install tree-sitter tree-sitter-python tree-sitter-javascript
import tree_sitter_python as tspython
import tree_sitter_javascript as tsjavascript
from tree_sitter import Language, Parser

LANGUAGES = {
    "python": Language(tspython.language()),
    "javascript": Language(tsjavascript.language()),
}

def code_chunk(source_code: str, language: str = "python") -> list[dict]:
    lang = LANGUAGES.get(language)
    if not lang:
        raise ValueError(f"Unsupported language: {language}")

    parser = Parser(lang)
    tree = parser.parse(source_code.encode("utf-8"))

    chunks = []

    def extract_chunks(node, class_context: str = None, depth: int = 0):
        # Function/method definitions
        if node.type in ("function_definition", "async_function_definition",
                          "function_declaration", "method_definition"):
            name_node = node.child_by_field_name("name")
            func_name = name_node.text.decode() if name_node else "anonymous"
            full_name = f"{class_context}.{func_name}" if class_context else func_name

            text = source_code[node.start_byte:node.end_byte]

            # Extract docstring if present (Python)
            docstring = ""
            body = node.child_by_field_name("body")
            if body and body.child_count > 0:
                first_stmt = body.children[0]
                if first_stmt.type == "expression_statement":
                    expr = first_stmt.children[0] if first_stmt.children else None
                    if expr and expr.type == "string":
                        docstring = expr.text.decode()

            chunks.append({
                "text": text,
                "name": full_name,
                "docstring": docstring,
                "start_line": node.start_point[0] + 1,
                "end_line": node.end_point[0] + 1,
                "language": language,
                "type": "method" if class_context else "function",
                # Prefix for better embedding context:
                "embed_text": f"# {language} {('method' if class_context else 'function')}: {full_name}\n{text}",
            })

        # Class definitions — recurse with class context
        elif node.type in ("class_definition", "class_declaration"):
            name_node = node.child_by_field_name("name")
            class_name = name_node.text.decode() if name_node else "Anonymous"
            for child in node.children:
                extract_chunks(child, class_context=class_name, depth=depth+1)
            return  # don't recurse again below

        # Module-level code (not inside a function/class)
        elif node.type == "module" or (depth == 0 and node.type not in
                                        ("function_definition", "class_definition",
                                         "async_function_definition")):
            for child in node.children:
                extract_chunks(child, class_context=class_context, depth=depth)
            return

        # Recurse for other nodes
        if node.type not in ("function_definition", "async_function_definition",
                               "function_declaration", "method_definition"):
            for child in node.children:
                extract_chunks(child, class_context=class_context, depth=depth)

    extract_chunks(tree.root_node)
    return chunks

# Usage:
with open("my_module.py") as f:
    source = f.read()

chunks = code_chunk(source, language="python")
for chunk in chunks:
    print(f"{chunk['name']} ({chunk['start_line']}-{chunk['end_line']})")
    print(chunk["embed_text"][:200])
    print("---")
```

---

### Recipe 4: Contextual Enrichment Pipeline (Hybrid BM25 + Embedding)

```python
import anthropic
from opensearchpy import OpenSearch
import numpy as np
from sentence_transformers import SentenceTransformer

anthropic_client = anthropic.Anthropic()
embedding_model = SentenceTransformer("all-MiniLM-L6-v2")

def enrich_chunks_with_context(document: str, chunks: list[dict]) -> list[dict]:
    """Add LLM-generated context prefix to each chunk."""
    enriched = []
    for chunk in chunks:
        response = anthropic_client.messages.create(
            model="claude-haiku-4-5",  # claude-3-5-haiku-20241022 retired Feb 2026
            max_tokens=120,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"<document>\n{document}\n</document>",
                        "cache_control": {"type": "ephemeral"},  # cache large doc
                    },
                    {
                        "type": "text",
                        "text": (
                            f"<chunk>\n{chunk['text']}\n</chunk>\n\n"
                            "In one sentence, situate this chunk within the document. "
                            "Be specific about what section/topic this is from. "
                            "Do not include phrases like 'This chunk is about'. "
                            "Just write the context."
                        ),
                    }
                ],
            }]
        )
        context = response.content[0].text.strip()
        enriched_text = f"{context}\n\n{chunk['text']}"
        enriched.append({
            **chunk,
            "enriched_text": enriched_text,
            "context_prefix": context,
        })
    return enriched

def index_chunks(chunks: list[dict], index_name: str = "knowledge_base"):
    """Index chunks for hybrid BM25 + vector retrieval."""
    os_client = OpenSearch([{"host": "localhost", "port": 9200}])

    for chunk in chunks:
        # Generate embedding from ENRICHED text
        embedding = embedding_model.encode(chunk["enriched_text"]).tolist()

        doc = {
            "text": chunk["text"],              # original (for display)
            "enriched_text": chunk["enriched_text"],  # for BM25
            "context_prefix": chunk.get("context_prefix", ""),
            "embedding": embedding,              # for kNN
            **chunk.get("metadata", {}),
        }
        os_client.index(index=index_name, body=doc)

def hybrid_search(query: str, index_name: str = "knowledge_base",
                  bm25_weight: float = 0.4, vector_weight: float = 0.6,
                  top_k: int = 5) -> list[dict]:
    """Reciprocal Rank Fusion of BM25 and vector results."""
    os_client = OpenSearch([{"host": "localhost", "port": 9200}])
    query_embedding = embedding_model.encode(query).tolist()

    # BM25 search on enriched_text
    bm25_response = os_client.search(index=index_name, body={
        "query": {"match": {"enriched_text": query}},
        "size": top_k * 2,
    })

    # kNN vector search
    knn_response = os_client.search(index=index_name, body={
        "query": {
            "knn": {
                "embedding": {
                    "vector": query_embedding,
                    "k": top_k * 2,
                }
            }
        },
        "size": top_k * 2,
    })

    # Reciprocal Rank Fusion (RRF)
    rrf_scores = {}
    k = 60  # RRF constant

    for rank, hit in enumerate(bm25_response["hits"]["hits"]):
        doc_id = hit["_id"]
        rrf_scores[doc_id] = rrf_scores.get(doc_id, 0) + bm25_weight / (k + rank + 1)

    for rank, hit in enumerate(knn_response["hits"]["hits"]):
        doc_id = hit["_id"]
        rrf_scores[doc_id] = rrf_scores.get(doc_id, 0) + vector_weight / (k + rank + 1)

    # Merge and sort
    all_hits = {hit["_id"]: hit for hit in
                bm25_response["hits"]["hits"] + knn_response["hits"]["hits"]}

    ranked = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
    return [all_hits[doc_id]["_source"] for doc_id, _ in ranked[:top_k]]
```

---

## Key Gotchas

**1. Chunk size ≠ token count by default**
```python
# CharacterTextSplitter uses len() (character count), NOT token count
# 512 chars ≠ 512 tokens (can be 100-400 tokens depending on text)
# Always use a token-counting length_function for LLM-targeted chunking:
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")
length_function=lambda t: len(enc.encode(t))
```

**2. Stemming breaks phrase queries**
```
Index: "machine learning" → stem → ["machin", "learn"]
Phrase query: "machine learning" → ["machin", "learn"] at adjacent positions
This works! BUT:
"machine" → "machin" and "machinery" → "machin" → they match each other
In a phrase query context this might be acceptable, but watch for false positives.
```

**3. Semantic chunking drifts with embedding model updates**
```
Chunk boundaries are determined by cosine similarity thresholds.
If you update your embedding model, the SAME text produces DIFFERENT chunks.
This means your vector index and your chunking logic are now inconsistent.
Always reindex everything when changing embedding models.
```

**4. PDF text extraction quality varies wildly by PDF type**
```
PDF Type                     | PyMuPDF quality
-----------------------------|----------------
Text-layer PDF (pdflatex)    | Excellent
Scanned + OCR embedded       | Good (OCR quality varies)
Scanned without OCR          | Empty (need OCR: pytesseract)
Complex layout (2-column)    | Poor (text order jumbled)
Tables                       | Very poor (cells merged)
```

**5. BPE token counts ≠ word counts**
```python
# Common assumption: 1 word ≈ 1 token
# Reality: depends heavily on the text

import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")

texts = [
    "The quick brown fox",          # 5 tokens (common words)
    "antidisestablishmentarianism", # 6 tokens (rare long word splits)
    "2024-01-15T10:30:00Z",        # 9 tokens (structured data)
    "```python\ndef foo():\n```",   # 10 tokens (code)
]
for t in texts:
    print(f"'{t[:30]}': {len(enc.encode(t))} tokens")
```

**6. Stop word removal breaks Boolean must-have queries**
```
Query: "NOT gate" (logic gate type)
After stop word removal: "gate" (removed "NOT")
Result: every document with "gate" matches — wrong!

Similarly:
"to be or not to be" → "" after stop word removal → no results
```

**7. Case folding loses entity distinctions**
```
"Apple" (company) → "apple" (fruit)
"WHO" (World Health Organization) → "who" (pronoun)
"IT" (Information Technology) → "it" (pronoun)
"US" (United States) → "us" (pronoun)

Mitigation: Use multi-field mapping
  "title": analyzed field (lowercased)
  "title.raw": keyword field (exact case)
```

**8. Embedding models have max input lengths**
```
Model                          | Max tokens
-------------------------------|------------
text-embedding-ada-002         | 8,191
text-embedding-3-small         | 8,191
text-embedding-3-large         | 8,191
all-MiniLM-L6-v2               | 256 (!)
all-mpnet-base-v2              | 384
e5-large-v2                    | 512
BAAI/bge-large-en-v1.5        | 512

Text beyond max tokens is SILENTLY TRUNCATED, not errored.
A 1000-token chunk fed to all-MiniLM-L6-v2 → only first 256 tokens embedded.
ALWAYS verify chunk size ≤ model max input.
```

**9. Synonym expansion at index time causes phrase query failures**
```
Query: "New York" (phrase query, requires "New" adjacent to "York")
Synonyms: "New York" → ["New York", "NYC", "Big Apple"]

If synonyms are expanded at INDEX time:
  "New York" → tokens: ["new", "york", "nyc", "big", "apple"] at positions 0,1,2,3,4
  But "big" and "apple" are at positions 3 and 4, not adjacent to "new" and "york"
  Phrase query for "New York" might still work (position 0,1), but complex synonym
  graphs with multi-word synonyms can corrupt position information.

Use SynonymGraphFilter (not deprecated SynonymFilter) at SEARCH time for phrases.
```

**10. Contextual chunk enrichment can introduce hallucinations**
```python
# The LLM generating context prefixes can hallucinate
# If the document is long and the chunk is ambiguous:

chunk = "The parameter defaults to 0.9."
# LLM context: "This discusses PostgreSQL checkpoint_completion_target"
# But the chunk was actually about Nginx worker_processes!
# (in a long document covering multiple technologies)

# Mitigation:
# 1. Use smaller documents or include section headers in the prompt
# 2. Have the LLM quote from the document ("As stated in section X...")
# 3. Keep context prefix concise (1 sentence max)
# 4. Use a strong model (Haiku is fast but Sonnet is more accurate)
```

**11. Recursive character splitter respects separators but not token model context**
```python
# RecursiveCharacterTextSplitter respects "\n\n" paragraph breaks
# BUT: a code block spanning multiple paragraphs will be split mid-block:

"""
Here is an example:

```python
def foo():
    # This is a very long function
    # with many lines
    ...


    return result   # ← split happens HERE if too long
```

The function above shows...
"""
# The ``` fence is not a separator! Use MarkdownTextSplitter for markdown content.
```

**12. Language detection fails on short text**
```python
from langdetect import detect
detect("OK")           # might return "sv" (Swedish!) or random
detect("Hello")        # "en" usually correct but not guaranteed
detect("Bonjour")      # "fr" usually correct

# langdetect is non-deterministic — add DetectorFactory.seed = 0 for reproducibility
from langdetect import DetectorFactory
DetectorFactory.seed = 0

# For short product titles or queries: use langdetect with a fallback
# and minimum character threshold (e.g., skip detection for < 20 chars)
```

---

## Quick Reference Card

```
TOKENIZATION
  BM25/Search: StandardTokenizer (word-level, position-aware)
  LLM/Embed:   BPE/WordPiece/SentencePiece (subword, model-internal)

STEMMING vs LEMMATIZATION
  Stemming:  fast, crude, for high-throughput BM25 indexing
  Lemmatize: slow, accurate, for quality-critical lexical search
  Neither:   for embeddings (model handles morphology)

CHUNKING DECISION TREE
  Code? → AST-based (tree-sitter)
  Markdown/HTML? → Structure-aware (header split)
  Mixed-topic? → Semantic chunking
  High quality needed? → Parent-child + contextual enrichment
  Default/quick: → RecursiveCharacterTextSplitter(512, 50)

CHUNK SIZE TARGETS
  Embedding only:  256-512 tokens (model-specific max)
  RAG (LLM):       512-1024 tokens (balance context vs precision)
  Parent chunks:   1024-2048 tokens
  Child chunks:    128-256 tokens

PREPROCESSING BY SYSTEM
  BM25:    normalize → tokenize → lowercase → stop words → stem/lemmatize → synonyms
  Embed:   strip HTML → NFKC normalize → chunk (nothing else)
  LLM:     strip HTML → preserve structure → add metadata → manage token budget
```
