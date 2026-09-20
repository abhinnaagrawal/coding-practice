# Redis vs Memcached

## 30-Second Intuition

Both are in-memory key-value stores used as a cache in front of a slower system of record — but they made opposite architectural bets on concurrency. Memcached is multi-threaded from the ground up and treats every value as an opaque byte blob; Redis kept command execution single-threaded (parallelizing only network I/O) in exchange for rich, native data structures (lists, sets, sorted sets, hashes, streams) and optional durability. The one fact that matters operationally in 2026: **the "Redis" you install today is legally and technically forked** — Redis Ltd. relicensed Redis away from permissive open source in March 2024, and the last BSD-licensed version (7.2) was forked by AWS/Google/Oracle and the Linux Foundation as **Valkey**, which is now the *default* in major managed cache services (AWS ElastiCache, Google Memorystore) and is a fully drop-in, protocol-compatible replacement — "Redis" as a technical architecture and "Redis Ltd.'s current licensing terms" are now two separate questions you have to answer separately when choosing what to deploy.

---

## Resource-Layer Map

| Layer | Role | What each is optimizing for |
|---|---|---|
| CPU | Redis: single-threaded command execution (avoids locking complexity entirely) + optional multi-threaded I/O since Redis 6.0. Memcached: fully multi-threaded from the start (one listener thread + many worker threads, lock-based shared hash table) | Redis trades multi-core command throughput for the total absence of race conditions in data-structure mutation — every command is atomic by construction, no locks needed, because only one thread ever touches the data. Memcached trades that simplicity for raw multi-core throughput on simple GET/SET, accepting locking overhead on its shared hash table as the cost |
| Memory | Both are fundamentally memory-resident stores; the entire working set lives in RAM by design in both | Memcached's slab allocator pre-partitions memory into fixed size classes and never returns memory to the OS (fast allocation, but classic size-class fragmentation when values don't fit size classes well). Redis uses more varied per-data-structure encodings (e.g. compact "listpack" encodings for small collections) that adapt to value shape rather than fixed size classes |
| Disk | Redis: optional persistence via RDB snapshots and/or AOF (Append-Only File) log — genuinely optional, can run pure in-memory with zero durability if configured that way. Memcached: none, ever — a restart means total data loss, always, by design | Redis treats disk as an optional durability layer for a cache that some deployments want to survive a restart (or use as a lightweight primary store); Memcached explicitly refuses this scope — it stays a pure, ephemeral cache, deliberately simpler because it never has to reason about durability at all |
| Network | Both are single-node-per-instance in the request path; Redis Cluster and Memcached's client-side consistent hashing both add a network hop for cross-shard reads (client must route to the right shard) | Neither does inter-node coordination the way Kafka replication or etcd consensus does — "clustering" here means client-side or proxy-based sharding, not a replicated consensus protocol; each shard is independently responsible for its own keys |
| GPU | Not applicable | Neither engine's design touches GPU compute at all |

The Redis/Memcached CPU-model split is the actual differentiator in this pair — everything else (data structure richness, persistence) follows from that initial architectural choice, not the other way around.

---

## The Signature Mechanism: Redis's Single-Threaded Event Loop (+ I/O Threads)

Redis's `ae` (Async Events) event loop is the mechanism that makes single-threaded command execution viable at high throughput at all: instead of one OS thread per client connection (which doesn't scale past a few thousand connections due to context-switch and memory overhead per thread), Redis registers every client socket with the OS's native I/O multiplexing API (`epoll` on Linux, `kqueue` on macOS/BSD, `select` as a portable fallback) and a single thread repeatedly asks the kernel "which of these many sockets are ready to read/write right now," processing only the ready ones, one at a time, per loop iteration.

```
Thousands of client connections, one event loop thread:

epoll_wait() → kernel returns: sockets [3, 47, 812, 1024] are ready

for each ready socket:
    read command
    execute command against the in-memory dataset   ← ATOMIC, no lock needed:
                                                         only this one thread
                                                         ever touches data
    write response

loop back to epoll_wait()
```

Because only one thread ever executes a command against the dataset, every Redis command is trivially atomic with zero locking overhead — the entire class of race-condition bugs a lock-based concurrent data structure has to defend against simply doesn't exist here. This is a deliberate trade: Redis gives up using multiple cores *for command execution*, in exchange for eliminating an entire category of concurrency bugs and the performance overhead of lock contention.

