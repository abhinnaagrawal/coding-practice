# LSM Trees (Log-Structured Merge Trees)

## 30-Second Intuition

Random disk writes are catastrophically slow (spinning disk seeks ~10ms each, so ~100 writes/sec max). LSM trees fix this by never doing random writes — all writes are sequential. The tradeoff: reads get more complex because data is spread across multiple sorted files, and a background compaction process continuously merges those files to keep reads manageable.

---

## The Write Problem

### Why Random Writes Kill You

A B-Tree update modifies a page in-place, anywhere on disk:

```
Write "user_123" → page at byte offset 4,827,392
Write "user_456" → page at byte offset 102,918,771
Write "user_789" → page at byte offset 9,003,412
```

Each write = seek (move disk head) + rotational wait + write. Spinning disk: ~10ms seek → 100 IOPS max. Even SSDs, while better (~100μs random write), still suffer from write amplification at the flash translation layer.

### The LSM Solution: Sequential Everything

```
All writes → append to WAL (sequential)
           → insert into MemTable (in-memory sorted structure)
           → periodic flush to SSTable (sequential write to new file)
```

Every byte that hits disk is written sequentially. No seeks. SSDs love this. Spinning disks love this even more.

---

## Core Structure

```
WRITE PATH:
  Client Write
       │
       ▼
  ┌─────────┐
  │   WAL   │  ← append-only log (crash recovery)
  └─────────┘
       │
       ▼
  ┌─────────────────┐
  │   MemTable      │  ← sorted in-memory structure (SkipList/RBTree)
  │  (mutable)      │    e.g., 64MB
  └─────────────────┘
       │ (when full)
       ▼
  ┌─────────────────┐
  │   MemTable      │  ← becomes immutable, awaiting flush
  │  (immutable)    │
  └─────────────────┘
       │ (background flush)
       ▼
  ┌─────────────────┐
  │   L0 SSTables   │  ← sorted, immutable files. May have overlapping key ranges
  │  file1, file2   │
  └─────────────────┘
       │ (compaction)
       ▼
  ┌─────────────────┐
  │   L1 SSTables   │  ← non-overlapping key ranges, bounded total size
  └─────────────────┘
       │
       ▼
  ┌─────────────────┐
  │   L2 SSTables   │  ← 10× larger than L1
  └─────────────────┘
       ...
  ┌─────────────────┐
  │   Ln SSTables   │  ← bulk of data lives here
  └─────────────────┘
```

**Key invariant** (leveled compaction): Each level Ln has non-overlapping key ranges. A given key exists in at most one SSTable per level (except L0).

---

## Write Path: Step by Step

**Example**: write `("user_123", {name: "Alice", age: 30})`

1. **WAL append**: record written to write-ahead log. If crash here, replay WAL on restart.
2. **MemTable insert**: key inserted into SkipList. O(log n). Immediately visible to reads.
3. **MemTable full** (~64MB): MemTable becomes immutable. New MemTable created for incoming writes. Writes never stall here.
4. **Background flush**: immutable MemTable flushed to L0 SSTable. Sequential disk write. WAL segment can be discarded.
5. **L0 accumulates** files. When file count hits threshold (e.g., 4 files), compaction triggers.

---

## Read Path: Step by Step

**Example**: read `"user_123"`

```
1. Check MemTable          → found? return (most recent write wins)
2. Check immutable MemTables → found? return
3. Check L0 SSTables (newest first) → check bloom filter first!
   - bloom says "maybe" → read SSTable index, seek to key, check data block
   - bloom says "no"    → skip entirely (no false negatives)
4. Check L1 SSTables → binary search on index (non-overlapping, so check at most 1 file)
5. Check L2, L3, ...
6. If not found anywhere → key does not exist
```

**Worst-case reads** = check every SSTable at every level. Bloom filters make this manageable.

### Bloom Filters

A probabilistic data structure that answers "is key K definitely NOT in this SSTable?"

```
bits_per_key = 10  →  ~1% false positive rate
bits_per_key = 20  →  ~0.01% false positive rate
```

False negatives: impossible (if key is in SSTable, bloom always says "maybe").
False positives: bloom says "maybe" but key isn't there → unnecessary SSTable read.

With bloom filters, a point lookup in a healthy LSM typically touches O(1) SSTables even across many levels, because each level check is a fast bloom filter, and the non-overlapping property at L1+ means at most one SSTable per level needs inspection.

