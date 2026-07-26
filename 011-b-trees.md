# The Ubiquitous B-Tree

**Date:** 2026-07-26
**Sources:**
- [The Ubiquitous B-Tree — Douglas Comer (Purdue, ACM Computing Surveys 1979)](https://web.archive.org/web/20170809145513id_/http://sites.fas.harvard.edu/~cs165/papers/comer.pdf)

**Related entries:**
- [008-lsm-trees.md](008-lsm-trees.md) — B-trees are the in-place-update counterpart to LSM trees: optimized for reads at the cost of random writes
- [009-bigtable.md](009-bigtable.md) — Bigtable chose LSM over B-trees for write throughput; B-trees remain the standard for read-heavy OLTP (PostgreSQL, MySQL/InnoDB)

---

## Scope and Limitations of This Paper

This is a classic 1979 survey — excellent for understanding B-trees historically, but it doesn't cover modern concerns like SSD optimization, MVCC, write-ahead logging details, or buffer pool management that are critical for today's database internals. For a deep modern understanding, supplement with DDIA Chapter 3 and the CMU Database Systems course materials.

That said, the paper is thorough on the fundamentals: the basic B-tree, insertion/deletion/balancing algorithms, cost analysis, all the variants (B*, B+, prefix B+, virtual B-trees, binary B-trees), concurrency/locking, and IBM's VSAM system as a real-world case study.

---

## The Context: Why Indexes Exist

The paper begins with a physical intuition. Imagine a filing cabinet with employee records. If you want to find "J. Smith," you don't scan every folder — you go to the drawer labeled "S-Z." This is what an index does for a computer: it directs the search to the small part of the file containing the desired record, avoiding a full scan.

The critical insight for understanding B-trees is that secondary storage (disk) is the bottleneck. Every time you visit a node, you incur one disk I/O (seek + read a page). The number of disk accesses is therefore the primary cost metric — not CPU time, not comparisons, but **how many pages you read from disk.** Everything about B-tree design follows from minimizing this number.

---

## From Binary Trees to B-Trees: The Key Insight

A binary search tree stores one key per node and has two children. To search n records requires visiting log₂(n) nodes — each is one disk access. For 1 million records: ~20 disk accesses. At 10ms per access, that's 200ms for one lookup. Unacceptable.

The insight: **pack more keys into each node.** If a node holds d keys, it has d+1 children, and the tree's height drops to log_d(n). A node is sized to match one disk page (typically 4-16KB). If keys are 10 bytes and pointers are 8 bytes, a 4KB page holds roughly 200+ keys. So for 1 million records with d=200: height = log₂₀₀(1,000,000) ≈ 3. Three disk accesses instead of twenty.

```
Binary tree (d=1):              B-tree (d=200):
Height for 1M records: ~20      Height for 1M records: ~3
Disk accesses per find: ~20     Disk accesses per find: ~3
```

This is the fundamental magic of B-trees: by making each node large enough to fill a disk page, you minimize the number of I/Os. A higher branching factor means a shallower tree means fewer disk accesses.

---

## The Basic B-Tree: Definition and Structure

A B-tree of **order d** has these properties:

1. Every node has at most 2d keys and 2d+1 pointers (children)
2. Every node except the root has at least d keys (at least half full)
3. The root has at least 1 key (unless the tree is empty)
4. All leaves are at the same depth (perfectly balanced)
5. Keys within a node are sorted, and the subtree between two keys contains only keys in that range

```
         ┌─────────────────────────┐
         │    [51]                  │  ← root (1 key, 2 children)
         └───────┬────────┬────────┘
                 │        │
    ┌────────────┘        └────────────┐
    │                                  │
┌───┴───────────┐          ┌───────────┴───┐
│ [11 | 30]     │          │ [66 | 78]     │  ← internal nodes
└──┬────┬────┬──┘          └──┬────┬────┬──┘
   │    │    │                 │    │    │
  ...  ...  ...              ...  ...  ...
                    (leaves at the bottom)
```

The half-full guarantee (property 2) ensures that storage utilization is always at least 50%, and in practice averages about 69% (ln 2 ≈ 0.693) for random insertions.

---

## Insertion: How Balance is Maintained

Inserting a key is a two-phase process: first find where it belongs (descend to a leaf), then insert and fix any overflow (propagate upward if necessary).

**Case 1: The leaf has room.** Simply insert the key in sorted position. Done. One write.

**Case 2: The leaf is full (2d keys already).** A **split** occurs:

```
Before insertion of key "72" into a full node (order d=2, max 4 keys):

    Leaf: [66 | 68 | 69 | 71 | 76]  ← 5 keys = too full (max is 2d=4)

Split into two nodes, promote the middle key to parent:

    Parent: [...  71  ...]        ← "71" promoted as separator
             /         \
    [66 | 68 | 69]   [72 | 76]   ← two half-full nodes
```

The middle key (71) is promoted to the parent node as a separator. If the parent is also full, it splits too, and so on. In the worst case, splitting propagates all the way to the root, creating a new root and increasing the tree's height by 1. **This is the only way a B-tree grows taller** — it grows from the bottom up.

This bottom-up growth is what guarantees all leaves remain at the same depth. Unlike binary search trees that can become unbalanced (some paths much longer than others), a B-tree is always perfectly balanced — every leaf is exactly the same distance from the root.

---

## Deletion: Redistribution and Concatenation

Deleting a key is more complex because you must maintain the half-full invariant.

**Case 1: Key is in a leaf, leaf stays at least half full after deletion.** Simply remove the key. Done.

**Case 2: Key is in an internal node.** You can't just remove it — it serves as a separator. Replace it with the **next sequential key** (the leftmost key in the right subtree, which is always in a leaf). Then delete that key from the leaf.

**Case 3: Deletion causes a leaf to have fewer than d keys (underflow).** Two repair strategies:

**Redistribution:** Borrow a key from a sibling node through the parent. The borrowed key goes through the parent (the parent's separator changes). This evens out keys between neighbors and avoids creating or destroying nodes.

**Concatenation (merge):** If the sibling also has exactly d keys (minimum), redistribution isn't possible — the combined count of both nodes plus the parent separator is exactly 2d+1 keys, which fits in a single node. The two nodes merge into one, and the parent loses a separator. If this causes the parent to underflow, the process repeats upward. In the worst case, the root loses its last key and the tree shrinks by one level.

Concatenation is the inverse of splitting: splitting increases keys in a level, concatenation decreases them. Together they keep the tree balanced regardless of the insertion/deletion pattern.

---

## Cost of Operations

The height h of a B-tree of order d indexing n records satisfies:

```
h ≤ log_d((n+1)/2)
```

Since each node visit is one disk I/O, the cost of a **find** is at most h disk accesses. For practical numbers:

| File size (records) | Node size (keys) | Max disk accesses |
|----|----|----|
| 10³ | 10 | 3 |
| 10³ | 50 | 2 |
| 10³ | 100 | 2 |
| 10⁶ | 50 | 4 |
| 10⁶ | 100 | 3 |
| 10⁷ | 50 | 5 |
| 10⁷ | 100 | 4 |
| 10⁷ | 150 | 4 |

A B-tree of order 50 can search **fifty million records** with at most 5 disk accesses. In practice, the root and often the first level or two are cached in memory, reducing actual I/Os to 2-3 for even very large files.

**Insertion and deletion** cost is at most double the find cost (descend to find, then ascend propagating splits/merges). The height still dominates, so all operations are O(log_d(n)) in disk I/Os.

**Sequential processing** is where the basic B-tree struggles. To scan all keys in order, a preorder tree walk visits each node at depth h, which requires stacking all nodes along a root-to-leaf path in memory. Processing a `next` operation may require tracing an entire path from one leaf to another. This motivates the B+-tree variant.

---

## B*-Trees: Delaying Splits

The B*-tree (Knuth's definition) requires each node to be at least **2/3 full** (instead of 1/2 full). It achieves this by a local redistribution scheme: instead of splitting a full node immediately, it tries to redistribute keys with a sibling. Only when two siblings are both full does a split occur — and it splits two nodes into three (each 2/3 full) rather than one into two (each 1/2 full).

This guarantees at least 66% storage utilization (vs. 50% for basic B-trees) with only moderate changes to the maintenance algorithms. The tree is also slightly shorter because nodes are more full on average, meaning slightly fewer disk accesses. The cost is slightly more complex insertion logic.

---

## B+-Trees: The Dominant Variant

The B+-tree is the most important variant and the one used by virtually all modern database systems (MySQL/InnoDB, PostgreSQL, Oracle, SQL Server). The paper explains why it dominates.

**The key separation:** In a B+-tree, ALL keys reside in the leaves. The internal (upper) levels are purely an index — a roadmap to guide searches to the correct leaf. The leaf level forms a **sequence set** — a linked list of all keys in sorted order.

```
              ┌─────────────────┐
              │  [10 | 20 | 30] │         ← Index (roadmap only)
              └──┬────┬────┬──┬┘           Keys here are copies/separators
                 │    │    │   │
    ┌────────────┘    │    │   └──────────────┐
    │                 │    │                   │
┌───┴───┐    ┌───────┴┐  ┌┴────────┐    ┌────┴───┐
│1|3|7|9│───→│11|17|18│─→│20|23|27│───→│30|35|40│  ← Sequence set (leaves)
└───────┘    └────────┘  └────────┘    └────────┘    (linked together)
                                                      ALL actual keys here
```

**Why this separation matters:**

**For find operations:** You always descend to a leaf (the index just guides you). Since all values are found in leaves, the search is consistent — you always read exactly h levels of index plus one leaf.

**For sequential processing (range scans):** Once you reach the leftmost leaf in your range, you simply follow the linked list forward. No more tree walking, no stacking parent nodes, no re-descending. The `next` operation is a single pointer follow along the leaf chain. This is O(1) per record for sequential access.

**For deletion:** Since keys in the index part are just separators (not actual data), a deleted key doesn't need to be removed from the index. As long as the separator still correctly directs searches to the right leaf, it can remain. The key "20" in the index doesn't mean record 20 exists — it means "keys < 20 are to the left, keys ≥ 20 are to the right." This simplifies deletion enormously.

```
After deleting key "20" from the data:

Index still shows:  [...  20  ...]     ← "20" as separator is STILL VALID
                     /        \
Leaves:  [...|17|18] → [23|27|...]     ← "20" removed from leaf,
                                          but separator still guides correctly
```

**For storage efficiency:** The index nodes don't need to store full key-value pairs — just enough of the key to serve as a separator. The leaves store the full records. This means index nodes can have a much higher branching factor (more keys per page) because they're storing less data per entry.

---

## Prefix B+-Trees: Compressing Separators

Since index entries in a B+-tree are just separators (they don't need to match any actual key), they can be shortened. Between the keys "computer" and "electronic," any string that falls between them alphabetically (like "e" or "d" or "elec") works as a separator.

The **prefix B+-tree** uses the shortest distinguishing prefix as the separator. This dramatically reduces the space consumed by index entries, allowing many more separators per index page, increasing the branching factor, and reducing tree height.

```
Leaf keys:    "binary"  "compiler"  "computer"  "electronic"  "program"  "system"

Full separators:  "compiler"  |  "electronic"
Prefix separators:    "c"     |      "e"          ← much shorter!
```

In practice, prefix compression means index nodes can have branching factors of thousands rather than hundreds, further reducing disk I/Os.

---

## Virtual B-Trees: Exploiting Demand Paging

On systems with virtual memory (demand paging), each node of a B-tree can be mapped to exactly one page of virtual address space. The operating system's paging hardware handles bringing nodes into physical memory as needed and evicting unused ones. The B-tree code simply accesses pointers as if everything is in memory.

This gives three advantages:
1. Disk transfers happen at hardware speed (DMA) without software overhead
2. The OS's memory protection isolates different users' trees
3. Frequently accessed nodes (root, first few levels) stay in memory automatically via LRU — the paging system acts as a B-tree node cache for free

Bayer and McCreight suggested that at minimum, the root should always stay resident in memory, since it's accessed by every single search.

---

## Compression Techniques

**Pointer compression:** Instead of storing full absolute addresses for child pointers, store a base address once per node and small offsets for each pointer. This is particularly valuable when pointers are large (64-bit addresses).

```
Uncompressed: [ptr1: 0xAB000100] [ptr2: 0xAB000200] [ptr3: 0xAB000300]
Compressed:   [base: 0xAB000000] [off1: 0x100] [off2: 0x200] [off3: 0x300]
```

Offsets are smaller than full addresses → more keys fit per node → higher branching factor.

**Key compression (front and rear):** Keys sharing common prefixes can be compressed within a node. "computer" and "compiler" share "comp" — store "comp" once and only the suffixes "uter" and "iler" individually. This is the principle behind prefix B+-trees described above.

**Variable-length entries:** Some implementations allow variable-length keys and values within nodes, compacting storage but complicating insertion (no fixed slot positions).

---

## Binary B-Trees

The Binary B-tree (Bayer, 1972a) makes B-trees suitable for a one-level (in-memory) store. It's essentially a B-tree of order 1: each node has 1 or 2 keys and 2 or 3 pointers. Nodes that are only half full use a linked representation. The right pointer in a node may point to either a sibling or a descendant, with an extra bit to indicate which.

Analysis shows that insertion, deletion, and find still take only O(log n) steps. Symmetric Binary B-trees (Bayer, 1973) extend this to allow both left and right sibling links, and contain AVL trees as a subclass.

---

## 2-3 Trees and Theoretical Results

Hopcroft developed the 2-3 tree — each node has 2 or 3 children (equivalent to a B-tree of order 1). The small node size makes 2-3 trees impractical for external storage but useful for in-memory data structures.

Yao (1978) analyzed random 2-3 trees and showed the expected storage utilization is ln 2 ≈ 69%. Extending to B-trees of higher order, the expected utilization remains approximately ln 2 regardless of order — a surprising and fundamental result.

---

## B-Trees in a Multiuser Environment

The paper discusses concurrency — what happens when multiple processes read and write the B-tree simultaneously.

**The problem:** A find operation descends top-down (reading nodes). An insertion or deletion may modify nodes bottom-up (splitting or merging propagates upward). One process might be reading a node while another is splitting it. Without coordination, reads can see inconsistent state.

**Samadi's approach (1976):** During a find, as the search progresses to the next depth, the reader releases its lock on the ancestor. Readers lock at most two nodes at any time — allowing other readers to access the rest of the tree freely.

**Bayer and Schkolnick's reservation protocol:** An update process makes "reservations" (shared locks) on nodes as it descends. When it reaches the leaf and begins modifying, reservations are converted to exclusive locks top-down only for the path that actually needs modification. If the update doesn't propagate upward (no split/merge), higher reservations are simply cancelled. This allows high concurrency because most insertions only modify a leaf — higher levels remain available.

**Top-down splitting (Guibas et al., 1978):** Split nodes that are full on the way DOWN, before they actually overflow. This eliminates the need to ever travel back up the tree during insertion. Only one pair of nodes is locked at a time. The cost: slightly worse space utilization (some splits are premature), but dramatically simpler locking.

**Security:** In a multi-user environment, data encryption may be needed. Bayer and Metzger (1976) show that encipherment has high cost unless implemented in hardware, but changes to B-tree algorithms to accommodate encoded files are minor — the encipherment can be done "on the fly" during transmission.

---

## IBM's VSAM: A Real-World B+-Tree System

The paper closes with a detailed look at IBM's VSAM (Virtual Storage Access Method), a production B+-tree implementation used across IBM mainframe systems.

VSAM demonstrates that a general-purpose file access method can be built entirely on B+-trees. The system uses B+-trees for both user data files and its own internal catalog (the directory of all VSAM files).

**Key design decisions:**
- A leaf node is called a "control interval" — the basic unit of I/O
- Nodes within a subtree ("control area") are placed on the same disk cylinder to minimize seek time
- The sequence set node is replicated on the first track of its cylinder to reduce initial seek latency
- Keys are compressed using both prefix and suffix compression
- The index can be placed on a separate disk device for concurrent access with data
- The system supports virtual B-trees using the paging hardware
- The master catalog (directory of all VSAM files) is itself a VSAM file — the system uses B-trees to catalog B-trees

**Performance enhancements:** Control intervals within a control area are allocated on the same cylinder. The sequence set node is replicated on the first track, reducing seek time. Indexes can be on separate devices for parallel I/O with data.

**Tree-structured file directory:** The master catalog is a B-tree-based VSAM file containing entries for all other VSAM files. Users can define local catalogs to reduce contention. User catalogs are entered into the master catalog, forming a multilevel tree-structured catalog scheme.

---

## The Cost Table: Putting It All Together

| Operation | B-tree | B+-tree |
|-----------|--------|---------|
| Find (random) | log_d(n) | log_d(n) |
| Insert | ~2 × log_d(n) worst case | ~2 × log_d(n) worst case |
| Delete | ~2 × log_d(n) worst case | ~2 × log_d(n) worst case |
| Next (sequential) | Expensive (may traverse multiple levels) | O(1) — follow leaf pointer |
| Range scan | Expensive | Optimal — sequential through leaf chain |
| Space utilization | ≥ 50% (B-tree), ≥ 66% (B*) | ≥ 50%, avg ~69% |

The B+-tree dominates because it retains logarithmic cost for random operations while adding O(1) sequential access — making it ideal for applications needing both random lookups and range scans (which is virtually all databases).

---

## Paper's Conclusions

The B-tree is a balanced, multiway, external file organization that is efficient, versatile, simple, and easily maintained. The B+-tree adds efficient sequential processing while retaining logarithmic random access. B-tree schemes guarantee 50%+ storage utilization, grow and shrink dynamically without ever needing "reorganization," and work in multiuser environments with appropriate locking protocols.

Compression of keys and pointers, careful allocation (and replication) of nodes on secondary storage, and local redistribution of keys during insertion or deletion all improve performance and make B-trees viable in production environments.

---

## Topics Not Covered (Modern Supplements Needed)

For a modern B-tree deep dive, supplement this paper with how modern systems handle:

- **Write-ahead logging and crash recovery with B-trees** — how WAL ensures atomicity of page splits across crashes
- **MVCC (multi-version concurrency control) on B-trees** — how PostgreSQL/InnoDB provide snapshot isolation without blocking readers
- **Buffer pool management and page replacement policies** — how databases manage their own page cache instead of relying on OS paging
- **Copy-on-write B-trees** (used by btrfs, LMDB) — append-only B-tree variants that never modify pages in place
- **Fractal trees / Bε-trees** — write-optimized B-tree variants that buffer writes at internal nodes
- **How B-trees interact with SSDs vs. HDDs** — random reads are cheap on SSD, changing the calculus of node size and write amplification
