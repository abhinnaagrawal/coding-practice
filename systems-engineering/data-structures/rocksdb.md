# RocksDB

## 30-Second Intuition

RocksDB is Facebook's embedded LSM-tree key-value store, forked from Google's LevelDB and tuned aggressively for SSDs and high-throughput workloads. It's a library you link into your process — no network, no server, just a `Put(key, value)` / `Get(key)` API. It powers Kafka's log storage, TiKV, CockroachDB, MyRocks (MySQL), and Flink's state backend because it combines high write throughput with fine-grained tuning control.

> Read [lsm-trees.md](lsm-trees.md) first if you're unfamiliar with LSM fundamentals.

---

## What RocksDB Is

- **Embedded library**: no client-server protocol. Link `librocksdb` into your process. Used in same-process storage engines.
- **LSM-tree**: MemTable → immutable MemTable → L0 SSTables → L1 → ... → Ln
- **SSD-first design**: LevelDB was designed for spinning disks. RocksDB adds parallelism, subcompaction, rate limiting, and compression choices that make sense for SSD I/O characteristics.
- **Key users**:
  - **Kafka**: KIP-405 tiered storage, KIP-956 native storage uses RocksDB
  - **MyRocks**: Facebook's MySQL storage engine (InnoDB replacement)
  - **TiKV**: storage layer of TiDB (distributed SQL)
  - **CockroachDB**: underlying KV store
  - **Flink**: `EmbeddedRocksDBStateBackend` for large keyed state (legacy `RocksDBStateBackend` name removed as of Flink 2.0)
  - **Spark**: state store for structured streaming

---

## Column Families

A RocksDB DB can have multiple **Column Families** (CFs) — logical partitions that share a WAL but have independent MemTables, SSTables, and compaction configurations.

```
DB "mydb"
├── CF: "default"    ← always exists, cannot be dropped
├── CF: "hot_data"   ← configured with FIFO compaction, 1GB limit
├── CF: "cold_data"  ← configured with leveled compaction
└── CF: "metadata"   ← small, frequently updated config data
```

**Why CFs matter**:
- Different data has different access patterns. Hot event data: write-heavy, TTL-based → FIFO. Historical data: read-heavy → leveled.
- Each CF has its own block cache allocation, bloom filter settings, compression config.
- Atomic writes across CFs: use `WriteBatch` — either all CF writes commit or none do.

```cpp
// Write atomically across two column families
WriteBatch batch;
batch.Put(cf_hot, "event_123", event_bytes);
batch.Put(cf_meta, "last_event", "event_123");
db->Write(WriteOptions(), &batch);
```

**CF isolation**: compaction on `cold_data` CF doesn't block `hot_data` CF. Each CF has its own compaction thread pool slot.

---

## Write Path Deep Dive

```
Client: db.Put("key", "value")
         │
         ▼
  ┌──────────────────────────────────────────────────────────┐
  │ WAL (Write-Ahead Log)                                    │
  │  ← append record                                        │
  │  ← group commit: batch multiple writes into one fsync   │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────────────────────────────────────────────────┐
  │ Active MemTable (per CF)                                 │
  │  ← SkipList insert (lock-free reads, single writer)     │
  │  ← O(log n), no disk I/O                                │
  └──────────────────────────────────────────────────────────┘
         │ (MemTable full → write_buffer_size hit)
         ▼
  ┌──────────────────────────────────────────────────────────┐
  │ Immutable MemTable queue                                 │
  │  ← new active MemTable created immediately              │
  │  ← writes continue without stall                        │
  │  ← background thread picks up immutable MemTable → flush│
  └──────────────────────────────────────────────────────────┘
         │ (background flush)
         ▼
  L0 SSTable (new file written sequentially to disk)
```

### WAL Durability Options

```cpp
WriteOptions opts;

// Option 1: sync=true (safest, slowest)
// fdatasync() called after each write. Survives crash immediately.
opts.sync = true;

// Option 2: sync=false, disableWAL=false (default)
// Write goes to OS buffer. Crash before OS flush = lose last few ms of writes.
// Typical: lose < 5 seconds of data on crash.
opts.sync = false;

// Option 3: disableWAL=true (fastest, no durability)
// WAL not written at all. Crash = lose everything not flushed to SSTable.
// Use case: rebuilding data from external source (Kafka consumer offsets rebuilt from Kafka)
opts.disableWAL = true;
```