---

## Compaction — The Heart of LSM

Without compaction, L0 grows unbounded → reads check every L0 file → performance degrades.

### Why Compaction Exists

```
Without compaction after 1000 writes:
  L0: [file1, file2, ..., file250]   ← all may contain key "user_123"
  Read "user_123": must check all 250 files

With compaction:
  L0: [file1, file2, file3, file4]
  L1: 1 file covering key "user_123"
  Read "user_123": bloom check 4 L0 files + 1 L1 lookup
```

### Size-Tiered Compaction (Cassandra, ScyllaDB)

Merge SSTables of similar size together into one larger SSTable.

```
4 × 10MB files → merge → 1 × 40MB file
4 × 40MB files → merge → 1 × 160MB file
```

- **Good for**: write-heavy workloads (low write amplification)
- **Bad for**: reads (overlapping key ranges within a tier, many SSTables to check)
- **Space amplification**: high (old versions of keys survive in old SSTables until next merge)

### Leveled Compaction (LevelDB, RocksDB)

Each level has a size limit (10× per level). Keys within a level are non-overlapping.

```
L0: 4 files (overlapping allowed, up to ~256MB)
L1: non-overlapping, total ~256MB
L2: non-overlapping, total ~2.56GB
L3: non-overlapping, total ~25.6GB
```

When L_n exceeds its size limit:
1. Pick one SSTable from L_n
2. Find all overlapping SSTables in L_{n+1}
3. Merge them all, write new SSTables to L_{n+1}
4. Delete the source SSTables

- **Good for**: reads (binary search per level, non-overlapping)
- **Bad for**: write amplification (data re-written many times as it moves through levels)

### FIFO Compaction

Drop the oldest SSTable when total size exceeds a limit. No merging.

```
SSTable1(oldest) → SSTable2 → SSTable3(newest)
When size > limit: delete SSTable1
```

- **Only valid for**: append-only time-series with TTL. Never use for general KV.
- **Write amplification**: 1 (no rewriting)
- **Read amplification**: O(files) — no merging means many files

### The Amplification Trilemma

You cannot minimize all three simultaneously. Pick two:

| Strategy      | Write Amp | Read Amp | Space Amp |
|---------------|-----------|----------|-----------|
| Leveled       | High      | Low      | Low       |
| Size-tiered   | Low       | High     | High      |
| FIFO          | ~1        | High     | Highest   |

**Write amplification (WA)**: bytes written to disk / bytes written by app.
- Leveled WA = O(levels × level_size_ratio). Typical: 10–30×
- Size-tiered WA = O(log_{size_ratio}(total_size)). Typical: 5–10×

**Read amplification (RA)**: SSTables read per point lookup.
- Leveled RA: O(L0_files + levels) ≈ ~10 with bloom filters effectively ~1-2
- Size-tiered RA: O(total_files) — much worse

**Space amplification (SA)**: disk_used / live_data_size.
- Leveled: ~1.1× (compaction keeps dead data brief)
- Size-tiered: up to 2× (old versions survive until tier merges)

---

## SSTable Internals

An SSTable (Sorted String Table) is an immutable file with this structure:

```
┌──────────────────────────┐
│  Data Block 0            │  ← sorted KV pairs, compressed (e.g., snappy)
│  "aardvark" → val        │    typically 4KB–64KB per block
│  "apple"    → val        │
│  ...                     │
│  "azimuth"  → val        │
├──────────────────────────┤
│  Data Block 1            │
│  "baby"     → val        │
│  ...                     │
│  "byzantine"→ val        │
├──────────────────────────┤
│  ...                     │
├──────────────────────────┤
│  Index Block             │  ← one entry per data block: (last_key, block_offset)
│  ["azimuth", offset=0]   │    used for binary search to find which block has key
│  ["byzantine", offset=4K]│
│  ...                     │
├──────────────────────────┤
│  Filter Block            │  ← bloom filter(s), one per data block
├──────────────────────────┤
│  Meta Index Block        │  ← offsets of filter block, compression dict, etc.
├──────────────────────────┤
│  Footer (fixed size)     │  ← magic number, pointers to index + meta index blocks
└──────────────────────────┘
```

**Read sequence for point lookup "baby"**:
1. Read footer (always at known offset from end of file) → get index block offset
2. Read index block → binary search → block containing "baby" is at offset 4K
3. Check filter block for block 1 → bloom says "maybe"
4. Read data block 1 → binary search → find "baby"