**Since Redis 6.0**, this single-threaded guarantee is relaxed *only* for network I/O: **I/O threads** parallelize the read-parse-and-buffer and buffer-and-write parts of handling a connection across multiple OS threads, while command *execution* against the dataset remains strictly single-threaded and serialized. This targets exactly the bottleneck that mattered once single-core command throughput stopped being the limiting factor: on a high-connection-count, high-throughput workload, parsing/writing network bytes for thousands of connections was itself becoming CPU-bound on one core, even though the actual command execution was fast. Recent Redis releases (Redis 8.6-era reporting ~3.5M ops/sec, roughly 5x a pure single-threaded Redis 7 baseline) show this I/O-threading investment paying off specifically for network-bound workloads — command-execution-bound workloads don't get that speedup, since execution itself is still one thread.

Check and tune I/O threading directly:

```
redis-cli CONFIG GET io-threads
1) "io-threads"
2) "1"                       # default: I/O threading disabled, event loop does it all

redis-cli CONFIG SET io-threads 4
OK                            # requires io-threads-do-reads to also be on for read-side parallelism

redis-cli CONFIG GET io-threads-do-reads
1) "io-threads-do-reads"
2) "no"                       # default: only the write path is threaded even with io-threads > 1
```

`INFO server` exposes the fields relevant to confirming the threading model actually in effect:

```
redis-cli INFO server
# Server
redis_version:8.8.0
io_threads_active:1          # 0 = event loop only; 1 = I/O threads are live for this instance
process_id:4213
run_id:9f1c2b7a...
tcp_port:6379
```

`io_threads_active` is the one field that tells you, at a glance, whether the "signature mechanism" above is running in its baseline single-thread form or its I/O-threaded variant on this specific instance.

---

## High-to-Low Walkthrough: `SET key value`

**Redis:** the literal session a client actually runs —

```
redis-cli SET orders:42 shipped
OK

redis-cli GET orders:42
"shipped"

redis-cli OBJECT ENCODING orders:42
"embstr"                      # short strings (<= 44 bytes) use the compact
                               # "embstr" encoding — string header + payload
                               # in one allocation, no separate pointer chase;
                               # longer strings fall back to "raw" encoding
```

Encoding flips once the value shape changes, which is the same "adapt to value shape rather than fixed size classes" behavior described in the Memory row above — a short list gets the same treatment:

```
redis-cli RPUSH orders:42:events queued packed shipped
(integer) 3

redis-cli OBJECT ENCODING orders:42:events
"listpack"                    # small list → compact listpack encoding, not
                               # a full linked-list structure
```

`INFO memory` shows the effect of the SET landing in the in-memory dataset — no disk step unless AOF/RDB is on:

```
redis-cli INFO memory
# Memory
used_memory:1054296
used_memory_human:1.01M
used_memory_rss:9871360
mem_fragmentation_ratio:9.36
maxmemory:0
maxmemory_policy:noeviction
```

```
Client sends SET orders:42 "shipped"
      │
      ▼
I/O thread (if enabled) reads/parses    parallelized across I/O threads if
the command bytes off the socket        Redis 6.0+ I/O threading is on;
                                          otherwise the single event-loop
                                          thread does this directly
      │
      ▼
Single command-execution thread          the ONE thread that ever mutates
applies SET to the in-memory dataset     the dataset processes this command;
                                          no lock needed — nothing else can
                                          be touching the data concurrently
      │
      ▼
(if persistence enabled) AOF append      if AOF is on, the command itself
                                          (or its effect) is appended to the
                                          append-only log before/after
                                          acknowledging, depending on fsync
                                          policy — this is Redis's optional
                                          durability layer, entirely absent
                                          in Memcached
      │
      ▼
Response written back                    via I/O thread or event loop,
                                          symmetric to the read path
```

**Memcached:** the wire protocol is text-based — this is the literal exchange over a `telnet localhost 11211` session (`\r\n` line endings, `0 0 7` = flags, TTL-seconds, byte-length of the value):

```
set orders:42 0 0 7
shipped
STORED

get orders:42
VALUE orders:42 0 7
shipped
END
```

`stats slabs` shows which size class actually absorbed that value and how full its slab page is:

```
memcached-tool localhost:11211 stats slabs
  #  Item_Size  Max_age   Pages   Count   Full?
  1      96B     3     1     123    no
  2     120B     5     1      42    no
  3     152B     2     1      11    no      ← "orders:42" + "shipped" (14 bytes
                                               of key+value) lands here, in the
                                               smallest class that fits it
```