**Group commit**: when multiple threads write concurrently, RocksDB batches their WAL records and issues a single `fdatasync`. At high write concurrency, this dramatically improves throughput (10 threads × 1 write = 1 fsync instead of 10).

### MemTable Implementations

```cpp
ColumnFamilyOptions cf_opts;

// SkipList (default): O(log n) point lookup + range scan
cf_opts.memtable_factory = std::make_shared<SkipListFactory>();

// HashSkipList: O(1) prefix lookup + O(log n) within prefix. Great for time-series.
// Key format: "prefix:suffix" → hash on prefix → SkipList of suffixes
cf_opts.memtable_factory = NewHashSkipListRepFactory(bucket_count);

// HashLinkedList: O(1) prefix lookup, O(n) within prefix. Use when prefix uniquely identifies few keys.
cf_opts.memtable_factory = NewHashLinkListRepFactory(bucket_count);
```

**Write stall from MemTable**: if flush can't keep up with write rate, immutable MemTables accumulate. When count hits `max_write_buffer_number`, writes stall until a flush completes.

```
Active MemTable  [filling...]
Immutable #1     [waiting for flush]
Immutable #2     [waiting for flush]
Immutable #3     ← hits max_write_buffer_number → WRITE STALL
```

---

## Compaction in RocksDB

### Level Compaction (Default)

```
L0: [sst_a, sst_b, sst_c, sst_d]   ← overlapping key ranges OK here
    When file count ≥ level0_file_num_compaction_trigger:
    compact all L0 files + overlapping L1 files → new L1 files

L1: [sst_L1_1, sst_L1_2, sst_L1_3] ← non-overlapping, total ≤ max_bytes_for_level_base (256MB)
    When total size > max_bytes_for_level_base:
    pick one L1 file → find overlapping L2 files → merge → new L2 files

L2: total ≤ max_bytes_for_level_base × level_multiplier^1  (default: 256MB × 10 = 2.56GB)
L3: total ≤ 25.6GB
...
```

**L0→L1 is the bottleneck**: L0 files all overlap each other (different flush times, same key ranges possible). Compacting L0 → L1 requires reading all L0 files simultaneously. Keep L0 files small and flush fast.

### Compaction Score

RocksDB picks the next level to compact using a score:

```
L0 score = file_count / level0_file_num_compaction_trigger
Ln score = current_bytes / max_bytes_for_level_n
```

Level with highest score > 1.0 wins. Compaction thread picks one file from that level, finds all overlapping files in the next level, merges them.

### Subcompaction

Parallelizes a single compaction job:
```
Compacting L1→L2 over key range ["a", "z"]:
  Subcompaction 1: ["a", "h"] → thread 1
  Subcompaction 2: ["h", "p"] → thread 2
  Subcompaction 3: ["p", "z"] → thread 3
```

```cpp
options.max_subcompactions = 4;  // up to 4 threads for one compaction job
```

Useful when one large compaction is blocking progress (common when a level has huge files).

### Universal Compaction (Size-Tiered Variant)

```cpp
cf_opts.compaction_style = kCompactionStyleUniversal;
```

Merges all sorted runs when the size ratio between adjacent runs exceeds a threshold. Like Cassandra's STCS. Good for:
- Time-series data with monotonically increasing keys (write timestamps)
- Write-heavy workloads tolerating high space amplification
- When write amplification must be minimized

### FIFO Compaction

```cpp
cf_opts.compaction_style = kCompactionStyleFIFO;
FIFOCompactionOptions fifo_opts;
fifo_opts.max_table_files_size = 1ULL * 1024 * 1024 * 1024; // 1GB
cf_opts.compaction_options_fifo = fifo_opts;
```

When total SSTable size exceeds limit: delete oldest SSTable. No merging. Write amplification = 1. Only valid for append-only data with TTL where you don't need reads of old data.