**Block cache**: caches decompressed data blocks in memory. First read pays decompression cost; subsequent reads are cache hits. Separate from OS page cache.

**Row cache** (optional, e.g., Cassandra): caches deserialized row objects. Eliminates even the block decode step for hot keys. Higher memory cost per entry.

**Two-level (partitioned) index** (RocksDB): for large SSTables (GBs), the index block itself becomes large. Partition it: top-level index points to index partitions, each partition covers a key range. Reduces memory footprint.

---

## Tombstones and Deletes

LSM trees cannot do in-place deletes — SSTables are immutable.

**Delete mechanism**: write a tombstone marker (a special KV entry with a deletion flag).

```
Write: ("user_123", value, seq=100)
Delete: ("user_123", TOMBSTONE, seq=150)

Read at seq=200: see tombstone at seq=150 → key is deleted
Read at seq=120: see value at seq=100 → key exists (snapshot read!)
```

**Tombstone danger**: if a tombstone gets compacted away before the data it covers, the old data re-appears (called "resurrection").

**Tombstone GC rules**:
- Tombstone can only be discarded during compaction if:
  - It has reached the bottommost level (no older data hiding below it), AND
  - The tombstone's sequence number is older than all active snapshots
- Most implementations have a `gc_grace_seconds` (Cassandra) or equivalent

**Tombstone accumulation**: write-heavy delete workloads can accumulate millions of tombstones, causing read slowdowns (have to scan past all of them). Monitor tombstone count per SSTable.

---

## Snapshots and MVCC

Each write gets a monotonically increasing **sequence number** (seqno).

```
Write "user_123" → (key="user_123", seqno=100, value="Alice")
Write "user_123" → (key="user_123", seqno=200, value="Alice Smith")  ← update
```

**Snapshot**: a specific seqno. Reads at that snapshot only see writes with seqno ≤ snapshot_seqno.

**Point-in-time reads**:
```
Snapshot S=150: read "user_123" → returns "Alice"   (seqno 100 ≤ 150)
Snapshot S=250: read "user_123" → returns "Alice Smith" (seqno 200 ≤ 250)
```

**Snapshot pinning problem**: compaction can only GC entries that are not visible to any active snapshot. A long-lived snapshot forces the LSM to retain all historical versions → space amplification explodes.

```
Active snapshot at seqno=5 (from an old read transaction):
  ← compaction cannot delete any version written after seqno=5
  ← all updates since seqno=5 must be kept
```

**Production implication**: monitor snapshot age. A stuck iterator/transaction in RocksDB/LevelDB can prevent all compaction GC.

---

## LSM vs B-Tree

| | LSM Tree | B-Tree |
|---|---|---|
| Write throughput | High (sequential) | Lower (random) |
| Read throughput | Lower (check multiple files) | Higher (single tree traversal) |
| Space amplification | Medium–High | Low (~1.1×) |
| Write amplification | Medium (leveled: 10–30×) | High for random updates (~page size / record size) |
| Range scans | Excellent (merge iterators) | Excellent (leaf page traversal) |
| Crash recovery | WAL replay | WAL or redo log |
| Write stalls | Yes (compaction debt) | Rare |

**When LSM wins**: sustained write throughput > ~1000 writes/sec, especially with high cardinality (many different keys). Cassandra, LevelDB, RocksDB, HBase, BigTable.

**When B-Tree wins**: read-heavy workloads, point lookups dominate, predictable latency required. PostgreSQL, MySQL InnoDB, SQLite.

**The crossover**: a B-Tree update to a large page for a small value = high write amplification. At sustained ~1000+ writes/sec, LSM's sequential write advantage overcomes its compaction overhead.

---

## Concrete End-to-End Example

### Write: `"user_123" → {name: "Alice", age: 30}`

```
t=0ms:   WAL.append(key="user_123", value=json_bytes, seqno=42)
          → disk: sequential write to wal.log, 1 μs

t=0.01ms: MemTable.insert("user_123", value, seqno=42)
          → SkipList insert, O(log n), no disk I/O

t=later:  MemTable reaches 64MB threshold
          → becomes immutable MemTable
          → new empty MemTable created for new writes
          → background thread starts flush

t=flush:  Read immutable MemTable in sorted order
          → write sorted KV pairs to new L0 SSTable: "sst-00042.sst"
          → write index block, filter block, footer
          → fsync
          → delete WAL segment covering these keys

t=compact: L0 has 4 files now. Compaction triggered.
           → merge L0 files + overlapping L1 files
           → write new L1 SSTables with non-overlapping ranges
           → "user_123" ends up in L1 file covering "user_000" to "user_500"
           → delete old L0 files and merged L1 files
```