`stats` (plain, no `-tool`) gives the running counters — `get_hits`/`get_misses`/`evictions` are exactly the numbers referenced in the Deep Internals LRU discussion below:

```
stats
STAT curr_connections 12
STAT cmd_get 918273
STAT cmd_set 40211
STAT get_hits 902100
STAT get_misses 16173
STAT evictions 3821
STAT bytes 10485760
END
```

```
Client sends SET orders:42 "shipped"
      │
      ▼
Listener thread accepts connection,      one dedicated listener thread hands
hands off to a worker thread              off new connections to a pool of
                                           worker threads round-robin
      │
      ▼
Worker thread: find/allocate a slab       Memcached's slab allocator picks
chunk for this value's size class         the smallest pre-allocated size
                                           class that fits "orders:42" +
                                           "shipped" — if no free chunk of
                                           that class exists, allocates a
                                           new slab page from the OS (never
                                           returned once allocated)
      │
      ▼
Worker thread: lock + update the          Memcached's shared hash table is
shared hash table                          protected by locking (this worker
                                            thread and others may be
                                            concurrently mutating different
                                            keys) — this is the direct cost
                                            of the multi-threaded design:
                                            lock acquisition/contention on
                                            the shared structure
      │
      ▼
Response written back                     no persistence step exists —
                                           there is nothing further to do;
                                           the value now exists only in RAM
```

The contrast is direct: Redis pays zero locking cost but only ever uses one core for the actual mutation; Memcached pays a locking cost on every mutation but can genuinely parallelize across cores for that mutation itself.

---

## Deep Internals

### Redis: Rich Data Structures, Not Just Strings