---

## Read Path Deep Dive

```
db.Get("user_123")
      │
      ▼
  Active MemTable → found? return (highest seqno wins)
      │
      ▼
  Immutable MemTables (newest first) → found? return
      │
      ▼
  L0 SSTables (newest first):
    for each L0 file:
      check bloom filter → "no" → skip
      "maybe" → read index block (from block cache or disk)
              → read data block (from block cache or disk)
              → check if key exists
      │
      ▼
  L1+ SSTables:
    for each level Ln:
      binary search on level's file index → at most 1 file contains key
      check bloom filter → "no" → next level
      "maybe" → read index block → read data block → check key
```

### Block Cache

Caches **decompressed** data blocks in memory. **Default (current RocksDB): HyperClockCache** — it replaced LRUCache as the default block cache implementation. HyperClockCache avoids LRU order-tracking overhead and scales better under lock contention, so RocksDB maintainers now recommend it over LRUCache for new deployments, not just high-concurrency ones.

```cpp
// HyperClockCache (current default)
std::shared_ptr<Cache> hcc = HyperClockCacheOptions(
    4ULL * 1024 * 1024 * 1024,  // 4GB
    4096                         // estimated average block size
).MakeSharedCache();
BlockBasedTableOptions table_opts;
table_opts.block_cache = hcc;

// LRUCache: legacy default, still supported
// Prefer HyperClockCache unless you have a specific reason to keep LRU eviction semantics
std::shared_ptr<Cache> cache = NewLRUCache(
    4ULL * 1024 * 1024 * 1024,  // 4GB
    8,                           // shard bits (256 shards → reduces lock contention)
    false,                       // strict capacity limit
    0.5                          // high-priority pool ratio (for index/filter blocks)
);
```

**Block cache vs OS page cache**: RocksDB uses direct I/O (`O_DIRECT`) optionally. Without it, both block cache and OS page cache hold copies. With `use_direct_reads=true`, only block cache holds uncompressed data; OS page cache holds compressed data. Avoids double-buffering. Recommended for large RocksDB instances.

### Row Cache

Caches deserialized row objects (above block cache in the read path):

```cpp
options.row_cache = NewLRUCache(512 * 1024 * 1024);  // 512MB
```

Skip block decoding entirely for hot keys. Higher memory cost per entry (deserialized object > compressed block slice). Worth it for extremely hot key patterns (top-1000 user profiles).

### Bloom Filter Configuration

```cpp
BlockBasedTableOptions table_opts;
// 10 bits/key → ~1% false positive
// 20 bits/key → ~0.01% false positive
table_opts.filter_policy = NewBloomFilterPolicy(10, false);
// false = full filter (one filter per SSTable, not per block) → fewer I/Os
```

**Partitioned index/filter**: for large SSTables (> few GB), index and filter blocks themselves become large. Partition them:

```cpp
table_opts.index_type = BlockBasedTableOptions::kTwoLevelIndexSearch;
table_opts.partition_filters = true;
table_opts.metadata_block_size = 4096;  // each index partition covers this many bytes
```

Top-level index stays in memory; partitions loaded on demand. Reduces index/filter memory by 10-20× for large SSTables.

### Iterator (Range Scan)

A RocksDB iterator is a **merge iterator** over all MemTables + SSTables:

```
MergingIterator
├── MemTableIterator (active MemTable)
├── MemTableIterator (immutable MemTable #1)
├── L0 SSTable Iterator (file 1)
├── L0 SSTable Iterator (file 2)
├── L1 SSTable Iterator (single file covering current range)
└── ...
```

At each step: find the smallest key across all sub-iterators. If multiple sub-iterators have the same key, the one with the highest seqno wins (most recent write).

**Iterator cost**: O(log(total_sstables)) to initialize, O(log(total_sstables)) per `Next()` call (heap-based minimum extraction). Avoid creating iterators in hot loops; reuse via `Seek()`.

---

## Tuning for Production

### Core Write Tuning

