# The RUM Conjecture: Read, Update, Memory — Pick Two

**Date:** 2026-07-27
**Sources:**
- [Designing Access Methods: The RUM Conjecture — Athanassoulis, Kester, Maas, Stoica, Idreos, Ailamaki & Callaghan (EDBT 2016)](https://openproceedings.org/2016/conf/edbt/paper-12.pdf)

**Related entries:**
- [008-lsm-trees.md](008-lsm-trees.md) — Entry #8 already invokes the RUM conjecture as "you can't have everything"; this is the primary source, and write amplification there is exactly RUM's *update overhead*
- [011-b-trees.md](011-b-trees.md) — B-trees are the classic read-optimized corner of the RUM triangle
- [012-b-trees-modern.md](012-b-trees-modern.md) — Bε-trees and CoW B-trees are deliberate slides across the RUM triangle (buffering trades reads for writes; CoW trades writes/space for crash-safety)
- [013-vector-search-ann.md](013-vector-search-ann.md) — HNSW's ~60–450 bytes/vector of graph links is *memory overhead*; the recall/latency/memory triangle there is a domain-specific RUM triangle
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — "which two of R/U/M did this system optimize?" is a fast diagnostic when choosing a store

---

## Why This Note Is Different

Most of the notes in this repo are deep dives into a specific system or data structure. This one is a **conceptual keystone** — a single idea that sits underneath many of the others and gives them a shared vocabulary. The paper itself is explicitly labelled a "Visionary Paper," not an experimental one. It doesn't prove a theorem or benchmark a system. Instead it proposes a *conjecture* — a lens — and argues that this lens explains four decades of data-structure design, and predicts where the field is heading. It's a short paper with a big, clarifying idea, and once you have it, you start seeing it everywhere.

The authorship is worth noting because it tells you the idea is grounded in real systems, not just theory. Among the seven authors are **Stratos Idreos** (the database-cracking / self-designing-data-structures line of work at Harvard) and **Mark Callaghan** (who ran RocksDB and MyRocks at Facebook). These are people who build the storage engines your other notes describe, stepping back to name the pattern they kept running into.

---

## The Setup: What "Access Methods" Are and What They Cost

The paper uses the term **access method** for what you'd loosely call a data structure plus the algorithms that read and update it — a B-tree, an LSM tree, a hash index, a bitmap, a Bloom filter. The central claim is that *every* access method, no matter how clever, is fighting the same three-way battle, and it has been the same battle since the 1970s. Only the hardware changes: in the 1970s the enemy was random disk seeks; forty years later it's random main-memory accesses; the underlying tension is identical.

The three things in tension are what the paper calls the **RUM overheads**. The crucial move is that each is defined as an *amplification ratio* — how much more work you do than the bare minimum the query logically required. This is the same "amplification" framing your LSM note (Entry #8) already uses, now generalized.

**Read overhead (R)** is *read amplification*: the total bytes you had to read — including all the auxiliary index structure, not just the answer — divided by the bytes you actually wanted. When you walk a B+-tree to fetch one tuple, you read a chain of internal nodes purely to navigate; those extra reads are your read overhead.

**Update overhead (U)** is *write amplification*: the physical bytes written to perform one logical update, divided by the size of that logical update. This is precisely the write-amplification number from Entry #8 — a single inserted byte that gets rewritten thirty times as it's merged down through LSM levels is an update overhead of 30.

**Memory overhead (M)** is *space amplification*: the total bytes stored — auxiliary structures plus base data — divided by the base data alone. The 60–450 bytes of graph links per vector that HNSW stores on top of the raw vectors (Entry #13) is memory overhead. So is the obsolete-version bloat an LSM tree carries before compaction reclaims it.

The theoretical best for each overhead is a ratio of exactly **1.0**: read only the bytes you asked for, write only the bytes that changed, store nothing beyond the base data. The whole point of the paper is that **you cannot achieve 1.0 on all three at the same time.**

---

## The Argument: Chase Any Corner to Perfection and the Other Two Explode

What makes the paper convincing is that it doesn't wave its hands. It picks a deliberately trivial dataset — an array of integers, stored in fixed-size blocks — and shows concretely that driving any *one* overhead to its perfect 1.0 value forces the other two to blow up. The simplicity is the point: if the tension is unavoidable even here, it's unavoidable everywhere.

**To minimize reads perfectly,** store the value `v` directly in block number `v`, so you always know exactly where to look — one read, nothing wasted. But now the array is mostly empty gaps (you can't predict the largest value you'll ever store), so memory overhead runs to infinity. And changing a value means clearing its old block and filling a new one — two physical writes for one logical change, so update overhead is 2.0.

```
   Prop 1:   min(Read) = 1.0   ⇒   Update = 2.0  and  Memory → ∞
```

**To minimize updates perfectly,** never reorganize anything — just append every change to an ever-growing log. Each write is one write; update overhead is a perfect 1.0. But now reads have to scan and merge the entire log to reconstruct the current value, so read overhead grows without bound, and the log keeps consuming more and more space, so memory overhead does too.

```
   Prop 2:   min(Update) = 1.0   ⇒   Read → ∞  and  Memory → ∞
```

**To minimize memory perfectly,** store nothing but a dense array of the base data — no index at all. Space overhead is a perfect 1.0, and in-place updates touch only the data itself. But with no index, finding anything means scanning the whole dataset, so read overhead becomes N.

```
   Prop 3:   min(Memory) = 1.0   ⇒   Read = N  and  Update = 1.0
```

Every corner is *reachable* — but only by paying dearly on the other two axes. That is the entire argument, in miniature.

---

## The Conjecture, Stated Precisely

Having shown that chasing perfection in one dimension is impractical, the paper makes the realistic version of the claim:

> **An access method that sets an upper bound on two of the read, update, and memory overheads also sets a hard lower bound on the third — a bound that cannot be reduced further.**

Said more simply: **you choose which two of Read, Update, and Memory to optimize, and you pay for that choice on the third.** There is no free lunch and no universal winner. This is the same shape of law as the CAP theorem for distributed systems or the RUM-adjacent tradeoffs you've already seen — a statement that a certain kind of "have it all" is simply not on the menu.

---

## The Triangle: The Picture Worth Memorizing

The paper's key figure lays the three goals at the corners of a triangle and drops real data structures into it according to which overheads they favor. This is the mental model to carry around:

```
                          READ-OPTIMIZED
                       (minimize read overhead)
                     Hash indexes · B-Trees · Tries
                            · Skip lists ·
                                 /  \
                                /    \
                               /      \
                              /  adaptive  \
                             / (cracking,   \
                            /   merging)      \
                           /                   \
                          /                     \
            WRITE-OPTIMIZED ───────────────── SPACE-OPTIMIZED
        (minimize update overhead)        (minimize memory overhead)
        LSM-trees · Partitioned           Bloom filters · bitmaps
        B-trees · MaSM ·                  · sparse indexes (ZoneMaps)
        differential structures           · approximate indexes
```

Each corner corresponds to a family of designs — and, satisfyingly, to notes you've already written:

The **read-optimized** corner holds structures built for fast lookups: hash indexes, B-trees, tries, skip lists. Your **B-tree notes (#11–#12)** live here. They pay for their fast reads with extra space and comparatively expensive updates (every insert may trigger a page split, and in-place updates demand write-ahead logging and torn-write protection).

The **write-optimized** corner holds structures that make updates cheap by refusing to reorganize eagerly — they buffer or log changes and apply them in bulk later. The paper's headline examples are the **Log-Structured Merge tree**, partitioned B-trees, and MaSM. This is exactly your **LSM-tree note (#8)**. The price is paid on reads (a query may have to check many levels) and on space (obsolete versions linger until compaction).

The **space-optimized** corner holds structures that shrink the footprint, often by becoming *lossy or probabilistic*. The paper's flagship example is the **Bloom filter** — which you already met in Entry #8. A Bloom filter saves enormous space precisely by accepting false positives; it trades a little read accuracy for a lot less memory. That trade *is* the RUM conjecture in a single data structure.

The **middle** holds *adaptive* structures like database cracking, which don't sit at a fixed point — they *move* through the triangle as the workload teaches them what to optimize, gradually building index structure on the columns that queries actually touch.

The paper reinforces the picture with a complexity table comparing a B+-tree, a hash index, ZoneMaps (a sparse index), and a leveled LSM tree. The punchline is that **no column has a single winner**: hash indexes give the best point queries, B+-trees the best range queries, ZoneMaps the smallest index size, and LSM trees the cheapest updates. Every structure wins somewhere and loses somewhere else — the triangle made concrete.

---

## Two Subtleties Most Summaries Miss

The first is that **computation is the escape valve.** The conjecture is about the fundamental floor on R, U, and M — but it doesn't say you're trapped. You *can* nudge all three overheads down at once, by spending CPU instead. Compression is the obvious case: it shrinks memory *and* the bytes moved for reads, at the cost of compress/decompress work. Exploiting structure you already know does the same — a clustered index storing only a handful of page pointers cuts both read and memory overhead by spending a little arithmetic to compute locations. Computation moves cost *off* the R/U/M axes rather than violating the triangle. This is why "operate on compressed data, decompress as late as possible" is a pervasive modern design.

The second is that **the triangle applies per level of the memory hierarchy, and you can trade vertically.** The paper's simple examples pretend everything lives on one storage medium, but real systems span cache → RAM → flash → disk, and each level has its own R/U/M balance. That opens a new move: cache or buffer more data at level *n−1* (raising *its* memory overhead) to cut the read and update cost at level *n*. The tradeoff can be read vertically across the hierarchy, not just horizontally within one level — and the authors expect this to get more complex as hardware hierarchies deepen.

---

## The Vision: Structures That Move Through the Triangle

Because no universal winner exists and hardware and workloads keep shifting, the paper argues the future belongs to **RUM-aware, tunable, "morphing" access methods** — structures that can slide around inside the triangle at runtime rather than being redesigned from scratch every few years. They sketch examples: B-trees that dynamically retune node size and split conditions; approximate indexes that absorb updates into updatable probabilistic structures; bitmap indexes that stay update-friendly by buffering changes in compressible side-vectors. This vision is essentially the research program Idreos's group went on to pursue under the banner of "self-designing data structures." The deeper message, in the authors' own words, is that **there is no panacea** — you cannot build the universally best access method, so the goal shifts to building ones that adapt gracefully.

---

## Why This Is Useful

The RUM conjecture earns its place in this repo for three reasons.

First, **it's the unifying spine under things already written here.** Write amplification in the LSM notes, the memory cost of HNSW's graph, the read-versus-write-versus-space table in the B-tree notes — those are three encounters with the same underlying law. RUM names it. Entry #8 already gestures at "the RUM conjecture says you can't have everything"; this note is the source that line points to.

Second, **it's a fast diagnostic when choosing or evaluating a store.** For any system, ask: *which two of Read, Update, and Memory did the designers optimize, and where did they pay for it?* LSM/DynamoDB chose update and memory, and paid on reads — which is why they bolt on Bloom filters to claw some read performance back. B-trees/PostgreSQL chose reads, and paid on writes and space. It turns a vague "this is fast" into a specific "it moved its point *here*, so it must be worse *there* — let me check whether that matters for my workload."

Third, **it sharpens how to read every future storage paper.** Whenever a paper claims a win, the RUM lens prompts the right skeptical question: *at whose expense?* Somewhere, some overhead got worse, or some CPU got spent, or some level of the hierarchy absorbed the cost. Finding that hidden price is usually where the real understanding is.

---

## Key Takeaways

- **Read, Update, and Memory overheads form a competing triangle;** you can optimize two, and the third is then bounded below — it cannot be pushed to its ideal.
- **Each overhead is an amplification ratio** (bytes touched ÷ bytes logically needed), with 1.0 as the unreachable-for-all-three ideal.
- **The corners map to familiar families:** read-optimized (B-trees, hashes), write-optimized (LSM trees, differential structures), space-optimized (Bloom filters, bitmaps, sparse indexes) — all of which appear elsewhere in this repo.
- **Computation is the escape valve:** compression and exploiting known structure can lower all three overheads by spending CPU, without breaking the conjecture.
- **The triangle repeats at every level of the memory hierarchy,** and you can trade overheads vertically by caching more at a higher level.
- **There is no universal best access method** — the paper's real thesis — which is why the future points toward adaptive, tunable structures that morph through the triangle as needs change.