Where Memcached only ever stores opaque byte blobs (you get GET/SET/DELETE and nothing structural), Redis natively implements lists, sets, sorted sets (skip-list-backed, giving O(log n) ranked operations), hashes, streams (an append-only log structure usable for lightweight pub/sub-with-history), and HyperLogLog (probabilistic cardinality estimation in fixed ~12KB regardless of set size). Small instances of these structures use compact "listpack" encodings (a serialized, cache-friendly byte layout) rather than the full generic structure, automatically converting to the full structure past a configurable size threshold — an optimization aimed squarely at the common case of many small collections (a user's session hash, a short recent-activity list) rather than a few enormous ones.

Each structure, with its actual commands:

```
redis-cli LPUSH mylist a b c
(integer) 3
redis-cli LRANGE mylist 0 -1
1) "c"
2) "b"
3) "a"
redis-cli OBJECT ENCODING mylist
"listpack"                    # small list; converts to "quicklist" past
                               # list-max-listpack-size (default 128 entries)

redis-cli ZADD leaderboard 100 user1 250 user2 90 user3
(integer) 3
redis-cli ZREVRANGE leaderboard 0 -1 WITHSCORES
1) "user2"
2) "250"
3) "user1"
4) "100"
5) "user3"
6) "90"
redis-cli OBJECT ENCODING leaderboard
"listpack"                    # converts to "skiplist" past
                               # zset-max-listpack-entries (default 128)

redis-cli HSET session:abc user_id 42 role admin
(integer) 2
redis-cli HGETALL session:abc
1) "user_id"
2) "42"
3) "role"
4) "admin"
redis-cli OBJECT ENCODING session:abc
"listpack"                    # converts to "hashtable" past
                               # hash-max-listpack-entries (default 128)

redis-cli XADD events:orders '*' type shipped order_id 42
"1757766123456-0"              # auto-generated stream ID: <ms-time>-<seq>
redis-cli XLEN events:orders
(integer) 1

redis-cli PFADD visitors:2026-09-13 user1 user2 user3
(integer) 1
redis-cli PFCOUNT visitors:2026-09-13
(integer) 3                    # fixed ~12KB backing structure regardless
                                # of how many millions of elements are added
```

### Redis Persistence: RDB vs AOF

- **RDB (snapshot)**: periodic point-in-time binary dumps of the whole dataset — fast to load on restart, but any writes since the last snapshot are lost on a crash.
- **AOF (Append-Only File)**: every write command is logged; replaying the log reconstructs full state. Configurable fsync policy (`always`, `everysec`, `no`) trades durability against write latency — `always` fsyncs every write (safest, slowest), `everysec` batches (default, bounded data loss window), `no` leaves fsync entirely to the OS (fastest, least durable).
- Both can be enabled together; Redis can also run with neither, becoming a pure volatile cache functionally closer to Memcached's durability model, while retaining Redis's data structures.

Triggering and inspecting each mechanism directly:

```
redis-cli BGSAVE
Background saving started        # forks a child process; the parent (event
                                  # loop) keeps serving traffic while the
                                  # child writes dump.rdb via copy-on-write

redis-cli LASTSAVE
(integer) 1757766000             # unix timestamp of the last successful RDB save

redis-cli INFO persistence
# Persistence
rdb_changes_since_last_save:128
rdb_bgsave_in_progress:0
rdb_last_bgsave_status:ok
aof_enabled:0
aof_rewrite_in_progress:0
```

```
redis-cli CONFIG SET appendonly yes
OK                                # turns AOF on at runtime; triggers an
                                  # initial AOF rewrite in the background

redis-cli CONFIG SET appendfsync everysec
OK                                # the default: fsync at most once per
                                  # second, bounding crash data loss to ~1s

redis-cli CONFIG SET appendfsync always
OK                                # fsync on every write — safest, adds
                                  # per-write latency

redis-cli CONFIG GET appendfsync
1) "appendfsync"
2) "everysec"
```

### Memcached: Slab Allocation and LRU

Memcached pre-partitions memory into a fixed set of size classes (a geometric progression, e.g. 96B, 120B, 152B... up to 1MB by default), and every stored value is placed into the smallest class that fits it. This makes allocation itself fast (grab a free chunk of the right class, no general-purpose allocator call) at the cost of internal fragmentation — a 100-byte value stored in a 120-byte-class slot wastes the remaining bytes, and because Memcached never returns slab memory to the OS once allocated, a workload whose value-size distribution shifts over time can end up with memory "stuck" in size classes that no longer match the actual data, even though total memory usage looks unchanged. Eviction uses per-slab-class LRU (least-recently-used) — an oversized influx of one size class evicts within that class, not globally across all stored data.

`stats slabs` is the literal command that shows per-class chunk counts and total pages committed to each class — this is where "memory stuck in a size class" becomes visible:

```
stats slabs
STAT 1:chunk_size 96
STAT 1:chunks_per_page 10922
STAT 1:total_pages 1
STAT 1:total_chunks 10922
STAT 1:used_chunks 8213
STAT 1:free_chunks 2709
STAT 3:chunk_size 152
STAT 3:chunks_per_page 6898
STAT 3:total_pages 42
STAT 3:total_chunks 289716
STAT 3:used_chunks 288900
STAT 3:free_chunks 816
STAT active_slabs 12
STAT total_malloced 536870912
END
```

Class 3 above (`used_chunks` ≈ `total_chunks`, 42 pages committed and never released) is exactly the "memory stuck in a size class" failure mode — pages were allocated when values of that size were common and stay reserved for that class even if the workload's value-size distribution later shifts away from it.

Per-class eviction and hit-rate counters, from plain `stats`:

```
stats
STAT evictions 3821
STAT expired_unfetched 112
STAT evicted_unfetched 40
STAT get_hits 902100
STAT get_misses 16173
STAT get_expired 890
END
```

A rising `evictions` count alongside a low `get_hits`/`get_misses` ratio for a specific key pattern is the operational signal that a slab class is both full and being pressured — the fix (per the Gotchas section below) is usually a restart, not a config tweak.

### Clustering: Client-Side Sharding, Not Consensus

Both Redis Cluster and traditional Memcached deployments shard data across nodes primarily via **client-side or proxy-side hashing** (consistent hashing in Memcached's classic deployment model; hash-slot assignment in Redis Cluster) — this is fundamentally different from a consensus-replicated system like etcd (`coordination/etcd.md`): there's no quorum write, no linearizability guarantee across the cluster, and a node holding a given key's data is simply "the one your hash function points you to," not "the current elected leader for that data after a consensus round." Redis Cluster does add asynchronous primary-replica replication *within* a shard for availability, but cross-shard operations aren't transactionally consistent the way a consensus store's operations are.

Redis Cluster's hash-slot assignment (16384 fixed slots, `CRC16(key) mod 16384`) is directly inspectable:

```
redis-cli CLUSTER KEYSLOT orders:42
(integer) 2865                   # this key always maps to slot 2865,
                                  # regardless of cluster topology

redis-cli CLUSTER SHARDS
1)  1) "slots"
    2) 1) (integer) 0
       2) (integer) 5460
    3) "nodes"
    4) 1)  1) "id"
            2) "07c37dfeb235213a872192d90877d0cd55635b91"
            3) "port"
            4) (integer) 6379
            5) "role"
            6) "master"

redis-cli CLUSTER COUNTKEYSINSLOT 2865
(integer) 4                      # how many keys currently live in the slot
                                  # that "orders:42" hashes to — useful for
                                  # spotting a hot/skewed slot
```

`CLUSTER SHARDS` is what a client (or a proxy like a cluster-aware connection pool) uses to build its slot-to-node routing table — this is the literal mechanism behind "a node holding a given key is the one your hash function points you to."

---

## Comparative: Cache vs Coordination Store

Worth stating explicitly since the two are sometimes reached for interchangeably: Redis/Memcached are **caches** — optimized for high-throughput, low-latency reads/writes of a working set, with weak-to-optional durability and no built-in distributed-consensus guarantees. Systems like etcd (`coordination/etcd.md`) are **coordination stores** — optimized for a small amount of critical configuration/state that must be linearizable and durably replicated via consensus (Raft), explicitly accepting much lower throughput and higher write latency in exchange for that guarantee. Using Redis as a source of truth for distributed locking or leader election (a common but risky pattern) is reaching for a cache to do a coordination store's job — it can work under specific conditions (e.g. the Redlock algorithm), but it isn't what Redis's core design targets, and etcd/ZooKeeper-family systems exist specifically because that's a different, harder problem.

---

## Key Gotchas

- **"Redis" now means at least two different license situations**: Redis 8.x from Redis Ltd. ships tri-licensed (choose AGPLv3, or the source-available SSPLv1/RSALv2) — AGPLv3 is OSI-approved open source, but its copyleft terms are far more restrictive for SaaS/managed-service use cases than the old BSD license was. **Valkey** (the Linux Foundation fork of the last BSD-licensed Redis, 7.2) is a fully protocol-compatible, genuinely permissively-licensed alternative, and is now the *default* in AWS ElastiCache and Google Memorystore for new instances — confirm which one a "Redis" dependency in your stack actually resolves to before assuming license terms.
- **Single-threaded command execution means one slow command blocks everything**: a large `KEYS *`, an expensive Lua script, or a big `SORT`/aggregation on a huge collection stalls every other client's commands on that instance for its full duration — there's no other command-execution thread to pick up slack. Use `SCAN` instead of `KEYS`, and watch for O(n) commands on large collections in production traffic paths.
- **AOF fsync policy is a real latency/durability tradeoff, not a default to ignore**: `appendfsync everysec` (the common default) means up to ~1 second of writes can be lost on a crash — acceptable for a cache, potentially not for data you're treating as durable via Redis's persistence.
- **Memcached's slab classes can strand memory permanently**: a workload whose typical value size shifts over the life of the process can leave large amounts of memory allocated to slab classes that no longer match incoming data — this shows up as "memory usage is high but hit rate for the current workload is low," and the fix is usually a controlled restart, not a config tweak, since Memcached never returns slab memory to the OS.
- **Neither is a replicated-consensus system, despite "cluster" in the name**: Redis Cluster's hash-slot sharding and Memcached's consistent-hashing client libraries both distribute *load*, not provide the linearizable, quorum-committed guarantees a coordination store gives — don't reach for either as a distributed lock/leader-election primitive without understanding exactly what failure modes (split-brain, stale reads during a primary failover) you're accepting.
- **Memcached's total lack of persistence is a feature, not a missing one to work around**: some teams try to bolt durability onto Memcached via external replication tooling; if you find yourself doing that, the workload has outgrown "pure cache" and Redis (with RDB/AOF) or a real database is very likely the correct tool, not a patched-together durable Memcached.

---

*Grounded against Redis's official license/release documentation, the Valkey project (Linux Foundation), and 2026 community benchmarks/writeups as of September 2026. Current Redis line: 8.8 (May 2026), tri-licensed (AGPLv3 / SSPLv1 / RSALv2). Valkey (BSD-3-Clause, forked from Redis 7.2) reached Valkey 9 GA October 2025 and is now the default in AWS ElastiCache and Google Memorystore for new instances. Memcached remains a stable, widely-deployed pure-cache engine with no comparable licensing controversy. Re-verify which fork/license a specific deployment is actually running before making licensing-sensitive decisions.*