```cpp
Options opts;

// MemTable size before flush. Larger = fewer L0 files, better compaction efficiency.
// Larger also means more memory used and more data lost on crash (if WAL disabled).
opts.write_buffer_size = 256 * 1024 * 1024;  // 256MB (default: 64MB)

// Max total MemTable memory (active + immutable).
// active: 1 × write_buffer_size
// immutable: up to (max_write_buffer_number - 1) × write_buffer_size
opts.max_write_buffer_number = 4;  // 4 × 256MB = 1GB total MemTable memory

// Min immutable MemTables before flushing. Default 1 (flush immediately).
// Increasing allows merging small MemTables before flush → fewer L0 files.
opts.min_write_buffer_number_to_merge = 2;
```

### Compaction Tuning

```cpp
// L1 total size. Should be roughly 10× write_buffer_size.
opts.max_bytes_for_level_base = 1024 * 1024 * 1024;  // 1GB

// Size multiplier per level. Default 10.
// L1=1GB, L2=10GB, L3=100GB, L4=1TB
opts.max_bytes_for_level_multiplier = 10;

// Target SSTable size at L1. Files at deeper levels are proportionally larger.
opts.target_file_size_base = 64 * 1024 * 1024;  // 64MB
opts.target_file_size_multiplier = 1;  // same size at all levels (common choice)

// Compaction thread pool size.
opts.max_background_compactions = 4;
opts.max_background_flushes = 2;  // separate pool for flushes (always keep at least 1)

// Subcompaction parallelism.
opts.max_subcompactions = 4;
```

### Write Stall Knobs

```cpp
// L0 file count triggers:
opts.level0_file_num_compaction_trigger = 4;   // start compaction
opts.level0_slowdown_writes_trigger = 20;       // slow writes to 1/8 speed
opts.level0_stop_writes_trigger = 36;           // halt writes until compaction clears L0

// MemTable stall (immutable MemTable count):
// Stall = max_write_buffer_number - 1 immutable MemTables queued
```

Watch these metrics to tune stall thresholds:
- `rocksdb.num-files-at-level0`: target < 4 under normal load
- `rocksdb.compaction-pending`: should trend to 0
- `rocksdb.stall-micros`: cumulative write stall time — alert if increasing

### Compression Configuration

```cpp
// Per-level compression. Deeper levels store more data → use higher compression ratio.
opts.compression_per_level = {
    kNoCompression,   // L0: fastest write, don't compress MemTable flushes
    kNoCompression,   // L1
    kSnappyCompression,  // L2: fast, ~2× ratio
    kSnappyCompression,  // L3
    kZSTD,            // L4: slow but ~3-4× ratio, worth it for bulk cold data
    kZSTD,            // L5
    kZSTD,            // L6
};
opts.zstd_max_train_bytes = 0;  // disable dictionary training unless you have small records
```

### Block Cache Sizing

Rule of thumb: **30% of RAM** allocated to RocksDB block cache.

```cpp
// For a 64GB machine running RocksDB:
// Block cache: ~20GB
// MemTables: ~2GB (4 × 256MB × 2 CFs)
// OS + JVM/process overhead: ~10GB
// Bloom filters: ~10 bits/key × keys_in_memory — calculate separately
```

### Rate Limiter (Compaction I/O Throttling)

Compaction can saturate disk I/O, starving foreground reads.

```cpp
// Allow compaction to use up to 100MB/s disk bandwidth.
opts.rate_limiter = NewGenericRateLimiter(
    100 * 1024 * 1024,  // 100MB/s rate limit
    100 * 1000,          // refill period: 100ms
    10,                  // fairness: 1-10, lower = more equal sharing
    RateLimiter::Mode::kWritesOnly  // only limit compaction writes, not reads
);
```

Set based on your disk's sustainable throughput. For NVMe (~2GB/s): start at 500MB/s and tune. For SATA SSD (~500MB/s): start at 100-200MB/s.

---

## Common Production Issues

### Write Stalls: Diagnosis and Fix

**Symptoms**: p99 write latency spikes, `stall-micros` counter increasing, error logs showing "Stalling writes".