### Read: `"user_123"`

```
1. MemTable.get("user_123")          → not found (flushed)
2. Immutable MemTables               → none active
3. L0 files (newest first):
   sst-00050.sst: bloom.check("user_123") → "no"  → skip
   sst-00049.sst: bloom.check("user_123") → "no"  → skip
   (these were written after user_123 and don't contain it)
4. L1: find SSTable covering "user_123" range
   sst-l1-00010.sst covers "user_000"–"user_500"
   bloom.check("user_123")           → "maybe"
   Read index block: block 7 contains "user_1xx" keys
   Read data block 7: binary search → "user_123" found!
   Return {name: "Alice", age: 30}
```

**Total disk reads for a warm cache**: 0 (block cache hit on L1 data block). Cold: 1 data block read after following index.

---

## Deep Internals

### MemTable Implementation

Common choices:
- **SkipList** (RocksDB default): O(log n) insert/lookup, concurrent reads with single-writer. Lock-free reads possible.
- **Red-Black Tree**: same complexity, higher constant factor, used in older implementations.
- **HashSkipList**: hash on key prefix → bucket → SkipList. O(1) for prefix-match point lookups, worse for range scans.

### Merge Semantics During Compaction

When compacting two SSTables both containing `"user_123"`:
```
SSTable A: ("user_123", seqno=100, value="Alice")
SSTable B: ("user_123", seqno=200, value="Alice Smith")

Merge result: keep seqno=200, discard seqno=100
(if seqno=100 is below all active snapshots)
```

The merge is a k-way merge of sorted iterators (like merge sort). Output is written sequentially.

### Compaction Score (RocksDB)

RocksDB continuously computes a "score" per level:
```
L0 score = L0_file_count / level0_file_num_compaction_trigger
Ln score = current_size / max_bytes_for_level_n
```

Level with highest score > 1.0 gets compacted next. Ensures no level gets too far ahead.

### WAL Group Commit

Multiple concurrent writes are batched:
```
Thread 1: write "key_A"  ─┐
Thread 2: write "key_B"  ─┼─→ single WAL fsync for all → huge throughput gain
Thread 3: write "key_C"  ─┘
```

Without group commit: 3 fsyncs. With group commit: 1 fsync. At 200μs/fsync, this is a 3× throughput improvement.

---

## Key Gotchas

1. **Compaction debt is catastrophic**: if write rate exceeds compaction throughput, L0 file count grows unbounded → reads check every L0 file → latency spikes → eventual write stall. Watch `rocksdb.num-files-at-level0` religiously.

2. **Tombstone accumulation**: deleting lots of data without compaction completing leaves tombstones everywhere. Reads must scan past them. In Cassandra: `gc_grace_seconds` must expire and compaction must run before tombstones are GC'd.

3. **Snapshot pinning**: a long-lived snapshot (e.g., a backup process, a stuck iterator) prevents compaction from GC'ing old versions. Space amplification can explode from 1.1× to 5× overnight.

4. **L0 is special**: L0 SSTables can have overlapping key ranges (they're just flushed MemTables). Every read must check all L0 files. Keep L0 file count small (default trigger: 4 files).

5. **Write stalls are binary**: LSM writes either go at full speed or stall completely (when L0 file count hits `stop_writes_trigger`). There's no graceful degradation — it's a cliff. Build backpressure into your write path before hitting this.

6. **Bloom filter memory is not free**: 10 bits/key × 1 billion keys = ~1.2GB just for bloom filters. At 20 bits/key, it's 2.4GB. This memory lives outside the block cache and must be accounted for separately.

7. **Range scans bypass bloom filters**: bloom filters only help point lookups. A range scan `[user_100, user_200]` must check all SSTables whose key range overlaps the scan range. For range-heavy workloads, keep levels compact and consider clustering keys appropriately.

8. **Intra-L0 ordering matters**: within L0, SSTables are ordered by creation time (flush sequence). Reads must check L0 files newest-first to get the latest value. If L0 has 10 files, that's 10 bloom filter checks minimum per read.
