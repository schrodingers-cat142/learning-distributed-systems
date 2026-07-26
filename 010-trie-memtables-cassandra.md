# Trie Memtables in Cassandra

**Date:** 2026-07-26
**Sources:**
- [Trie Memtables in Cassandra — Branimir Lambov (DataStax, PVLDB 2022)](https://www.vldb.org/pvldb/vol15/p3359-lambov.pdf)

**Related entries:**
- [008-lsm-trees.md](008-lsm-trees.md) — This paper improves the memory component (memtable) of the LSM stack described in Entry #8
- [009-bigtable.md](009-bigtable.md) — Bigtable established the memtable → SSTable → compaction pattern; this paper optimizes the memtable layer for JVM-based systems

---

## What This Paper Is About

A new memtable implementation for Apache Cassandra that replaces the traditional comparison-based skip-list with a **trie (prefix tree)** built on byte-comparable key representations. In production in DataStax Enterprise 6.8, being integrated into Apache Cassandra as CEP-19.

The paper solves specific performance problems with Cassandra's existing memtable that stem from Java/JVM constraints and comparison-based data structures.

---

## The Problem with Comparison-Based Memtables

Cassandra's existing memtable uses a concurrent skip-list, which has inherent inefficiencies:

**Key comparisons are expensive.** Cassandra keys are composite (partition key + clustering columns). Comparing two keys requires multi-morphic virtual calls for each component's comparison logic. Binary search branching is unpredictable. Keys must be fully present in comparable form, and cache space for frequently-accessed keys is large. Worst-case lookup: O(k log n) for n entries and key length k.

**The heap size problem.** Cassandra runs on the JVM. The memtable uses off-heap memory for data, but indexing structures (skip-list nodes and pointers) live on-heap. On-heap size is limited (usually under 32GB) and large on-heap footprints worsen garbage collection, causing high-percentile latency spikes.

**Garbage collection pressure.** Memtables have medium-term lifetimes — promoted to old GC generations, then freed during expensive full GC cycles. The larger the heap fraction the memtable takes, the worse GC pauses become. This is one of the less-obvious causes of Cassandra's tail latency problems.

---

## The Solution: Byte-Comparable Keys + Tries

### Part 1: Byte-Comparable Representations

Transform any Cassandra key into a byte sequence where lexicographic byte comparison gives the same ordering as the original typed comparison. This eliminates type-aware comparison logic at lookup time.

Translation rules by type:
- Unsigned fixed-length integers → big-endian byte sequence
- Signed fixed-length integers → invert the sign bit, then big-endian
- IEEE floating point → invert sign bit; if negative, invert all other bits
- Variable-length blobs → encode all 00s as 00 01, terminate with 00 00 (ensures prefix-freedom)

Two required properties of the translation ψ_T for type T:
1. **Comparison equivalence:** Byte-comparing ψ_T(x) vs ψ_T(y) gives the same result as comparing x vs y using T's comparator
2. **Prefix-freedom:** No key's byte representation is a prefix of another's (prevents ambiguity in composite keys)

Composite keys: concatenate the byte translations of each component. Prefix-freedom ensures the boundary between components is unambiguous.

### Part 2: Trie (Prefix Tree) Instead of Skip-List

Once keys are byte-comparable sequences, a trie exploits their structure naturally:

```
Keys: "trie", "this", "the", "types", "without", "with", "allow"

              root
            / | \ \
           a  n  o  t    w
          /       |   \     \
         l        ...  h  y   i
        ...           |  |    \
       (allow)       e  p     t
                    / \  |     \
                  (the) i    h → (with) → o → u → t → (without)
                        |
                      s → (this)
```

Keys sharing a common prefix share the same path — automatic compression. Lookup is O(k) where k is key length in bytes, independent of entry count. No comparisons — just follow transitions byte by byte.

---

## The Trie Memtable Design

### Off-Heap Storage in Large Byte Buffers

Instead of Java objects for each trie node (millions of objects for GC to track), the trie structure is packed into large pre-allocated byte buffers. Custom memory management within these buffers means the GC sees only a handful of large buffer objects — not millions of small nodes. This is the key to eliminating GC pressure.

### 32-Byte Blocks

Data is packed in 32-byte blocks matching cache line fetches. Each block is the unit of allocation and release. No 1:1 correspondence between nodes and blocks — nodes are packed efficiently.

### Five Node Types

| Type | Children | Blocks | Typical Location | Lookup |
|------|----------|--------|------------------|--------|
| **Leaf** | 0 | 0 | Leaf of trie | — |
| **Chain** | 1 (sequences of 1-28) | 1/28 to 1 | Before leaf (after unique prefix) | Equality check |
| **Sparse** | 2-6 | 1 | Middle of trie | Linear scan (small array) |
| **Split** | >6 | 3-37 | Near root (many children) | Bit manipulation + pointer hops |
| **Prefix** | — | 0-1 | Intermediate with content | — |

**Split nodes** use a 3-level structure indexed by bits of the transition byte (2+3+3 bits). Fast lookup via bit manipulation.

**Sparse nodes** don't store transitions in order — allows concurrent modification by appending without reordering existing entries.

**Chain nodes** are the most common optimization: after a unique prefix is determined, the remaining bytes form a single-child sequence stored compactly (up to 28 bytes per block).

### Concurrency: Single-Writer, Multiple-Reader

Writes are serialized within a shard. Readers proceed concurrently without locks.

Adding a child to a sparse node: append the new target and transition byte with a volatile write — immediately visible to all readers.

When a node must be restructured (chain → sparse, sparse → split): old node is left in place for existing readers while parent pointer is atomically updated to new node.

### Sharding

The memtable is split into multiple independent tries (shards), default one per CPU thread. Because Cassandra hashes partition keys evenly across token space, each shard receives roughly equal write load. This eliminates single-writer bottleneck while maintaining the simple single-writer-per-shard concurrency model.

---

## Traversal and Merging

The critical operation for an LSM memtable is iterating in sorted order (range queries, flushes, merges).

The trie uses a **cursor** that walks nodes in lexicographic order:
1. Take first child transition (descend)
2. If no children: ascend to nearest parent with remaining children, take next transition

Two cursors advanced in parallel can determine merge order — the descend-depth and incoming character at each step are sufficient to compare positions without reconstructing full keys.

This enables efficient union/intersection of multiple tries — essential for the merged view across shards used for range queries and flushes.

---

## Performance Results

### Microbenchmarks (10M partitions, 100-byte rows)

| Operation | Trie vs Skip-list |
|-----------|------------------|
| Random lookup (key exists) | ~2× faster |
| Random lookup (key doesn't exist) | ~2× faster |
| Write (fill memtable) | ~2.5× faster |
| Memory (heap footprint) | 20-30% smaller |

### Long-Running Throughput (1TB write, 90:10 write:read)

- **Apache Cassandra branch:** Trie averages ~145K ops/sec vs. skip-list ~85K ops/sec (70% improvement)
- **DataStax Converged Cassandra** (with improved compaction): Trie sustains >2× skip-list throughput

### Latency at Fixed Throughput (110K ops/sec)

- p95 latency: Trie consistently lower
- Old-gen GC time: **696s (skip-list) → 0s (trie)** over test duration
- Young-gen GC time: 10,144s → 4,290s

### SSTable Output

Trie memtable produces **30% larger L0 SSTables** (flushes less frequently due to compactness) — fewer SSTables means fewer compactions needed downstream.

### Memtable switch count

Over the test: 1,510 switches (trie) vs. 1,982 (skip-list). Fewer flushes = less compaction pressure.

---

## Alternatives Considered

**Sharded skip-lists:** Better concurrency than single skip-list but still suffer from GC pressure and comparison overhead. Under skewed workloads (hot partitions), sharding helps less than the trie.

**B-Trees:** Cassandra already uses B-Trees for the partition map. Could be extended to cover all memtable levels, but concurrent modification is complex, and detaching keys from nodes adds pointer-chasing overhead.

**Adaptive Radix Trees (ART):** Shares concepts (typed nodes, multi-byte paths). But authors couldn't find an ART implementation supporting concurrent reads, and ART uses variable-size blocks that complicate memory management.

**Succinct tries (SuRF):** Extremely compact, theoretically optimal. But not suitable for mutable structures — useful for on-disk SSTables as a Bloom filter alternative (complementary to this work).

---

## How This Connects to LSM Trees

This paper improves the **memory component** of the LSM stack without changing the fundamental design (levels, compaction, SSTables). The cascading benefits:

- Faster writes → higher sustained throughput
- Less GC → lower tail latency (the operational challenge identified in Entry #8)
- Larger effective memtable → fewer flushes → fewer L0 SSTables → less compaction
- Efficient in-order iteration → faster memtable flushes and merges

---

## Future Work

The paper identifies several directions:
- Extend trie to lower levels of the data hierarchy (currently maps to partitions; could cover rows and cells within partitions)
- Apply the trie paradigm to immutable on-disk SSTables (different locality and caching characteristics, but benefits of prefix sharing and faster lookup apply)
- Improvements to traversal (descending to specific children)
- Write atomicity support (consistent views of partition during writes)
- Recycling and reuse of blocks within the byte buffers