**Diagnosis**:
```bash
# RocksDB stats (dump with LOG file or stats callback)
rocksdb.stall-micros        # cumulative stall time
rocksdb.num-files-at-level0 # should be < level0_slowdown_writes_trigger
rocksdb.num-immutable-mem-table  # should be < max_write_buffer_number - 1
```

**Root causes and fixes**:

| Cause | Symptom | Fix |
|-------|---------|-----|
| L0 file count too high | `num-files-at-level0` > 20 | Increase `max_background_compactions`; increase `level0_slowdown_writes_trigger` temporarily |
| Flush can't keep up | `num-immutable-mem-table` high | Increase `max_background_flushes`; reduce `write_buffer_size` |
| Compaction rate < write rate | L0 growing continuously | Increase compaction threads; reduce write throughput; increase `max_bytes_for_level_base` |
| Rate limiter too aggressive | Stalls despite few L0 files | Increase rate limiter bandwidth |

### Compaction Debt

When writes consistently exceed compaction throughput, debt accumulates:

```
Write rate: 200 MB/s
Compaction throughput: 150 MB/s
Net accumulation: 50 MB/s → 180GB/hour of growing debt
```

Leading indicators: `pending-compaction-bytes` metric. Alert when > 2× `max_bytes_for_level_base`. At 10× you're probably minutes from a write stall.

Fix: reduce write rate (backpressure in application), increase compaction parallelism, increase rate limiter, scale to faster disk.

### Space Amplification Explosion

Dead data accumulating faster than compaction clears it:
```
Live data: 100GB
Actual disk usage: 500GB  → space amplification = 5×
```

**Causes**:
- Snapshot pinning (old snapshot preventing GC of old versions)
- FIFO compaction with too-large limit
- Compaction debt (old data not yet compacted out)
- Universal compaction with many similar-sized tiers

**Fix**: audit active snapshots, ensure snapshots are released after use, monitor `rocksdb.estimate-live-data-size` vs `du -sh /rocksdb/path`.

### Bloom Filter Memory

```
10 bits/key × 1,000,000,000 keys = 10,000,000,000 bits = 1.25GB
```

This is OUTSIDE the block cache. RocksDB loads bloom filters into memory per SSTable (or keeps them in block cache if `cache_index_and_filter_blocks = true`). For multi-billion key datasets, bloom filter memory dominates.

```cpp
// Put index/filter blocks in block cache instead of dedicated memory:
table_opts.cache_index_and_filter_blocks = true;
table_opts.pin_l0_filter_and_index_blocks_in_cache = true;  // pin L0 filters (hot path)
```

Tradeoff: block cache now competes between data blocks and filter blocks. Size cache accordingly.

### Iterator Invalidation

A RocksDB iterator holds a **snapshot** internally. If you call `db->GetSnapshot()`, hold it, call `db->ReleaseSnapshot()`, and then continue using an iterator that was created while the snapshot was held — the iterator may return incorrect results or crash.

Always release iterators before releasing their associated snapshot:
```cpp
auto snap = db->GetSnapshot();
auto iter = db->NewIterator(ReadOptions(), snap);
// ... use iter ...
delete iter;          // release iterator first
db->ReleaseSnapshot(snap);  // then release snapshot
```

---

## RocksDB as Flink State Backend

### Why RocksDB for Flink

`EmbeddedRocksDBStateBackend` stores Flink's keyed state in RocksDB instead of JVM heap.

**Naming history:** the legacy `RocksDBStateBackend` (and `MemoryStateBackend`) classes were deprecated back in Flink 1.13 in favor of `EmbeddedRocksDBStateBackend` (and `HashMapStateBackend`), with checkpoint storage (e.g., `FileSystemCheckpointStorage`) split out as a separate concern. As of Flink 2.0, the old `RocksDBStateBackend`/`MemoryStateBackend` classes have been **removed entirely** (FLINK-36323), not just deprecated — `EmbeddedRocksDBStateBackend` is the only supported RocksDB-backed state backend going forward.

