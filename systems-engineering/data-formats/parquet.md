# Apache Parquet

## 30-Second Intuition

Parquet is a columnar file format: instead of storing rows one after another (like CSV or Avro), it groups values column-by-column, so a query that only touches 3 of 50 columns only has to read those 3 columns off disk. Every table format built on top of it — Iceberg, Delta Lake, Hudi — is fundamentally "a transaction log pointing at a pile of Parquet files." The one fact that matters operationally: Parquet's own internal min/max statistics and page indexes are what make file-skipping and predicate pushdown possible upstream — Z-ordering, Liquid Clustering, and manifest-level pruning all exist to make Parquet's per-file stats *tight*, not to replace them.

---

## File Anatomy: Row Groups → Column Chunks → Pages

```
Parquet file
├── Row Group 0 (~128MB default target size)
│   ├── Column Chunk: user_id   (contiguous bytes for this column, this row group)
│   │   ├── Page 0 (dictionary page, optional)
│   │   ├── Page 1 (data page, ~1MB)
│   │   └── Page 2 (data page)
│   ├── Column Chunk: event_type
│   │   └── ...
│   └── Column Chunk: ts
│       └── ...
├── Row Group 1
│   └── ... (same column chunk layout)
├── Column Index / Offset Index  (per-page min/max + byte offsets, near footer)
└── Footer
    ├── File metadata (schema, row group locations, key-value metadata)
    └── Magic bytes "PAR1"
```

A **row group** is a horizontal slice of the table — all columns for some N rows, physically contiguous per column within the group. A **column chunk** is one column's data for one row group — guaranteed contiguous bytes, which is what lets a reader seek directly to `user_id`'s bytes in row group 3 without touching any other column. A **page** is the smallest unit of encoding/compression — typically ~1MB, chosen so decompression can happen without materializing an entire column chunk in memory at once.

This is the actual reason row-oriented formats can't do column pruning cheaply: in Avro/CSV, column values for the same row are adjacent, so reading "just column X" still means scanning every byte of every other column between one X value and the next. Parquet's layout makes "read only these columns" a matter of seeking to specific byte ranges, nothing more.

---

## Encoding vs. Compression: Two Separate, Stacked Layers

These get conflated constantly. They are not the same operation, and Parquet applies both, in order: **encoding** first (structural, exploits known patterns in the data type), then **compression** (general-purpose byte-level squeezing) on top of the encoded bytes.

**Common encodings:**
- **Dictionary encoding** (default for most columns below a cardinality threshold): store distinct values once in a dictionary page, then store per-row integer indices into that dictionary. `event_type` with 5 distinct values over 1M rows becomes a 5-entry dictionary + 1M small integers instead of 1M repeated strings.
- **RLE (Run-Length Encoding) + bit-packing**: used for dictionary indices and repetition/definition levels. Runs of the same value collapse to `(value, count)`; low-cardinality integer ranges get packed into the minimum number of bits (e.g. values 0-4 only need 3 bits each, not a full 32-bit int).
- **Delta encoding** (DataPageV2): for sorted/near-sorted integer or byte-array columns (timestamps, auto-increment IDs), store the delta from the previous value instead of the absolute value — deltas are small and compress extremely well.

**Compression codecs** (applied after encoding, per page): Snappy (fast, low ratio, long-time default), Gzip/zlib (higher ratio, slower), **Zstd** (now the common recommendation — better ratio than Snappy at comparable speed, tunable compression level), LZ4, Brotli. Codec choice is a per-write configuration, not a file-format constraint — different row groups in principle could even use different codecs, though in practice a table uses one codec consistently.

**Worked example — why this stacking matters:**

```
Column: event_type, 1,000,000 rows, 5 distinct values ("click","view","purchase","add_to_cart","remove")

Raw (no encoding): 1,000,000 rows × ~8 bytes avg string = ~8 MB

Dictionary encoding:
  Dictionary page: 5 strings × ~8 bytes = 40 bytes
  Data: 1,000,000 × 3-bit index (2^3=8 ≥ 5 values) ≈ 375 KB
  → ~21x smaller than raw, before compression

Zstd compression on top of the dictionary indices (highly repetitive small ints):
  → additional ~3-5x reduction typical for skewed categorical data
  → final size: roughly 75-125 KB for what started as ~8 MB raw
```

The encoding step (dictionary + RLE) does most of the structural work for low-cardinality columns; compression squeezes further redundancy out of the already-encoded bytes. Skipping straight to "just gzip the raw column" would compress worse and slower than encode-then-compress.

---

## Statistics and Predicate Pushdown: Three Tiers of Pruning

Parquet supports pruning at three granularities, each catching what the previous one can't:

1. **Row-group statistics** (in every Parquet file): per-column min/max, null count, distinct count, stored in row group metadata. A reader can skip an entire row group without reading any of its column chunks if the predicate range can't overlap `[min, max]`.
2. **Page Index** (ColumnIndex + OffsetIndex, stored near the footer, separate from row group metadata so readers not doing selective scans don't pay to deserialize it): the **ColumnIndex** stores per-*page* min/max, letting a reader skip individual pages within a column chunk that survived the row-group-level filter. The **OffsetIndex** maps row indices to byte offsets so the reader can seek directly to the surviving pages instead of scanning the whole column chunk.
3. **Bloom filters** (optional, opt-in per column): a probabilistic membership structure for point-lookup predicates (`col = value`) that min/max stats can't help with — a column with min=1, max=9999999 gives zero pruning power for `WHERE user_id = 42` via stats alone, but a bloom filter can say "42 is definitely not in this row group" with zero false negatives.

```
WHERE user_id = 42 AND event_type = 'purchase'

Tier 1 (row group stats):   user_id [1, 9999999] → can't rule out row group → keep looking
Tier 2 (page index):        within surviving row group, only pages with
                             event_type range touching 'purchase' get read
Tier 3 (bloom filter):      if configured on user_id, check "is 42 possibly
                             present?" before even opening the row group
```

This is exactly the mechanism `z-ordering.md` and `liquid-clustering.md` in this KB are optimizing the *input* to — those techniques reorder rows so tier-1 and tier-2 stats end up tight instead of spanning the full column domain. Parquet doesn't know or care how the rows got sorted; it just stores whatever stats result from the order it was handed.

---

## Schema Evolution

Parquet files are self-describing (schema is in the footer), but a single physical file's schema is fixed once written. "Schema evolution" in practice means:

- **Column addition**: new files include the new column; readers merging old + new files fill missing values with `null` for old files, matched by field name (or field ID, if the writer assigns one — Iceberg always does this).
- **Column reordering / rename**: safe if matching is done by field ID (Iceberg's approach) rather than by name/position — renaming a column doesn't require touching existing Parquet files. Matching by name alone breaks on rename.
- **Type widening**: e.g. `int32` → `int64` is safe to read across old/new files (widening); narrowing or incompatible type changes are not safely mergeable and force a rewrite.

This is precisely why Iceberg and Delta don't ask Parquet to solve schema evolution — they layer field-ID-based schema resolution (Iceberg) or a JSON schema history in `_delta_log/` (Delta) on top, and Parquet stays a dumb, fixed-schema-per-file storage layer underneath.

---

## Where Iceberg/Delta Sit Relative to Parquet

```
┌─────────────────────────────────────────┐
│  Iceberg / Delta Lake                    │
│  - Snapshot isolation, transaction log   │
│  - Manifest/log-level min/max stats      │  ← lets engines skip whole FILES
│  - Schema evolution by field ID          │     before opening any Parquet file
│  - Partition/clustering metadata         │
└──────────────────┬────────────────────────┘
                    │ wraps
                    ▼
┌─────────────────────────────────────────┐
│  Apache Parquet files                    │
│  - Row group / page stats                │  ← lets engines skip row groups/pages
│  - Column chunk layout                   │     WITHIN a file already opened
│  - Encoding + compression                │
└─────────────────────────────────────────┘
```

Iceberg manifests and Delta's `_delta_log/` duplicate a *coarser* copy of Parquet's own min/max stats at the file level, specifically so an engine can decide which Parquet files to open at all without touching the filesystem for each one. Once a file is opened, Parquet's own row-group/page stats take over for pruning inside that file. Both layers exist because they solve pruning at different physical granularities — file-level (avoid an S3 GET entirely) vs. within-file (avoid decompressing irrelevant bytes).

---

## Key Gotchas

- **Row group size is a tuning knob, not a spec constant**: default target is commonly ~128MB but is writer-configurable. Larger row groups mean fewer, coarser stats (less pruning power) but less per-file metadata overhead; smaller row groups mean finer pruning but more footer/metadata bytes and more page-index entries to deserialize. Very small row groups on huge tables can make metadata overhead dominate.
- **Dictionary encoding silently falls back**: if a column's distinct-value count exceeds the writer's dictionary size threshold mid-file, the writer falls back to plain encoding for that column chunk — this is automatic and correct, but means you can't assume dictionary encoding held for every chunk just because it applied to the first one you inspected.
- **Bloom filters must be explicitly enabled per column at write time** (`parquet.bloom.filter.enabled#col`) — they are not on by default, add write-time cost and file-size overhead (~1-2% of column size at typical false-positive rates), and only help point-lookup predicates, not ranges.
- **Nested/repeated fields (structs, arrays, maps) use Dremel-style definition/repetition levels**, not a simple flat column layout — this is why deeply nested schemas with many optional/repeated fields can have surprisingly large per-value overhead relative to flat schemas, independent of the actual data size.
- **Page index and bloom filters are opt-in additions, not universal**: older files or writers that don't enable them fall back to row-group-only pruning. Don't assume every Parquet file in a table has page-level pruning available — check what the specific writer/engine version actually emitted.
- **Statistics can be wrong or missing on legacy/adversarial files**: engines historically had correctness bugs around statistics for certain types (e.g., some string comparison edge cases) — modern readers apply guardrails, but this is why Iceberg/Delta prefer to also track their own manifest-level stats rather than trusting Parquet stats blindly for correctness-critical pruning.
- **Delta encoding (DataPageV2) isn't universally the default**: many writers still default to DataPageV1 with dictionary+RLE; delta encodings for sorted numeric/timestamp columns require DataPageV2 support on both writer and reader — verify your engine's Parquet writer version actually emits V2 pages before assuming delta encoding is in play.

---

*Grounded against apache/parquet-format (GitHub), parquet.apache.org docs (Page Index spec), and the "State of Apache Parquet in 2026" community writeups as of September 2026. Format spec is at 2.12.0; 2026 additions include a native Variant logical type (Feb 2026) and native Geometry/Geography logical types for geospatial data — both recent enough to re-verify before relying on them in a specific engine's support matrix.*
