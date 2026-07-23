# LSM Trees and LSM-Based Storage Engines

**Date:** 2026-07-23
**Sources:**
- [LSM-based Storage Techniques: A Survey — Luo & Carey (2019)](https://arxiv.org/pdf/1812.07527)
- DDIA Chapter 3 (Storage and Retrieval)

**Related entries:**
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — LSM trees power DynamoDB-like systems optimized for write-heavy workloads with known access patterns
- [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) — Event stores are append-only logs; LSM trees are a natural fit for append-heavy workloads

---

## The Core Idea: Why LSM Trees Exist

### The Problem with Traditional Index Structures (B+ Trees)

A B+ tree uses **in-place updates.** To update a value:
1. Find the page containing the key
2. Modify the page in memory
3. Write the modified page back to its location on disk

This means **random I/O** for every write — the disk head must seek to the exact page location. Random writes on HDDs are ~100× slower than sequential writes. Even on SSDs, random writes cause write amplification at the flash translation layer and reduce device lifespan.

B+ trees are **read-optimized:** finding a key is O(log N) with excellent locality. But the write penalty is severe for write-heavy workloads.

### The LSM Insight: Out-of-Place Updates

Instead of modifying data where it lives, **always write to a new location.** Accumulate writes in memory, then flush them sequentially to disk. This converts random writes into sequential I/O.

The tradeoff: reads become harder (data for one key may exist in multiple places), so you need a merge process to consolidate data and keep reads efficient.

---

## How an LSM Tree Works

### Step 1: The Memory Component (MemTable)

All writes go to an in-memory data structure first:

```
Write("user:1001", {name: "Alice", age: 30})
Write("user:1002", {name: "Bob", age: 25})
Write("user:1001", {name: "Alice", age: 31})  ← update
Delete("user:1002")                            ← tombstone marker
```

The memory component is typically a **skip-list** or **red-black tree** — a sorted data structure that supports fast inserts and range scans.

- **Inserts:** Simply add a new entry
- **Updates:** Add a new entry (the latest version wins during reads)
- **Deletes:** Add a special **tombstone** marker (tells readers "this key was deleted")

The MemTable is **mutable** — it's the only place where writes happen.

**Durability:** Since the MemTable is in memory, a crash would lose data. Solution: a **Write-Ahead Log (WAL)** — every write is first appended to a sequential log on disk. On crash recovery, replay the WAL to reconstruct the MemTable.

### Step 2: Flushing to Disk (Creating SSTables)

When the MemTable reaches a size threshold (e.g., 64MB), it's flushed to disk as an **SSTable** (Sorted String Table).

```
MemTable (full, 64MB)
    │
    ▼ flush (sequential write)
SSTable on disk: [key1, key2, key3, ... keyN]  ← sorted by key
```

After the flush:
- The MemTable is cleared (ready for new writes)
- The WAL for that MemTable is discarded (data is safely on disk now)
- A new empty MemTable begins accepting writes

### Step 3: What's Inside an SSTable

An SSTable is an **immutable, sorted file** on disk:

```
┌──────────────────────────────────────────────────┐
│                   SSTable File                    │
├──────────────────────────────────────────────────┤
│  Data Block 1: [key1:val1, key2:val2, ...]       │
│  Data Block 2: [key3:val3, key4:val4, ...]       │
│  Data Block 3: [key5:val5, key6:val6, ...]       │
│  ...                                             │
├──────────────────────────────────────────────────┤
│  Index Block: [key1→block1, key3→block2,         │
│                key5→block3, ...]                  │
├──────────────────────────────────────────────────┤
│  Bloom Filter (optional)                         │
├──────────────────────────────────────────────────┤
│  Footer (metadata, offsets)                      │
└──────────────────────────────────────────────────┘
```

**Data blocks:** Key-value pairs sorted by key, compressed. Each block is typically 4-64KB.

**Index block:** Sparse index mapping the first key of each data block to its offset. To find key "user:1001": binary search the index to find which data block contains it, then scan within that block.

**Key properties of SSTables:**
- **Immutable:** Once written, never modified. Simplifies concurrency (no locks for reads) and recovery.
- **Sorted:** Enables efficient range scans and merging.
- **Sequential on disk:** Written in one pass, read in large sequential chunks.

### Step 4: Reading (Multi-Component Search)

A read must search through multiple components because the latest value for a key could be anywhere:

```
Read("user:1001"):
  1. Check MemTable (newest data) → found? return it
  2. Check immutable MemTable (being flushed) → found? return it
  3. Check Level 0 SSTables (newest on disk) → found? return it
  4. Check Level 1 SSTable → found? return it
  5. Check Level 2 SSTable → found? return it
  ...
  
  Return the FIRST match (newest version wins)
```

**Problem:** If the key doesn't exist, you search every component — worst case. This is where Bloom filters help.

**Range scans:** Open iterators on all components simultaneously, merge results in sorted order (like a merge step in merge sort). Use a priority queue to pick the smallest key across all iterators, always preferring newer versions.

---

## Compaction: Merging SSTables

As SSTables accumulate, reads get slower (more files to search). **Compaction** (merging) consolidates SSTables:

1. Pick SSTables to merge
2. Read them in parallel (sorted, so this is a merge operation)
3. Write a new SSTable containing only the latest version of each key
4. Discard tombstones for keys that have no newer version
5. Delete the old SSTables

```
Before compaction:
  SSTable A: [a:1, c:3, e:5]
  SSTable B: [b:2, c:7, d:4]    ← c:7 is newer than c:3

After compaction:
  SSTable C: [a:1, b:2, c:7, d:4, e:5]   ← c:3 discarded (obsolete)
```

This is like merge sort — take two sorted files, produce one sorted file. O(N) in the total data size. All sequential I/O.

---

## Compaction Strategies: Leveling vs. Tiering

The central design choice in any LSM-tree implementation. Both organize SSTables into **levels** with a **size ratio T** between adjacent levels.

### Leveling Compaction (LevelDB, RocksDB default)

**Rule:** Each level has **at most one component** (logical run). Level L is T times larger than level L-1.

```
Level 0:  [0-100]                    ← 1 component
Level 1:  [0-100]                    ← 1 component (T× bigger)
Level 2:  [0-100]                    ← 1 component (T× bigger still)
```

**How merging works:** When level L fills up, its component is merged INTO the component at level L+1, producing a new (larger) component at L+1.

The component at level L will be merged multiple times (up to T-1 times) with incoming data from level L-1 before it fills up and is pushed to L+1.

**With partitioning (how LevelDB/RocksDB actually work):** Each level's component is range-partitioned into multiple fixed-size SSTables:

```
Level 0:  [0-100] [0-100]          ← unpartitioned (overlapping key ranges)
Level 1:  [0-30] [34-70] [71-99]   ← partitioned (non-overlapping ranges)
Level 2:  [0-10] [11-19] [20-32] [35-50] [51-70] [72-95]
```

To merge an SSTable from level L into L+1: find all overlapping SSTables at L+1, merge them all, produce new SSTables at L+1.

**Properties:**
- At most 1 component per level → fewer files to search → **better read performance**
- But each entry is merged up to T-1 times per level → **higher write amplification**

### Tiering Compaction (Cassandra default, HBase)

**Rule:** Each level has **up to T components.** When a level fills up (T components), ALL T are merged together into one component at the next level.

```
Level 0:  [0-100] [0-100]           ← up to T components (overlapping!)
Level 1:  [0-100] [0-100] [0-100]   ← up to T components
Level 2:  [0-100]                    ← up to T components
```

**How merging works:** When level L has T components, merge all T together → one new component at level L+1.

```
Level 0 (full, T=4): [A] [B] [C] [D]
                        │
                merge all 4 together
                        │
                        ▼
Level 1: [ABCD]  (one new component)
```

**Properties:**
- Each entry is merged only once per level → **lower write amplification**
- But up to T components per level → more files to search → **worse read performance**
- Multiple overlapping key ranges at each level → range scans more expensive

### Cost Comparison

| Metric | Leveling | Tiering |
|--------|----------|---------|
| Write amplification | O(T · L/B) | O(L/B) |
| Point lookup (with Bloom filter) | O(1) | O(1) |
| Point lookup (zero-result, Bloom filter) | O(L · e^(-M/N)) | O(T · L · e^(-M/N)) |
| Short range query | O(L) | O(T · L) |
| Long range query | O(s/B) | O(T · s/B) |
| Space amplification | O((T+1)/T) | O(T) |

Where: T = size ratio, L = number of levels, B = page size, M = total Bloom filter bits, N = total keys, s = range size.

**The fundamental tradeoff:**
- **Leveling:** optimized for reads and space. Pays more on writes.
- **Tiering:** optimized for writes. Pays more on reads and space.

### Which Systems Use What

| System | Default Strategy |
|--------|----------------|
| LevelDB | Partitioned leveling |
| RocksDB | Partitioned leveling (with tiering option at L0) |
| Cassandra | Tiering (size-tiered) OR leveling (leveled compaction strategy) |
| HBase | Tiering (exploring merge policy) |

---

## Bloom Filters: Avoiding Unnecessary Reads

### The Problem

A point lookup must search every level until it finds the key. If the key doesn't exist, you search ALL levels for nothing. With 7 levels, that's 7 disk I/Os wasted.

### How Bloom Filters Help

A Bloom filter is a **space-efficient probabilistic data structure** that answers: "Is key X in this SSTable?"

- **"No"** → guaranteed correct. Skip this SSTable.
- **"Probably yes"** → might be a false positive. Must actually check.

Each SSTable has its own Bloom filter. Before reading a data block, check the Bloom filter first:

```
Read("user:9999"):
  Level 0 SSTable: Bloom filter says "no" → SKIP (saved a disk I/O!)
  Level 1 SSTable: Bloom filter says "no" → SKIP
  Level 2 SSTable: Bloom filter says "probably yes" → read data block → not found (false positive)
  Level 3 SSTable: Bloom filter says "probably yes" → read data block → FOUND! return value
```

### How a Bloom Filter Works

A bit array of m bits, all initially 0. Use k independent hash functions.

**Insert key X:**
```
h1(X) = 3   → set bit 3 to 1
h2(X) = 7   → set bit 7 to 1
h3(X) = 11  → set bit 11 to 1
```

**Check key Y:**
```
h1(Y) = 3   → bit 3 is 1 ✓
h2(Y) = 5   → bit 5 is 0 ✗ → DEFINITELY NOT IN SET
```

**Check key Z:**
```
h1(Z) = 3   → bit 3 is 1 ✓
h2(Z) = 7   → bit 7 is 1 ✓
h3(Z) = 11  → bit 11 is 1 ✓ → PROBABLY IN SET (could be false positive)
```

### False Positive Rate

Formula: `(1 - e^(-kn/m))^k`

Where k = hash functions, n = keys inserted, m = total bits.

**Practical default:** 10 bits per key → ~1% false positive rate. Bloom filters are small (10 bits/key vs. full key-value pairs of hundreds of bytes) and typically cached entirely in memory.

**Optimal number of hash functions:** k = (m/n) · ln2

### Where Bloom Filters DON'T Help

- **Range queries:** Can't check "are there any keys between A and Z?" Only answers membership for specific keys.
- **Leveling with non-zero result:** If the key exists, you'll find it at the first matching level regardless.

---

## Write Amplification: The Central Problem

### What Is Write Amplification?

The ratio of total bytes written to disk vs. bytes written by the application.

If you insert 1 byte and the LSM tree ultimately writes that byte 30 times across merge operations, write amplification = 30.

### Why It Matters

1. **SSD lifespan:** Each cell can only be written a finite number of times. High write amp kills SSDs faster.
2. **Throughput:** Background merge I/O competes with foreground writes for disk bandwidth.
3. **Tail latency:** Large merges can stall writes (write stalls).

### Write Amplification by Strategy

**Leveling:** A component at each level is merged up to T-1 times (with incoming data from the level above) before it fills up and cascades to the next level. Total write amp: O(T · L/B).

With T=10 and L=5: each byte is written ~50 times.

**Tiering:** Each component is merged only once per level. Total write amp: O(L/B).

With T=10 and L=5: each byte is written ~5 times.

### The RUM Conjecture

The **RUM conjecture** states: no access method can be simultaneously optimal for:
- **R**ead cost
- **U**pdate (write) cost
- **M**emory (space) cost

You must always trade off between them. LSM trees make this tradeoff tunable via the size ratio T and merge policy choice.

---

## Space Amplification

### What Is It?

The ratio of total storage used vs. the actual data size (unique entries only).

If you have 1GB of live data but the database uses 2GB on disk (due to obsolete versions), space amplification = 2.

### By Strategy

**Leveling:** Worst case O((T+1)/T). With T=10: ~1.1×. Very space-efficient because most data lives at the largest level and obsolete entries are cleaned up quickly during frequent merges.

Facebook chose RocksDB (leveling) partly for this reason — with T=10, about 90% of data is at the largest level, at most 10% wasted. Outperforms B+ trees, which waste ~2/3 of space due to page fragmentation.

**Tiering:** Worst case O(T). With T=10: up to 10×. Much worse because you can have T copies of the same key across T components at the same level.

---

## Concurrency Control and Recovery

### Immutability is Key

Disk components (SSTables) are **immutable.** This simplifies concurrency:
- Readers never block writers (readers access immutable SSTables, writers go to MemTable)
- Writers never block readers
- Only the MemTable needs concurrency control (lock-free skip-list or similar)

### Component Metadata

The set of "active components" changes during flushes and merges. Synchronized via:
- Each component maintains a **reference counter**
- Before reading: snapshot the component list, increment ref counts
- After reading: decrement ref counts
- Component deleted only when ref count reaches zero

### Recovery

Writes go to memory first, so durability requires a **WAL:**

```
Write arrives → append to WAL (disk) → insert into MemTable (memory)
```

On crash recovery:
1. Replay WAL to reconstruct the MemTable
2. Load list of disk components (from metadata log or by scanning timestamps)

LevelDB/RocksDB maintain a **MANIFEST** metadata log recording all structural changes (SSTables added/removed). On recovery, replay the MANIFEST.

---

## Real-World Systems

### LevelDB (Google, 2011)

- Pioneered the partitioned leveling merge policy
- Embedded key-value store (library, not a server)
- Simple interface: Put, Get, Scan
- Each SSTable ~2MB, level 1 is 10MB, subsequent levels 10× larger

### RocksDB (Facebook, 2012)

Fork of LevelDB with major improvements:
- **Column families:** Multiple LSM trees sharing one WAL
- **Tiering at Level 0:** Absorbs write bursts without degrading deeper queries
- **Dynamic level sizing:** Adjusts level capacities to bound space amplification at all times (always O((T+1)/T))
- **Rate limiting for merges:** Leaky bucket controls merge disk bandwidth, prevents write stalls
- **Read-modify-write:** Delta records avoid reading before writing
- **Multiple merge policies:** Leveling, tiering, FIFO
- **Compaction filters:** User-defined logic to drop/transform entries during merge (TTL expiration)
- **Merge selection policies:** Cold-first, delete-first, round-robin

Used at Facebook for real-time data processing, graph processing, stream processing, and OLTP (MyRocks).

### Cassandra (Apache)

- Distributed (decentralized, no single point of failure)
- Each data partition powered by an LSM-tree storage engine
- Supports both tiering (size-tiered) and leveling (leveled compaction strategy)
- Date-tiered compaction for time-series data
- Local secondary indexes maintained lazily

### HBase (Apache)

- Distributed (master-replica, modeled after Google Bigtable)
- Data partitioned into regions, each managed by an LSM tree
- Default: exploring merge policy (robust tiering variant picking lowest-cost mergesets)
- Date-tiered merge policy for time-series
- Dynamic region splitting and merging

---

## When to Use LSM-Based Storage

### Good Fit

| Workload | Why LSM works |
|----------|--------------|
| Write-heavy (logs, metrics, IoT) | Sequential writes dominate; write amp tolerable |
| Time-series data | Append-mostly, tiering with date-based compaction |
| Key-value stores with known access patterns | Bloom filters eliminate most unnecessary reads |
| Large datasets where space efficiency matters | Leveling gives ~1.1× space amp vs. B+ tree's ~1.67× |
| SSD-based storage | Sequential writes extend SSD lifespan |

### Poor Fit

| Workload | Why B+ tree is better |
|----------|----------------------|
| Read-heavy with random point lookups | B+ tree: one I/O per lookup guaranteed |
| Heavy range scans over large datasets | B+ tree range scan is one sequential read |
| Latency-sensitive writes needing predictable timing | Compaction causes write stalls |
| Small databases that fit in memory | In-memory B+ tree simpler, no compaction overhead |

---

## Key Takeaways

1. **LSM trees trade read performance for write performance** — the fundamental bargain.
2. **Leveling vs. tiering** is the primary knob: leveling = better reads, tiering = better writes.
3. **Bloom filters** make point lookups nearly free (O(1) with high probability), but don't help range queries.
4. **Write amplification** is the central cost — every byte you write gets rewritten O(L) to O(T·L) times.
5. **Immutability of disk components** simplifies concurrency and recovery dramatically.
6. **Space amplification** varies widely: leveling is space-efficient (~1.1×), tiering can be expensive (~T×).
7. **The RUM conjecture** says you can't have everything — pick your tradeoffs based on workload.
8. **Compaction causes write stalls** — the biggest operational challenge with LSM-based systems.