```
HashMapStateBackend:                 EmbeddedRocksDBStateBackend:
  State → JVM HashMap               State → RocksDB (disk + block cache)
  Fast: O(1) hash lookup            Slower: O(log n) + disk I/O
  Limited by JVM heap (< 32GB      Unlimited: state on disk, hot data cached
    for compressed oops)
  Full checkpoint = full state      Incremental checkpoint = changed SSTables only
    snapshot transfer               (typically 5-10× smaller)
  GC pressure for large state      No GC pressure (state not on JVM heap)
```

### Incremental Checkpointing

```
Checkpoint #1: upload all SSTables (e.g., 100GB)
Checkpoint #2: only upload new/modified SSTables (e.g., 2GB delta)
Checkpoint #3: only upload new/modified SSTables (e.g., 1GB delta)
```

Unchanged SSTables are immutable → identical between checkpoints → upload once, reference by filename. Flink tracks which SSTables belong to each checkpoint.

**Gotcha**: if you restore from checkpoint #3, Flink needs all SSTables referenced by #3, which may include SSTables from checkpoint #1. Old checkpoints cannot be deleted until all newer checkpoints that reference their SSTables are also deleted.

### Compaction and Checkpointing Interaction

Compaction happens independently of Flink checkpointing. During compaction:
- Old SSTables (referenced by old checkpoint) cannot be deleted
- New SSTables written
- After checkpoint succeeds, Flink's checkpoint coordinator can release old SSTables

This means compaction is partially blocked until the checkpoint cycle completes. In practice: ensure checkpoint interval (e.g., 60s) is long enough for compaction to make progress.

### Random Access Pattern Problem

Flink processes events, each with a `userId` key → state lookup by `userId`. If events arrive in random user order:

```
event(userId=9999) → RocksDB.Get("9999")  → cache miss (cold key)
event(userId=1234) → RocksDB.Get("1234")  → cache miss (cold key)
event(userId=9999) → RocksDB.Get("9999")  → cache hit (if in block cache)
```

With 10M users and 4KB state per user: 40GB of state. Block cache (4GB) holds 10% → 90% cache miss rate. Each miss = disk I/O → ~100μs SSD read. At 100K events/sec, this is 9GB/s of I/O — exceeds most SSDs.

**Fix**: co-locate related keys, use key hashing to bucket users, or limit state size per key. Monitor `rocksdb.block-cache-miss` vs `rocksdb.block-cache-hit`.

---

## Concrete Example: Kafka Consumer Deduplication Store

**Use case**: Kafka consumer that must deduplicate messages by message ID. Write-heavy (every message = at least one write), point lookup reads (is this ID seen?), TTL-based cleanup (IDs older than 7 days can be deleted).

### Configuration

