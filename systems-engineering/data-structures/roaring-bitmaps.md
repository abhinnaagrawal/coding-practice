# Roaring Bitmaps

## 30-second intuition

A roaring bitmap represents a set of integers by splitting the 32-bit space into 65,536 "chunks" (top 16 bits = chunk key) and, per chunk, automatically picking whichever of three storage representations — sorted array, dense bitmap, or run-length-encoded ranges — is smallest/fastest for that chunk's actual data. The result is a set structure that is simultaneously as fast as a naive bitmap for dense data and far smaller than one for sparse data, with set operations (AND/OR/XOR/NOT) computed container-by-container using the best algorithm for each container type.

---

## The core problem

You need to represent large sets of integers (user IDs, document IDs, row IDs) and support fast set operations — union, intersection, difference — over them. Two naive extremes:

**Naive bitmap** (one bit per possible value, 0..N):
```
value:  0 1 2 3 4 5 6 7 8 9 ...
bit:    1 1 1 1 1 0 0 0 0 0 ...
```
- If the universe is small and the set is **dense**, this is excellent: 1 bit per element, cache-friendly sequential scans, trivially SIMD-able word-at-a-time AND/OR/XOR.
- If the universe is large (say, max value 10,000,000) and the set is **sparse** (say, 100 elements), you still need `10,000,000 / 8` bytes = ~1.25 MB to represent 100 numbers — wasteful.

**Naive list/hash set** — memory-proportional to element count, but set operations (intersection especially) are comparatively slow and cache-unfriendly compared to bitwise ops on a dense bitmap.

Real-world sets are neither uniformly dense nor uniformly sparse — they're often **clustered** (e.g. IDs 1–5, then a gap, then a dense run at 100,000–105,000). Roaring is designed exactly for this shape.

---

## Roaring's key idea: partition + per-chunk best representation

Split each 32-bit integer into:
- **high 16 bits** → chunk key (one of 65,536 possible chunks, each covering a contiguous range of 2^16 = 65,536 values)
- **low 16 bits** → the value's position within that chunk

Each chunk ("container") independently picks its storage format based on density:

```
┌─────────────────────────────────────────────────────────┐
│  32-bit value                                            │
│  ┌──────────────┬──────────────┐                          │
│  │  high 16 bits │  low 16 bits │                          │
│  │  (chunk key)  │ (in-chunk val)│                         │
│  └──────────────┴──────────────┘                          │
└─────────────────────────────────────────────────────────┘

  chunk 0        chunk 1        chunk 76        ...
  ┌────────┐    ┌────────┐    ┌────────┐
  │ Array  │    │ Bitmap │    │  Run   │
  │container│   │container│   │container│
  └────────┘    └────────┘    └────────┘
```

