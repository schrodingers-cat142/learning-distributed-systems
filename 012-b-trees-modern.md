# B-Trees: Modern Deep Dive

**Date:** 2026-07-26
**Sources:**
- [Torn Writes — transactional.blog (2025)](https://transactional.blog/blog/2025-torn-writes)
- DDIA Chapter 3 (Storage and Retrieval)

**Related entries:**
- [011-b-trees.md](011-b-trees.md) — The 1979 Comer survey covering B-tree fundamentals; this entry covers modern concerns that paper doesn't address
- [008-lsm-trees.md](008-lsm-trees.md) — LSM trees are the write-optimized counterpart to B-trees; the tradeoffs between them are covered here

---

## Overview

The 1979 Comer paper covers B-tree fundamentals but doesn't address six critical modern topics: WAL and crash recovery, MVCC, buffer pool management, copy-on-write B-trees, fractal/Bε-trees, and SSD behaviour. This entry covers all six, plus how the torn writes blog connects them.

---

## Part 1: Write-Ahead Logging and Crash Recovery

### The Core Problem

A B-tree page split is not a single atomic disk operation. Inserting a key into a full leaf requires:
1. Write the new left leaf (half the old keys)
2. Write the new right leaf (other half + new key)
3. Write the parent (promoted separator key + new child pointer)

Three separate disk writes. If power fails after write 1 but before writes 2 and 3, the tree is structurally inconsistent. The parent still points to the old (now missing) page. This is the **torn write** problem compounded with the **logical multi-step atomicity** problem.

### The WAL Guarantee

A Write-Ahead Log (WAL) enforces one invariant:

> Before any B-tree page is modified on disk, a log record describing the change must be durably written to disk first.

This gives crash recovery: if the database crashes mid-operation, the B-tree pages on disk may be incomplete, but the WAL is complete up to the last fsync. On restart, replay the WAL from the last checkpoint to bring B-tree pages into a consistent state.

The WAL must be **sequential-append-only** — this converts 3 random writes (scattered B-tree pages) into 1 sequential append plus 3 random page writes. You've added work, but you've guaranteed atomicity and made writes faster on spinning disk.

### The Torn Write Problem (from the blog)

The WAL solves logical atomicity but not physical torn writes. A 4KB database page spans multiple 512-byte disk sectors. If power fails mid-write, some sectors hold new data and others hold old data — the page is internally inconsistent byte-by-byte.

Databases use different strategies to protect against this, each with different write amplification and latency tradeoffs:

| Strategy | Write Latency | Write Amp | How it Works |
|---|---|---|---|
| Detection only (SQL Server) | 1× | 1× | Bit-per-sector counter; detects but can't self-repair |
| Log full pages (SQLite WAL) | 1× | 2× | Entire page goes in WAL before any B-tree write |
| Log page on first write (PostgreSQL `full_page_writes`) | ~1× | ~1× | Full page in WAL once per checkpoint; deltas after |
| Double-write buffer (InnoDB) | 2× | 2× | Write page to scratch area, then to final location |
| Copy-on-write (LMDB) | 2× | O(height) | Never overwrite; write new page, update parent upward |
| Atomic block writes (`RWF_ATOMIC`, Linux 6.11+) | 1× | 1× | Hardware guarantee; makes all other strategies optional |

**PostgreSQL's full-page writes** in depth: PostgreSQL uses 8KB pages. Its WAL normally contains logical records (e.g., "insert key K at position P in page X"). The problem: if a page is torn, applying logical deltas on top of torn garbage produces more garbage. Solution: on the **first modification of a page after a checkpoint**, write the entire 8KB page image (compressed) into the WAL. Subsequent modifications to that page within the same checkpoint cycle log only deltas. On crash recovery, PostgreSQL unconditionally replaces the page with the full-page image (no matter how torn), then applies all subsequent deltas. Controlled by `full_page_writes = on`.

With `RWF_ATOMIC` hardware support (Linux 6.11+, XFS/ext4 6.13+, multi-block in 6.15+), systems like AlloyDB Omni can disable `full_page_writes` entirely — hardware guarantees 8KB atomic writes, eliminating the 2× WAL amplification.

**InnoDB's double-write buffer:** Before writing a batch of dirty B-tree pages, write them all to a sequential scratch area with one fsync. Then write them to their final locations. If a crash occurs during the second write, recovery finds the clean copy in the double-write buffer. WAL handles logical atomicity; double-write buffer handles torn writes. These are independent mechanisms. MySQL 8.0.23 reduced the historical mutex contention.

### DDIA Chapter 3 Connection

DDIA emphasizes that WAL transforms random writes into sequential appends — an order of magnitude faster on spinning disk (no seeking). DDIA also distinguishes redo logs (replay writes that didn't reach the B-tree) from undo logs (roll back incomplete transactions). B-tree databases use both. LSM-trees don't need undo logs — uncommitted data is simply discarded from the MemTable.

---

## Part 2: MVCC on B-Trees

### Why Locking Alone Is Insufficient

Naive concurrency: readers and writers lock pages. Simple, correct, terrible performance — readers block writers and vice versa.

MVCC solution: keep multiple versions of each row. Readers see a consistent snapshot of an older version while writers create new versions. **Readers never block writers; writers never block readers.**

### PostgreSQL's Approach: Versions in the Heap

PostgreSQL stores multiple row versions directly inside the B-tree leaf pages. Every row has system columns:

```
(xmin, xmax, data...)
xmin: transaction ID that created this row version
xmax: transaction ID that deleted/updated this version (0 if still live)
```

A row version is visible to transaction T if:
- `xmin` is committed AND `xmin` < T's snapshot timestamp
- `xmax` is zero, uncommitted, or > T's snapshot timestamp

When T2 updates a row, it creates a **new version** and marks `xmax` on the old:

```
Before T2 updates (sets age 30 → 31):
  [xmin=100, xmax=0,   data="Alice, 30"]

After T2 updates (T2's txn_id = 200):
  [xmin=100, xmax=200, data="Alice, 30"]   ← old version
  [xmin=200, xmax=0,   data="Alice, 31"]   ← new version
```

T1 (snapshot before T2 committed) sees the old version. T3 (snapshot after T2) sees the new version. Neither blocks the other.

**The cost:** Dead rows accumulate. Old versions no longer visible to any transaction are "dead tuples." PostgreSQL's `VACUUM` process periodically scans tables, identifies dead tuples, and reclaims space. Without `VACUUM`, tables grow indefinitely.

### InnoDB's Approach: Undo Log Chain

InnoDB stores only the **latest version** of each row in the B-tree (clustered index). Old versions live in a separate **undo log**. To find an older version, follow undo chain pointers backward:

```
B-tree leaf (latest only):
  [row_id=1, trx_id=200, roll_ptr→undo, data="Alice, 31"]

Undo log chain:
  [row_id=1, old_data="Alice, 30", prev_ptr→...]
  [row_id=1, old_data="Alice, 29", prev_ptr→...]
```

A reader needing snapshot S follows the undo chain until `trx_id ≤ S`.

Advantage over PostgreSQL: live pages don't bloat with dead versions. Disadvantage: reads needing old versions must follow pointer chains — extra I/Os for deep undo history.

### The Read View

Both systems create a **read view** at transaction start:
- Current maximum transaction ID (snapshot timestamp)
- Set of currently active (uncommitted) transaction IDs

A version is visible if its `xmin` is committed AND before the snapshot, AND its `xmax` is zero, uncommitted, or after the snapshot. This gives each transaction a consistent point-in-time view without any locks.

---

## Part 3: Buffer Pool Management

### Why Databases Don't Use the OS Page Cache

Databases manage their own page cache (the **buffer pool**) rather than relying on the OS page cache. Three key reasons:

**WAL ordering:** The database must ensure WAL records reach disk before corresponding dirty pages. If the OS page cache manages pages, the database has no control over flush ordering. This breaks the WAL invariant — pages might reach disk before their WAL records.

**Double caching:** If both the database and the OS cache the same pages, you're wasting memory. Databases use `O_DIRECT` to bypass the OS page cache.

**Smarter eviction:** A sequential table scan has completely different locality than a random point lookup. The database can implement scan-aware eviction (don't cache pages only accessed once by a full scan), which the OS cannot.

### Buffer Pool Architecture

Fixed-size memory region divided into frames, each holding one page. Each frame has a dirty bit, a pin count (can't evict pinned pages), and access history.

```
Buffer Pool:

Frame 0: [Page 17 of T] ← dirty=1, pin=0
Frame 1: [Page 5 of I]  ← dirty=0, pin=2  (2 active queries)
Frame 2: [Page 42 of T] ← dirty=0, pin=0
Frame 3: [EMPTY]
...
```

On page request: check page table (hash map: page_id → frame), if found return it, if not evict a victim frame (writing it if dirty), read the new page from disk.

### Page Replacement Policies

**LRU:** Evict the page accessed longest ago. Simple, effective for temporal locality. Catastrophic for sequential scans — a 100GB table scan flushes the entire buffer pool with never-reused pages ("sequential flooding").

**Clock (Second-Chance):** Cheaper approximation of LRU. Each frame has a reference bit (set to 1 on access). Eviction clock hand sweeps frames: if bit=1, set to 0 and skip; if bit=0, evict. Recently-accessed pages get a "second chance." Much lower overhead than LRU (no linked list manipulation per access).

**LRU-K (InnoDB uses K=2):** Track timestamps of the K most recent accesses. Evict the page whose K-th most recent access is oldest. With K=2, a page must be accessed twice before becoming "hot." Scan pages accessed only once are quickly evicted. InnoDB splits the buffer pool into "young" (hot) and "old" (cold/scan) areas; old area is evicted first.

### Dirty Page Management and Checkpoints

Too many dirty pages → recovery takes too long (must replay entire WAL since last checkpoint).

A **checkpoint** forces all dirty pages to disk and records a position in the WAL. Recovery only needs to replay WAL after the checkpoint. But naively flushing all dirty pages at once causes an I/O spike.

Modern databases use **fuzzy checkpoints**: begin checkpoint without waiting for all dirty pages to flush. Record the checkpoint start position. Background writer continuously flushes dirty pages. Recovery replays from the oldest unflushed dirty page.

PostgreSQL's `bgwriter` continuously evicts and flushes dirty pages in the background, targeting pages near the end of the eviction list — proactively writing them so eviction doesn't need to block on a write.

---

## Part 4: Copy-on-Write B-Trees

### The Core Idea

Standard B-trees modify pages in-place: read page, modify in memory, write back to same disk location. This requires WAL + torn write protection.

Copy-on-write B-trees never overwrite a page. When you modify a page, allocate a new page, write modified content there, update the parent to point to the new page, recurse up to the root:

```
Before update (key 42 → 57 in leaf L):

Root → Internal1 → Internal2 → L (contains 30, 42, 55)

Step 1: Write L' = (30, 57, 55)        (new leaf, new page)
Step 2: Write Internal2' → L'          (new internal, points to L')
Step 3: Write Internal1' → Internal2'  (new internal)
Step 4: Write Root' → Internal1'       (new root)
Step 5: Atomically update root pointer to Root'

After:
Root' → Internal1' → Internal2' → L' (contains 30, 57, 55)
Old pages (Root, Internal1, Internal2, L) intact for existing readers.
```

Step 5 — updating the root pointer — is the single commit point. Either it happens (entire change visible) or it doesn't (entire change invisible). No WAL needed for crash safety — the old tree is always intact on disk.

### LMDB: Copy-on-Write B-Tree in Practice

LMDB (Lightning Memory-Mapped Database) uses memory-mapped I/O — the entire database file is mapped into the process's virtual address space. Reads are memory accesses; the OS handles paging transparently. No buffer pool needed.

LMDB maintains **two root pointers** in the database header. Writers:
1. Copy-on-write all modified pages (building new tree)
2. fsync new pages to disk
3. Atomically update header to new root (single sector write — guaranteed atomic)
4. fsync header

Readers see the last committed root pointer. Because old pages are never overwritten, readers use any committed snapshot without locks. MVCC is free — old versions are simply the unchanged old pages. No undo log needed.

**Write amplification cost:** Every modification writes O(tree height) new pages. For height 4, a single key update writes 4 pages instead of 1. Acceptable for LMDB's target (embedded, read-heavy, single-writer-at-a-time), prohibitive for write-heavy workloads.

### btrfs: Copy-on-Write at the Filesystem Level

btrfs applies the same CoW principle to filesystem B-trees (metadata, directory entries, extent mappings). This gives btrfs:
- Crash consistency without a separate journal (old tree always valid)
- Cheap snapshots (duplicate root pointer; pages shared via reference counting)
- Atomic updates spanning multiple files

The write amplification problem is real for btrfs: write-heavy workloads fragment the filesystem and cause performance degradation over time (especially on SSDs with many small random writes).

---

## Part 5: Fractal Trees / Bε-Trees

### The Write Amplification Problem

In a standard B+-tree, a single key insertion touches O(log_B(N)) nodes. With node size B=1000 and N=10⁹, that's 3 nodes. But each node is a full page (4-16KB) — you're writing B bytes (full page) to change O(1) bytes. Write amplification is enormous.

LSM-trees solve this with sequential writes and batching, but introduce read amplification (multiple levels to search) and compaction stalls. Is there a structure that gets sequential writes without LSM's read penalty?

### The Bε-Tree: Write Buffering at Internal Nodes

The Bε-tree (B-epsilon tree, Brodal-Fagerberg, Bender et al.) adds a **write buffer** to every internal node. Instead of traversing to the leaf immediately, insert a **message** (deferred write) into the root's buffer:

```
Root node:
  ┌─────────────────────────────────────────────────────────────┐
  │  Keys: [200, 500, 800]                                       │
  │  Child pointers: [→C1, →C2, →C3, →C4]                       │
  │  Buffer: [Insert(150), Update(320), Delete(470), Insert(620)] │
  └─────────────────────────────────────────────────────────────┘
```

Messages accumulate in node buffers. When a buffer fills, **flush** it: send all messages down to the appropriate children. Those children buffer more messages. Writes only reach leaves in batches — batch size = ε×B messages per flush.

Parameter ε (epsilon): internal nodes use fraction ε for keys/pointers, fraction (1-ε) for buffer. A larger buffer means more messages per flush per I/O.

**Write amplification:** Each message crosses each level once in a batch flush. Cost per level: O(1/B) amortized — 1 I/O to flush B messages. Total per insertion: O(log_B(N)/B) — better than standard B-tree O(log_B(N)) by a factor of B. For B=1000, that's 1000× less write amplification.

**Read amplification:** A point lookup must check every level's buffer (messages may modify the target) plus the leaf: O(log_B(N)) — same as standard B-tree. Unlike LSM, no extra level-scanning penalty.

The Bε-tree achieves the theoretical minimum write cost for comparison-based data structures under the I/O model.

### TokuDB / Percona's Fractal Tree Index

The Fractal Tree Index is the practical Bε-tree implementation by Tokutek (now Percona), available as TokuDB in Percona Server for MySQL.

Messages include inserts, updates, and deletes. A DELETE inserts a "delete message" at the root buffer; it propagates to the leaf in a later flush — the leaf is never directly accessed for deletes. This makes large bulk inserts dramatically faster than standard B-trees: 1 million inserts = 1 million messages at the root, cascading down in bulk flushes. Random inserts are nearly as fast as sequential.

**Read tradeoff:** A lookup must check all buffers on the path from root to leaf. Fractal Trees are less ideal for read-heavy workloads where queries frequently read recently-written data before it has flushed to leaves.

---

## Part 6: B-Trees on SSDs vs. HDDs

### The HDD Cost Model (Where B-Trees Were Designed)

B-tree papers assumed spinning disk:
- **Random seek:** ~5-10ms (mechanical head movement)
- **Rotational latency:** ~2-5ms
- **Sequential transfer:** ~100-200 MB/s
- **Random-to-sequential ratio:** ~200:1

Everything in B-tree design optimizes to minimize random I/Os. Node size (4-16KB) was calibrated so one node read = one random I/O — you can't do it cheaper regardless. Larger nodes increase branching factor, reduce tree height, fewer seeks.

### The SSD Cost Model (Changed Calculus)

SSDs have no mechanical parts:
- **Random read latency:** ~0.05-0.1ms (100× faster than HDD)
- **Sequential throughput:** ~2-7 GB/s
- **Random read throughput:** Competitive with sequential (unlike HDD)
- **Random-to-sequential ratio:** ~5-20× (vs. 200× for HDD)

The catastrophic random I/O penalty is much smaller. Node sizes can be reconsidered — some systems experiment with larger nodes (16-32KB) for higher branching factors.

**But SSDs make writes worse.** An SSD's Flash Translation Layer (FTL) erases and rewrites at the **erase block** level (typically 256KB-2MB), even for small random writes. Writing 4KB may cause the FTL to: read a 256KB erase block, modify 4KB, erase the block, write 256KB back. Write amplification of 64×.

Sequential writes (like WAL appends) write to fresh pages without the erase-modify-write cycle, achieving near-raw write speeds. LSM trees' sequential write pattern is fundamentally better for SSDs than B-trees' random page writes.

### B-Tree Random Writes on SSD

Each B-tree page write (random location) triggers FTL overhead. A write to a 16KB B-tree page on an SSD with 256KB erase blocks still costs ~16× amplification at the flash layer. Modern SSDs have improved FTL algorithms, but random write amplification remains a concern — and is the primary reason SSDs wear out faster under B-tree write-heavy workloads.

### ZNS SSDs: A New Hardware Model

Zone Namespace (ZNS) SSDs expose the erase-block structure directly to the host. The host must write sequentially within each "zone" (erasure unit). Random overwrites within a zone are prohibited.

This matches LSM-tree's sequential-write pattern exactly — RocksDB has been adapted for ZNS SSDs. B-trees with random page updates are fundamentally mismatched with ZNS SSDs. Bε-trees, whose buffer flushes are batch sequential writes, are a reasonable fit.

### Summary by Storage Type

```
                        HDD         SSD (TLC/QLC)    ZNS SSD
─────────────────────────────────────────────────────────────
B+-tree (standard)      Good        Moderate          Poor
LSM tree                Good        Good              Excellent
Bε-tree / Fractal       Excellent   Good              Good
Copy-on-write B-tree    Moderate    Poor              Poor
─────────────────────────────────────────────────────────────
Rating reflects: write amplification × I/O latency
```

---

## How All Six Topics Connect

These are not independent topics — they form a layered system:

```
Hardware Layer (HDD / SSD / ZNS SSD)
    ↓ determines cost of random vs. sequential I/O
Buffer Pool (manages which pages are in memory)
    ↓ determines how often physical I/Os occur
WAL + Torn Write Protection (ensures crash safety)
    ↓ adds write amplification on top of buffer pool writes
B-Tree Variant (standard / CoW / Bε-tree)
    ↓ determines write patterns sent to WAL + buffer pool
MVCC (provides snapshot isolation on top of the B-tree)
    ↓ adds version management overhead to B-tree pages
```

The torn writes blog sits at the WAL layer — specifically addressing how to ensure that the atomic disk write unit (sector) and the logical database unit (page) don't create inconsistencies. PostgreSQL's full-page writes, InnoDB's double-write buffer, and LMDB's CoW are different answers to the same question at that layer.

DDIA Chapter 3 covers the B-tree vs. LSM-tree comparison at the storage engine level — the "B-Tree Variant" layer above. The cost models it describes (write amplification, read amplification, space amplification) are the lens through which all these variants are evaluated.