```java
import org.rocksdb.*;

public class DeduplicationStore {
    
    private static Options buildOptions() {
        Options opts = new Options();
        opts.setCreateIfMissing(true);
        
        // Write path: we write one entry per Kafka message
        // Assume 100K messages/sec → 100K writes/sec
        // Each message ID ~36 bytes (UUID), value ~1 byte (seen marker)
        
        // MemTable: large enough to buffer ~30 seconds of writes before flush
        // 100K writes/sec × 37 bytes = 3.7MB/sec → 256MB covers ~69 seconds
        opts.setWriteBufferSize(256 * 1024 * 1024L);
        opts.setMaxWriteBufferNumber(4);
        opts.setMinWriteBufferNumberToMerge(2);
        
        // Compaction: universal (write-heavy, time-ordered keys work well)
        // Message IDs are UUIDs (random) — use leveled instead for random keys
        opts.setCompactionStyle(CompactionStyle.LEVEL);
        opts.setMaxBytesForLevelBase(1024 * 1024 * 1024L);  // 1GB L1
        opts.setMaxBytesForLevelMultiplier(10.0);
        opts.setMaxBackgroundCompactions(4);
        opts.setMaxBackgroundFlushes(2);
        
        // L0 stall thresholds: generous since we have fast SSD
        opts.setLevel0FileNumCompactionTrigger(4);
        opts.setLevel0SlowdownWritesTrigger(20);
        opts.setLevel0StopWritesTrigger(40);
        
        // Compression: snappy for hot levels, zstd for deep/cold levels
        opts.setCompressionPerLevel(Arrays.asList(
            CompressionType.NO_COMPRESSION,
            CompressionType.NO_COMPRESSION,
            CompressionType.SNAPPY_COMPRESSION,
            CompressionType.SNAPPY_COMPRESSION,
            CompressionType.ZSTD_COMPRESSION,
            CompressionType.ZSTD_COMPRESSION
        ));
        
        // Rate limiter: allow 200MB/s for compaction
        opts.setRateLimiter(new RateLimiter(200 * 1024 * 1024L));
        
        return opts;
    }
    
    private static BlockBasedTableConfig buildTableConfig() {
        BlockBasedTableConfig tableConfig = new BlockBasedTableConfig();
        
        // Block cache: 4GB (30% of 16GB RAM machine)
        tableConfig.setBlockCache(new LRUCache(4L * 1024 * 1024 * 1024));
        
        // Bloom filter: 10 bits/key → ~1% false positive
        // At 500M unique message IDs: 500M × 10 bits = 625MB bloom memory
        // Cache bloom filters in block cache to avoid separate memory allocation
        tableConfig.setFilterPolicy(new BloomFilter(10, false));
        tableConfig.setCacheIndexAndFilterBlocks(true);
        tableConfig.setPinL0FilterAndIndexBlocksInCache(true);
        
        // Block size: 16KB (larger than default 4KB → fewer index entries, better for sequential)
        tableConfig.setBlockSize(16 * 1024);
        
        return tableConfig;
    }
    
    public static RocksDB open(String path) throws RocksDBException {
        Options opts = buildOptions();
        opts.setTableFormatConfig(buildTableConfig());
        return RocksDB.open(opts, path);
    }
    
    // Deduplication check + write
    public boolean isDuplicate(RocksDB db, String messageId) throws RocksDBException {
        byte[] key = messageId.getBytes(StandardCharsets.UTF_8);
        byte[] existing = db.get(key);
        if (existing != null) {
            return true;  // duplicate
        }
        // Mark as seen (value doesn't matter, key presence is the signal)
        WriteOptions writeOpts = new WriteOptions();
        writeOpts.setSync(false);   // async WAL: lose at most ~5s on crash
        writeOpts.setDisableWAL(false); // keep WAL for durability
        db.put(writeOpts, key, new byte[]{1});
        return false;
    }
}
```

### TTL-Based Cleanup

RocksDB has built-in TTL support:

```java
import org.rocksdb.TtlDB;

// Open with TTL: entries older than 7 days auto-deleted during compaction
int ttlSeconds = 7 * 24 * 3600;
TtlDB db = TtlDB.open(opts, path, ttlSeconds, false);
```

**TTL internals**: RocksDB appends a 4-byte timestamp to each value at write time. During compaction, if `now - timestamp > ttl`, the entry is dropped. TTL is approximate — entries survive until a compaction touches them.

**Warning**: TTL and snapshots interact badly. A pinned snapshot prevents TTL GC just like it prevents tombstone GC. Don't hold long-lived snapshots in a TTL database.

---

## Deep Internals

### SkipList MemTable Concurrency

RocksDB's SkipList allows concurrent reads with a single writer using a lock-free design:
- Nodes are allocated from an arena (no malloc per node)
- Insert uses atomic compare-and-swap for pointer updates
- Readers use atomic loads with memory barriers
- Multiple reader threads: no lock needed
- Single writer thread: no lock needed (WriteThread manages serialization)

### Write Thread Serialization

Even though RocksDB is multi-threaded, WAL writes are serialized through `WriteThread`:

```
Thread 1: write A  ─┐
Thread 2: write B  ─┼─→ Leader thread batches A+B+C → single WAL fsync → wake all
Thread 3: write C  ─┘
```

The "leader" thread does the actual WAL write. "Follower" threads wait. This implements group commit without explicit mutex per-write.

### SSTable Format (RocksDB's Block-Based Table)