### Array container
Used when the chunk has **few values** (roaring's threshold: fewer than 4096 values in that chunk, i.e. < 6.25% density). Stores the low-16-bit values as a **sorted array of 16-bit shorts**.
- Size: 2 bytes × count. E.g. 5 values → 10 bytes.
- Fast for sparse data: intersection/union via a merge-join over two sorted arrays (like merging sorted lists), which is cache-friendly and simple.

### Bitmap container
Used when the chunk has **many values** (≥ 4096 in that chunk, i.e. ≥ 6.25% density up to fully dense = 65,536 values). Stores a **fixed-size 8 KB dense bitmap** (2^16 bits = 65,536 bits = 8192 bytes) regardless of exact count.
- Size: always 8 KB, no matter whether it's 4096 or 65,536 values set.
- Fast for dense data: word-at-a-time (64-bit word) AND/OR/XOR, vectorizable with SIMD (AVX2/AVX-512/NEON in CRoaring).

### Run container (RLE)
Used when the chunk has **long runs of consecutive values** (e.g. values 1000–5000 all present). Stores as a list of **(start, length) pairs**.
- Size: proportional to the number of runs, not the number of values — a single run of 4000 consecutive values costs just one (start, length) pair (4 bytes), vastly beating both array (4000×2 bytes) and bitmap (8192 bytes) representations.
- Best for sequential/clustered data — exactly the shape you get from auto-incrementing IDs inserted in bulk, or timestamp-derived IDs.

### Container conversion

Roaring automatically **converts a container's representation as elements are added or removed**, always keeping the most compact valid form:
- Array container crosses 4096 elements → convert to bitmap container (array would now be larger than the fixed 8 KB bitmap).
- Bitmap container drops below 4096 elements (after removals) → convert back to array.
- Runs are typically chosen at construction/optimization time (`runOptimize()` in most implementations) by scanning for long consecutive stretches, rather than continuously maintained on every insert — most libraries expose an explicit "optimize" step you call after bulk-loading data, since maintaining run-encoding on every single add/remove would add overhead to write-heavy paths.

---

## Concrete example

Set: `{1, 2, 3, 4, 5, 100000, 100001, 5000000}`

Partition each value by `value >> 16` (chunk key) and `value & 0xFFFF` (low bits):

| Value | Chunk key (high 16 bits) | Low 16 bits |
|---|---|---|
| 1, 2, 3, 4, 5 | 0 | 1, 2, 3, 4, 5 |
| 100000, 100001 | 1 (since 100000 = 65536 + 34464) | 34464, 34465 |
| 5000000 | 76 (since 5000000 = 76×65536 + 21824) | 21824 |

Resulting roaring bitmap:

```
chunk 0  -> Array container  [1, 2, 3, 4, 5]              (5 values, tiny — array wins)
chunk 1  -> Array container  [34464, 34465]               (2 values, consecutive but too
                                                             few to matter vs. array; a
                                                             run container of 1 run would
                                                             also be valid/possibly chosen)
chunk 76 -> Array container  [21824]                      (1 value)
```

None of these chunks reach the 4096-value bitmap threshold, so everything stays as compact arrays — total size is on the order of a few dozen bytes, versus a naive bitmap over a universe of 5,000,000+ which would need ~625 KB. This is the essential win: **roaring's cost is proportional to how the data is actually distributed, not to the size of the value universe.**

If instead the set were `{0, 1, 2, ..., 60000}` (60001 consecutive values, all in chunk 0), roaring would use a **run container**: one (start=0, length=60001) pair — a few bytes — instead of a bitmap container (8 KB) or array container (120 KB for 60001 shorts). If the set were random noise touching 50,000 of the 65,536 slots in chunk 0, it would use a **bitmap container** (8 KB, still cheaper and faster than a 100 KB array).

---

## Set operations

AND/OR/XOR/NOT are computed **container-by-container**, matching up chunks by key and dispatching to the fastest algorithm for the container-type pair involved:

| Container A | Container B | Algorithm |
|---|---|---|
| Array ∩ Array | merge-join over sorted arrays (like merging two sorted lists) | O(min size) |
| Bitmap ∩ Bitmap | word-wise AND over 64-bit (or SIMD 256/512-bit) words | O(8KB), vectorized |
| Array ∩ Bitmap | binary/linear probe of array elements against the bitmap | O(array size) |
| Run ∩ Run | interval-intersection over sorted (start,length) pairs | O(runs) |
| Array ∪ Array | sorted merge | O(sum of sizes) |
| Bitmap ∪ Bitmap | word-wise OR | O(8KB), vectorized |

Chunks present in only one operand are handled directly (e.g. for OR, just copy them through; for AND, they contribute nothing). This container-level dispatch is why roaring operations tend to be **both faster and smaller than naive bitmap operations on real-world sparse/clustered data**: naive bitmaps always pay O(universe size) for every operation regardless of actual data distribution, while roaring's cost tracks the data's actual footprint (few tiny containers instead of one huge one), and picks the cheapest applicable primitive per chunk.

---

## Real-world adopters

- **Lucene / Elasticsearch / OpenSearch**: postings lists (which documents contain term X) are stored and intersected using roaring-like structures for fast boolean queries at scale.
- **ClickHouse**: bitmap functions (`bitmapBuild`, `bitmapAnd`, etc.) backed by roaring for fast set-membership/intersection analytics.
- **Apache Druid, Apache Pinot**: bitmap indexes over dimension values (classic "bitmap index" technique from OLAP, roaring is the modern compressed variant) for fast filter evaluation over large columnar datasets.
- **Apache Spark**: internal bitset-backed structures for partition pruning / row-group filtering use similar ideas (not always roaring specifically, but the same compressed-bitmap philosophy).
- **InfluxDB**: uses roaring-style bitmap indexes for tag/series filtering.
- **Pilosa / FeatureBase**: a database built *natively* around roaring bitmaps as the core storage/query primitive — every value-to-row mapping is a bitmap, and queries are bitmap boolean algebra.
- **CRoaring** (the C/C++ reference for performance) is itself embedded in Apache Doris, Redpanda, YDB, StarRocks, and others — roaring has become close to a *lingua franca* compressed-bitmap format across the OLAP/search ecosystem.

---

## RBAC/ABAC application (the main point of interest)

This is where roaring bitmaps intersect directly with authorization systems like Zanzibar/SpiceDB (see `authz/zanzibar.md`, `authz/spicedb.md`).

### Permission bitmaps

- **Per-permission/role bitmap**: "set of user IDs who have permission X" → one roaring bitmap. Membership test ("does user 4821 have permission X") is a single bit test — O(1) after the bitmap is built, and the bitmap itself is compact even for millions of users.
- **Per-user visibility bitmap**: "set of resource IDs visible to user U" → one roaring bitmap per user (or computed on demand from role bitmaps). This is the structure that makes **bulk filtering** cheap.

### Bulk filtering — the main win

"Show me all documents visible to user U, out of a 10-million-document search-query result set" becomes:

```
result_bitmap = query_matches_bitmap  AND  user_visible_bitmap
```

One bitmap AND operation, container-by-container, instead of 10 million individual permission checks. This is **exactly** how OpenSearch/Elasticsearch document-level security (DLS) and Pilosa/FeatureBase-style permission filtering work in practice: the search engine already has a bitmap of "documents matching this query" (postings-list intersections are already bitmap ANDs); adding a security filter is just one more AND against a "documents this user can see" bitmap.

### Role hierarchy via OR

Effective permission = union of every role a user belongs to:

```
user_permissions = role_admin_bitmap OR role_editor_bitmap OR role_viewer_bitmap
```

Adding a user to a new role that grants access to a resource set is just merging that role's bitmap into the user's effective bitmap (or, more commonly, keeping resource visibility computed lazily as an OR over the roles the user currently holds, recomputed on role change).

### ABAC attribute filtering

Each attribute *value* maps to a bitmap of matching resources — e.g.:

```
dept_eng_bitmap      = { resources tagged department=eng }
clearance_secret_bitmap = { resources tagged clearance=secret }

policy_result = dept_eng_bitmap AND clearance_secret_bitmap
```

This resolves an ABAC policy like "department=eng AND clearance=secret" as a single AND across two attribute bitmaps — precisely the same trick as classical **bitmap indexes** in analytical databases (Druid, Pinot use this for dimension filtering), just applied to authorization attributes instead of query dimensions.

### Contrast with Zanzibar/SpiceDB's graph-traversal approach

| | Zanzibar/SpiceDB (ReBAC) | Roaring-bitmap materialized sets |
|---|---|---|
| Computation model | Graph traversal at Check-time — recursively walk relations/usersets | Precomputed/maintained bitmap per permission or per user, queried via bit test/AND |
| Expressiveness | Highly expressive: arbitrary nesting, arrows through resource hierarchies, intersection/exclusion on the fly | Effectively a materialized snapshot of a computed set — doesn't natively express relationship graphs, only *the result* of evaluating one |
| Best for | Point checks (`can user X do Y to Z`), correctness-critical revocation, complex org/sharing hierarchies | Bulk filtering / list views over large resource sets — "give me everything user X can see" |
| Cost profile | Cost grows with relation-graph depth/fan-out per Check call | Cost is ~O(1) bit test after the bitmap exists; the cost is pushed to *maintaining* the bitmap (invalidation on every permission change) |
| Freshness | Real-time correctness via zookies/ZedTokens (see `authz/zanzibar.md`) | Only as fresh as the last bitmap rebuild/invalidation — needs an event stream (e.g. Watch API) to stay current |

### Hybrid pattern used in practice

The two approaches are complementary, not competing:

1. **Source of truth**: keep Zanzibar/SpiceDB (or an equivalent ReBAC system) as the authoritative, real-time-correct permission model — write relationships on share/unshare, call `CheckPermission` for individual access-critical decisions.
2. **Materialized cache for bulk filtering**: rather than calling the expensive `LookupResources`/`LookupSubjects` APIs on every list-view request (see `authz/spicedb.md` for why those are costly), maintain a **roaring bitmap cache** of "resource IDs visible to user X" (or "user IDs who can see resource Y"), refreshed incrementally by subscribing to the **Watch API** stream of relationship changes.
3. This gets you: ReBAC's correctness and expressiveness for the authorization model itself, plus bitmap-AND-speed for the "filter a huge candidate set down to what this user can see" query pattern that ReBAC systems are comparatively slow at.

This pattern shows up directly in how OpenSearch-style document-level security is commonly bolted onto a Zanzibar-backed app: index-time or near-real-time bitmap materialization of visibility sets, invalidated by permission-change events, with the ReBAC system remaining the system of record.

---

## Libraries

| Language | Library |
|---|---|
| Java | `RoaringBitmap` (the reference implementation) |
| C/C++ | `CRoaring` (SIMD-optimized: AVX2, AVX-512, NEON) |
| Python | `pyroaring` (wraps CRoaring) |
| Rust | `roaring-rs` (pure Rust) / `croaring-rs` (FFI wrapper over CRoaring) |
| Go | `RoaringBitmap/roaring` |
| Node.js | `roaring-node` |
| SQL/Postgres | `pg_roaringbitmap` extension |
| Redis | `redis-roaring` module |

---

## Concrete example: bitmap-based visibility filter + Zanzibar equivalent

**Bitmap approach:**

```
role_bitmaps = {
  "org:acme:admin":  RoaringBitmap([1, 5, 9]),        // user IDs
  "org:acme:editor": RoaringBitmap([1, 2, 3, 5, 7]),
  "org:acme:viewer": RoaringBitmap([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]),
}

doc_visibility = {
  "doc:42": role_bitmaps["org:acme:viewer"],   // anyone in viewer role can see doc 42
}

# "Can user 7 see doc:42?"
doc_visibility["doc:42"].contains(7)   # -> True, O(1) bit test

# "Which of these 10M search results can user 3 see?"
result = query_match_bitmap & user_visible_bitmap[3]   # single AND op
```

**Equivalent Zanzibar/SpiceDB tuple model:**

```
definition user {}
definition organization {
  relation admin:  user
  relation editor: user
  relation viewer: user
  permission view = viewer + editor + admin
}
definition document {
  relation org: organization
  permission view = org->view
}
```
```
organization:acme#viewer@user:1
organization:acme#viewer@user:2
...
document:doc42#org@organization:acme
```
`CheckPermission(document:doc42, view, user:7)` walks `org->view` → `organization:acme`'s `view` (union of viewer/editor/admin) → finds `user:7` in `viewer` tuples → allowed.

**Trade-off**: the tuple model is more expressive (org membership can change independently, nested groups can join `viewer`, folder-style hierarchies compose naturally) but a single Check is a graph walk, and answering "which of these 10M documents can user 7 see" means either a LookupResources call (expensive) or maintaining exactly the bitmap cache shown above, refreshed whenever the underlying tuples change.

---

## Deep internals

- Bitmap containers are always exactly 8 KB regardless of how many of the 65,536 bits are set — this is a deliberate simplicity/speed trade-off (fixed-size containers are trivial to mmap, cache-align, and SIMD over) rather than a further-compressed dense representation.
- The 4096-element crossover threshold for array→bitmap conversion is derived from the point where `4096 × 2 bytes = 8192 bytes` — exactly equal to the bitmap container's fixed 8 KB size; below that, arrays are strictly smaller.
- Run containers are typically **not maintained continuously on every insert** in most implementations — they're produced by an explicit `runOptimize()` call after bulk construction, because maintaining a run-length encoding under arbitrary single-element inserts/removals is more expensive than array/bitmap mutation. This matters operationally: if you build a roaring bitmap incrementally and never call the optimize step, you may miss the size/speed benefits of run containers even when your data is highly sequential.
- SIMD acceleration (AVX2/AVX-512/NEON in CRoaring) applies specifically to bitmap-container word operations — array and run containers benefit less directly from wide SIMD and more from cache-friendly sequential access patterns and merge-style algorithms.
- Serialization format is standardized enough that Java's RoaringBitmap, CRoaring, and roaring-rs can read/write each other's on-disk/wire format — useful for cross-language pipelines (e.g. a Java analytics service producing bitmaps consumed by a Rust or Python query layer).
- Roaring bitmaps assume **non-negative integers up to 32 bits** natively (some implementations extend to 64-bit via an extra level of indirection/sharding, e.g. `Roaring64Bitmap` in Java) — for 64-bit ID spaces (e.g. Snowflake IDs, UUIDs truncated to int64), check whether your chosen library's 64-bit variant is efficient for your ID distribution before assuming it behaves the same as the 32-bit case.

## Key gotchas

- Don't forget `runOptimize()`/equivalent after bulk-loading sequential data — without it you keep array/bitmap containers even where a run container would be dramatically smaller and faster.
- Bitmap containers cost a fixed 8 KB per chunk even at very low density above the 4096 threshold — if your ID space is such that many chunks sit just above that threshold with mostly-sparse-but-not-quite data, you can end up paying more memory than a naive array might have, though this is a narrow edge case roaring's threshold is tuned to minimize.
- Materialized permission bitmaps (the RBAC/ABAC use case) are only as correct as your invalidation pipeline — a bitmap cache that misses a Watch-API event silently serves stale (and potentially *over-permissive*) access decisions, reintroducing a variant of Zanzibar's "new enemy problem" at the cache layer. Treat bitmap cache staleness as a security property to test, not just a performance concern.
- 64-bit integer universes need a 64-bit-aware roaring variant (e.g. `Roaring64Bitmap`); using the 32-bit structure with truncated/hashed IDs risks collisions.
- Roaring is a *set* structure — it does not natively store associated values/metadata per element; if you need "user ID → last-checked timestamp" alongside membership, you need a parallel structure, not an extension of the bitmap itself.

## When to use / When NOT to use

- **Use roaring bitmaps** when: you need fast set operations (AND/OR/XOR) over large integer ID sets, especially search/analytics postings lists, OLAP-style dimension filtering, or materialized "visible resource set per user" caches for bulk authorization filtering in list views.
- **Don't use them** when: your set is small enough that a hash set/array is simpler and the compression benefit is negligible; your IDs aren't dense/sequential-ish integers (e.g. random UUIDs as strings — you'd need to map to integers first, which may not be worth the complexity for small sets); or you need per-element associated data rather than pure membership (use a map/columnar structure instead, possibly alongside a bitmap for the membership-only queries).