```
┌────────────────────┐
│ Data Block 0       │  sorted KV pairs, prefix-compressed keys
│ Data Block 1       │  (key[n] stored as (shared_prefix_len, suffix))
│ ...                │
├────────────────────┤
│ Meta Block:        │
│  Filter (bloom)    │  one filter per data block OR one per SSTable (full filter)
│  Index (data)      │  last_key per data block + block offset
│  Compression dict  │  optional zstd dictionary for small-record workloads
├────────────────────┤
│ Meta Index Block   │  maps meta block names to offsets
├────────────────────┤
│ Footer (48 bytes)  │  magic number + meta index offset + index offset
└────────────────────┘
```

**Prefix compression**: keys within a block are often similar (same prefix). Store: `(shared_bytes_with_prev_key, delta_suffix)`. For sequential writes with common prefixes (e.g., `user:0001`, `user:0002`), this achieves 2-5× compression of keys before any block-level compression.

### Compaction Job State Machine

```
WAITING → INSTALLED → RUNNING → FINISHED
```

- WAITING: job queued, waiting for background thread
- INSTALLED: thread picked up job, locking input files
- RUNNING: merging input files, writing output files
- FINISHED: output files registered in VersionSet, input files marked for deletion

VersionSet tracks which files belong to each level at each point in time (MVCC for the file manifest). New readers see the new version; old readers finish against the old version.

---

## Key Gotchas

1. **Write stalls are a cliff, not a slope**: RocksDB writes go full speed until `level0_stop_writes_trigger` is hit, then halt completely. Build backpressure into your write path (rate limiting, write queue) BEFORE hitting the RocksDB stall. The stall itself is recoverable but the resulting latency spike often causes cascading failures upstream.

2. **Snapshot pinning is silent death**: a single snapshot held for hours (e.g., a backup process, a slow iterator) prevents all GC of old versions. `rocksdb.num-snapshots` metric should always be ~0 or match your expected active readers. A growing `rocksdb.estimate-live-data-size` gap vs actual disk usage is the symptom.

3. **L0→L1 compaction is your biggest bottleneck**: L0 files all overlap each other. L0→L1 compaction reads ALL L0 files simultaneously. If L0 grows faster than you can compact to L1, you're in trouble. Tune `max_bytes_for_level_base` to be ~10× `write_buffer_size × min_write_buffer_number_to_merge`.

4. **Bloom filters live outside block cache by default**: with `cache_index_and_filter_blocks=false` (default), bloom filters are loaded into memory separately from the block cache. For 1B keys at 10 bits/key = 1.25GB not accounted for in block cache sizing. Either set `cache_index_and_filter_blocks=true` or account for this in your memory budget.

5. **Random key access pattern kills Flink RocksDB state**: if your Flink keyed state has low temporal locality (each event touches a different key), block cache hit rate will be near 0% for large state. Consider pre-sorting events by key before state access, or use a hash-based sharding strategy to improve locality.

6. **disableWAL=true means crash = full state loss for that CF**: common for rebuilding state from Kafka (you'll replay from the committed Kafka offset), but dangerous if you forget it's set on a CF that you expect to survive crashes.

7. **FIFO compaction drops data**: FIFO's "cleanup" is deleting the oldest SSTable when size exceeds limit. If that SSTable contains keys you still need for reads, those reads will return not-found. Only use FIFO for append-only data where old data expiry is intentional and expected.

8. **Iterator seek on large state is expensive**: `iterator.seek(key)` on a merge iterator must seek all underlying MemTable and SSTable iterators. With 100 SSTables across levels, that's 100 seek operations. For prefix-based access, use `SetPrefix` iterators with a prefix extractor to skip SSTables outside the prefix range.

9. **Compaction and checkpoint interact in Flink**: RocksDB compaction may run during a Flink checkpoint. The compaction produces new SSTables; the checkpoint captures the post-compaction state. But old SSTables (referenced by the previous checkpoint) can't be deleted until the new checkpoint completes. Size your checkpoint storage for 2-3× your normal state size to absorb this overlap.

10. **`max_open_files=-1` vs bounded**: with `-1` (default), RocksDB keeps all SSTable file descriptors open. At scale (10K SSTables), this can exhaust OS file descriptor limits (`ulimit -n`). Set `max_open_files` to a reasonable bound AND raise `ulimit -n` to match.
