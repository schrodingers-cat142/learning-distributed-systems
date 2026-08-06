# Sharding: Strategies, Multitenancy, and Hot Spots

**Date:** 2026-08-06
**Sources:**
- DDIA Chapter 6 (Sharding/Partitioning) — pros/cons, multitenancy, sharding key-value data, skewed workloads
- [Sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion)
- [Consistent Hashing: Algorithmic Tradeoffs — Damian Gryski](https://dgryski.medium.com/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8)
- [Building and Operating a Pretty Big Storage System Called S3 — Andy Warfield (2023)](https://www.allthingsdistributed.com/2023/07/building-and-operating-a-pretty-big-storage-system.html)
- [Things Databases Don't Do… But Should — Gwen Shapira, Nile](https://www.thenile.dev/blog/things-dbs-dont-do)

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — sharding is *different data on different nodes*; replication is *same data on many nodes*. Real systems do both, and #016 already drew that distinction
- [027-leaderless-replication.md](027-leaderless-replication.md) — Dynamo-style consistent hashing (the ring) originated there; this note goes deep on the algorithm family
- [029-dynamo-to-dynamodb.md](029-dynamo-to-dynamodb.md) — DynamoDB's partitions are hash-range shards; its hot-partition saga is the canonical skewed-workload story
- [009-bigtable.md](009-bigtable.md) — Bigtable's tablets are key-range shards that split dynamically — the template for key-range sharding
- [031-sharding-operations.md](031-sharding-operations.md) — the operational half: once you've *chosen* a scheme (this note), how do you rebalance, route, and index across shards (that note)?
- [014-rum-conjecture.md](014-rum-conjecture.md) — every sharding choice is a tradeoff (range-scans vs even spread); the recurring "pick your point, pay elsewhere" shape

> **Note on scope:** This is the first of two sharding notes. It covers *designing the sharding scheme* — why shard, sharding for multitenancy, the ways to split key-value data, consistent hashing in depth, and relieving hot spots. The *operational* half — rebalancing, request routing, coordination services, and secondary indexes — is Entry #031. Read this one for "how do I divide the data?"; read #031 for "how do I run the result?"

---

## What Sharding Is, and Why

**Sharding** (DDIA's term; also called **partitioning**) means splitting a dataset so that *different records live on different nodes*. It is the answer to a hard limit: a single machine can only hold so much data and serve so many requests. When one node isn't enough, you split the data across many.

The first thing to fix is the distinction from replication (Entries #016–#028), because they're constantly confused and almost always combined:

```
   REPLICATION                          SHARDING (PARTITIONING)
   ───────────                          ───────────────────────
   the SAME data on many nodes          DIFFERENT data on different nodes
   → redundancy + read scaling          → more total data + more write capacity
   ┌─────┐ ┌─────┐ ┌─────┐              ┌─────┐ ┌─────┐ ┌─────┐
   │ABC  │ │ABC  │ │ABC  │              │ A   │ │  B  │ │   C │
   └─────┘ └─────┘ └─────┘              └─────┘ └─────┘ └─────┘
```

Real systems do **both**: each shard is itself replicated across several nodes for durability. So a "shard" is usually a *replication group* (as in DynamoDB, Entry #029, where each partition is a Multi-Paxos group). This note is only about the *splitting* dimension; replication is orthogonal and already covered.

**The pros of sharding** are the reasons you'd ever take on its complexity:
- **Scale beyond one machine** — store more data and handle more read *and* write throughput than any single node (unlike read replicas, which scale only reads).
- **Isolate workloads** — different shards can be operated, tuned, and blast-radius-contained separately (the multitenancy angle, below).

**The cons** are real and the reason you shouldn't shard until you must:
- **Lost cross-shard operations** — a query, join, or transaction spanning shards is far harder and slower; you lose the single-node database's easy consistency across all data.
- **Hot spots** — if the split is uneven, one shard gets hammered while others idle, and you've gained nothing (the skew problem, a whole section below).
- **Operational weight** — rebalancing, routing, and cross-shard indexes are all new problems (Entry #031).

Notion's story captures *when* to shard: they waited until their single Postgres monolith was already straining — `VACUUM` was stalling (risking transaction-ID wraparound, which would halt all writes), and CPU spikes were paging on-call. Their retrospective lesson is pointed: **shard earlier than you think**, because sharding *while already overloaded* forces frugal, painful choices (they couldn't even use logical replication for the migration because the DB couldn't take the load). But there's a counter-tension: shard *too* early and you risk locking in a data model before the product is settled — Notion notes that sharding by "user" and later pivoting to a team-centric product would cause an "architectural impedance mismatch." The judgment call is real.

---

## Sharding for Multitenancy (and Cell-Based Architecture)

One of the most common reasons to shard — especially for SaaS — is **multitenancy**: your database holds data for many independent customers (tenants), and the tenant is a *natural shard key*. This is the lens Notion and Nile both use, and it deserves its own treatment because it changes *why* you shard from "the data is too big" to "I want isolation between customers."

### Isolated vs Pooled

Nile frames the fundamental choice as two models:

```
   ISOLATED MODEL                        POOLED MODEL
   ──────────────                        ────────────
   each tenant gets its OWN database     ALL tenants share one database,
                                          each row tagged with a tenant_id
   ┌────────┐ ┌────────┐ ┌────────┐      ┌──────────────────────────┐
   │tenant A│ │tenant B│ │tenant C│      │ A A B C A B B C A C ...   │
   └────────┘ └────────┘ └────────┘      │ (tenant_id column on rows)│
   + tenant isolation "for free"         └──────────────────────────┘
   + easy per-tenant ops/backup          + far cheaper & more scalable
   − hard to scale to many tenants       − you must enforce isolation yourself
   − wasteful for small tenants          − one missing WHERE = data leak
```

The **isolated** model gives isolation, per-tenant backups, and per-tenant upgrades essentially for free, but it scales terribly — thousands of tenants means thousands of databases to operate, and small tenants waste a whole database each. The **pooled** model (one shared DB, a `tenant_id` column on every row) is cheap and scalable, but — as Nile stresses — *the database doesn't understand tenancy*, so you must enforce isolation in application code "and hope no one ever misses a `WHERE` clause." A single forgotten `WHERE tenant_id = ...` is a cross-tenant data leak. That's a scary property to depend on human discipline for, and it's Nile's central argument: databases *should* provide tenancy natively.

### Sharding by Tenant

The practical middle path — and what Notion actually did — is **shard by tenant**: pool many tenants per shard, but place different tenants on different shards. Notion shards by **workspace ID**: every workspace gets a UUID at creation, and because "each block belongs to exactly one workspace," partitioning by workspace keeps a block and all its related rows co-located on one shard. This is the ideal shard key for their access pattern — users work within one workspace at a time, so it avoids cross-shard joins almost entirely, and transactions (which only hold within a single datastore) stay intact.

### Cell-Based Architecture

Push tenant-sharding to its architectural conclusion and you get **cell-based architecture**: the whole stack — not just the database, but the compute, queues, and caches — is replicated into independent **cells**, each serving a subset of tenants. A tenant lives entirely within one cell.

```
   ┌─── CELL 1 ───┐  ┌─── CELL 2 ───┐  ┌─── CELL 3 ───┐
   │ app + db +   │  │ app + db +   │  │ app + db +   │
   │ queue + cache│  │ queue + cache│  │ queue + cache│
   │ tenants A,D  │  │ tenants B,E  │  │ tenants C,F  │
   └──────────────┘  └──────────────┘  └──────────────┘
        a bug or overload in Cell 2 can only hurt B and E —
        the "blast radius" is bounded to one cell
```

The point of cells is **blast-radius containment**: a bad deployment, a poison-pill request, an overload, or data corruption in one cell affects only that cell's tenants, not everyone. It's the same instinct as bulkheads in a ship's hull. A thin routing layer maps each tenant to its cell (routing is Entry #031). Cells are how you get the isolation benefits of the "isolated model" while still pooling tenants for efficiency within each cell.

### The Challenges of Multitenant Sharding

Multitenancy makes sharding harder in specific ways, and the sources name them:

- **Tenants are wildly uneven.** Notion: one large enterprise customer "generates more load than many average personal workspaces combined." So a naive even split of tenants across shards produces very *uneven load* — the hot-spot problem, tenant-flavored. You can't assume tenants are interchangeable units.
- **Noisy neighbors.** In a pooled shard, one tenant's traffic spike degrades every co-resident tenant. Isolation isn't just about data visibility; it's about performance.
- **Per-tenant operations are hard but needed.** Nile's cautionary example: Atlassian's multi-day outage, where rolling back a change for a *subset* of customers required "a very manual process" because the system couldn't operate on one tenant independently. Independent per-tenant backup, restore, and upgrade are "something few companies think about in advance, and hard to implement when actually needed."
- **Isolation must extend through the whole pipeline.** Nile's subtle point: the tenant-isolation problem *recurs downstream* — if you capture change events (CDC, Entry #017) into Kafka, "you now need to solve the same problem in the change capture system as well." Tenancy leaks are possible at every hop, not just the primary store.

Nile's overall thesis: the four core sharding responsibilities — **placement** (which shard for a new tenant), **routing** (direct each request to the right shard), **scale-out** (add shards), and **rebalancing** (balance load) — should be *native database capabilities*, not application plumbing every SaaS team rebuilds. Most databases make you build all four yourself.

---

## Sharding Key-Value Data

Set multitenancy aside and consider the general problem: you have key-value data (every record has a key), and you must decide *which shard each key goes to*. There are three main schemes, and the tradeoffs between them are the heart of DDIA's chapter.

### Scheme 1: Sharding by Key Range

Assign each shard a *contiguous range of keys*, kept sorted — like the volumes of an encyclopedia (A–C, D–F, …). This is what Bigtable/HBase (Entry #009) tablets do, and what any sorted-key store does.

```
   Shard 1: keys [A … F]     Shard 2: keys [G … M]     Shard 3: keys [N … Z]
```

**The big win: range scans.** Because keys are sorted within and across shards, a query like "all records between K1 and K2" reads a contiguous run — cheap and efficient. If your access pattern is range-heavy (time-series, "all events in this hour"), key-range sharding is ideal.

**The big risk: boundary hot spots.** If keys are accessed in a pattern that concentrates on one range, that shard is hammered. The classic failure is a **timestamp key**: if you shard sensor data by `timestamp`, then *today's* writes all land on the single shard owning the current time range, while all the historical shards sit idle. You've built a system that can only ever use one shard at a time for writes. The fix is to prefix the key with something high-cardinality (e.g., `sensor_id` before `timestamp`) so writes spread — but then a range scan over time requires querying all shards.

Key-range shards **rebalance by splitting**: when a shard gets too big or too hot, split its range in two and move half elsewhere (the operational mechanics are Entry #031). This dynamic splitting is exactly what Bigtable tablets and DynamoDB partitions do.

### Scheme 2: Sharding by Hash of Key

To spread load *evenly* regardless of access pattern, hash the key and use the hash to place it. A good hash scatters even sequential keys (`user1`, `user2`, `user3`) uniformly across shards, destroying the hot-spot-by-adjacency problem.

But *how* you use the hash matters enormously, and the naive way is a trap.

**The mod-N trap.** The obvious approach: `shard = hash(key) % N`, where N is the number of nodes.

```
   shard = hash(key) % N
```

This works until N changes — and then it's catastrophic. Because the result depends on N, *adding or removing a single node changes the modulus, so almost every key remaps to a different shard.* Going from 9 nodes to 10 doesn't move ~1/10th of the keys (as you'd hope); it moves nearly *all* of them. For a cache that's a total miss-storm; for a database it's a full data reshuffle. **Never shard by `hash % N` if N can change.** (Gryski: this is the entire motivation for consistent hashing, next.)

**The fixed-number-of-shards fix.** DDIA's pragmatic answer (used by Riak, Elasticsearch, Couchbase): create a **fixed, large number of shards** up front — many more than you have nodes — and assign whole shards to nodes. Say 1,000 shards across 10 nodes = 100 shards each. When you add an 11th node, you don't recompute any hashes; you just *move ~91 whole shards* onto it from the others. The key→shard mapping (`hash(key) % 1000`) never changes; only the shard→node mapping changes.

```
   Fixed 1000 shards, key→shard mapping NEVER changes:
      hash(key) % 1000 → shard number   (stable forever)

   Only the shard→node assignment moves when nodes change:
      10 nodes: 100 shards each
      11 nodes: move ~91 shards onto the new node; the other 909 stay put
```

This is why Notion chose **480 logical shards across 32 physical databases** — 480 is "divisible by a lot of numbers," so logical shards distribute evenly as physical hosts are added (32 → 40 → 48 all divide 480). Their explicit lesson: **"Pick values with a lot of factors!"** (512 has only power-of-two factors, forcing an awkward jump straight to 64 hosts.) The downside of hashing: you **lose efficient range scans** — adjacent keys are scattered, so a range query must hit every shard.

### Scheme 3: Sharding by Hash Range (the modern default)

The two schemes above trade off: key-range gives range scans but risks hot spots; hashing spreads evenly but kills range scans. **Hash-range sharding** is the clever synthesis that most modern systems use: hash the key to get a hash value, then **range-partition the *hash space*** (not the key space).

```
   1. hash(key) → a value in, say, [0, 2⁶⁴)
   2. split that HASH space into contiguous ranges, one per shard:
      Shard 1: hash ∈ [0,           2⁶⁴/3)
      Shard 2: hash ∈ [2⁶⁴/3,   2·2⁶⁴/3)
      Shard 3: hash ∈ [2·2⁶⁴/3,     2⁶⁴)
```

This gives you the best of both: hashing **spreads load evenly** (no adjacency hot spots), *and* range-partitioning the hash space means shards can **split and merge dynamically** exactly like key-range shards (just split a hash range in two) — without the mod-N reshuffle. You lose range scans *on the original key* (the hash scrambles order), but you gain even distribution plus elastic dynamic splitting.

This is what **DynamoDB** (Entry #029 — partitions split by consumed throughput), **MongoDB** (hashed sharding), and **YugabyteDB** all do. It's the default modern choice precisely because it decouples "spread evenly" from "resize elastically." (Some systems, like MongoDB and Yugabyte, let you choose *per table* whether to shard by hash-range or plain key-range, depending on whether that table needs range scans.)

---

## Consistent Hashing

Consistent hashing is the family of algorithms that solve the mod-N problem *directly* — letting you map keys to nodes such that adding/removing a node moves only ~1/N of the keys, with no central directory (every client computes the mapping independently and they all agree). Gryski's article is a superb tour of the whole family, and it's worth going deep because the tradeoffs recur everywhere.

First, the goal. An ideal key→node function has four properties, always in tension:

```
   • Balance          — keys spread evenly across nodes
   • Minimal disruption — adding/removing a node moves only ~1/N of keys,
                          and NEVER moves a key between two nodes that both stayed
   • Low memory       — the lookup structure is small
   • Fast lookup      — resolving key→node is cheap
```

No algorithm wins on all four — each "struggles to balance distribution, memory, lookup time, and construction time." Here are the five main approaches.

### Ring Hashing (Karger et al., 1997)

The classic, and the one Dynamo (Entry #027/#029), Cassandra, and Riak use. Map the hash space onto a circle. Hash each *node* to point(s) on the ring. To place a key, hash it to a point and **walk clockwise to the first node**.

```
        (ring)                     lookup = binary search over sorted node points → O(log n)
         N1
      N4    N2  ← key hashes here, walks clockwise → N2 owns it
         N3
```

A single point per node gives very uneven arcs, so each node is hashed to *many* points — **virtual nodes** (vnodes). This is where the central tradeoff bites: **more vnodes = better balance but more memory.** Gryski's numbers make it concrete:
- **100 vnodes/node** → load standard deviation ≈ 10% (some node carries up to 28% above average — a poor peak-to-average ratio).
- **1000 vnodes/node** → stddev ≈ 3.2%, but the ring structure is ~4 MB at 1000 nodes, and every lookup is a cache-cold binary search.

So ring hashing buys arbitrary add/remove flexibility, but pays a lot of memory (and cache-unfriendly lookups) to get low load variance.

### The Alternatives (and why they exist)

Each alternative fixes a specific weakness of the ring:

| Algorithm | How it works | Lookup | Memory | Add/remove | Key limitation |
|---|---|---|---|---|---|
| **Ring (Karger)** | walk clockwise on a circle of vnodes | O(log n) | High (~4 MB @1000 nodes) | Arbitrary | high memory to get low variance; cache-cold |
| **Jump hash** (Google 2014) | seed a PRNG with the key, "jump" through buckets, return last | O(ln n), in-register | **None** | **Only at the top of the range** | returns a shard *number*, not a node; **can't remove arbitrary nodes**; hard to weight |
| **Multi-probe** (Google 2015) | hash each node once; hash the *key* k times, closest node wins | O(k) | O(n) | O(1) | slower lookups as k grows (k=21 for 1.05 peak/mean) |
| **Rendezvous / HRW** (1997) | hash (key, node) for every node; highest score wins | **O(n)** | O(n) | Easy | linear scan — only for *small* node counts; but trivially weighted |
| **Maglev** (Google 2016) | precompute a lookup table (permutation); read one entry | **O(1)** | Low | minimal-disruption only | slow table rebuild caps max node count; assumes failures are rare |

The intuitions worth keeping:
- **Jump hash** has *near-perfect* distribution (stddev ~0.0000008%) and *zero* memory — but it returns bucket *0..N-1* (a shard number, not an arbitrary node identity) and can only grow/shrink at the *end* of the range, so it can't handle a random node in the middle crashing. Perfect for **data-storage sharding** where replication covers failures; useless for a fleet where any node can drop out.
- **Multi-probe** gets ring-like flexibility with O(n) memory instead of megabytes, by moving the cost into slightly slower (k-probe) lookups.
- **Rendezvous** is dead simple and cleanly weighted, but its O(n) per-lookup scan limits it to small clusters.
- **Maglev** (a load-balancer design) gives O(1) lookups and low memory, optimized for *minimal disruption* when a backend fails — but rebuilding its table is slow, capping how many backends it supports.

### Bounded Loads and Shuffle Sharding

Two important layers on top:

**Consistent hashing with bounded loads** (Google 2016): as keys are assigned, check the target node's current load and *skip it if it's already too loaded*, falling through to the next candidate. This caps any single node's load — used in HAProxy. It's the fix for consistent hashing's residual imbalance under skew.

**Shuffle sharding** (Amazon's technique, related to Gryski's "deterministic subsetting"): instead of assigning each tenant to *one* shard, assign each tenant to a small *random subset* of nodes. Two tenants might overlap on one node but are very unlikely to share their *entire* subset. So if one tenant sends a poison-pill request that takes down its nodes, only the tiny fraction of other tenants who share that *exact* subset are affected — everyone else is fine. It's a probabilistic blast-radius reduction, and it pairs naturally with cell-based architecture and multitenancy.

### A Caveat: Databases Use Consistent Hashing Less Than You'd Think

DDIA makes a point worth internalizing: despite its fame, the Karger *ring* is used less in databases than you'd expect. Many "consistent hashing" databases actually use the **fixed-number-of-shards** or **hash-range** schemes above, because those give cleaner control over shard placement and dynamic splitting. Ring hashing shines in **caching and load balancing** (memcached clients, CDNs, Maglev), where you genuinely need decentralized agreement with no coordinator and can tolerate its load variance. Know the family, but don't assume the ring is the default answer for a database.

---

## Skewed Workloads and Relieving Hot Spots

Even with a perfect sharding scheme that spreads *keys* evenly, you can still get a **hot spot** — because *access* is uneven even when *data* is even. The pathological case: a single key is red-hot. A celebrity's user record, a viral tweet, a flash-sale product. Hashing doesn't help here — that one key maps to one shard no matter how good your hash is, and that shard melts. This is exactly DynamoDB's hot-partition problem (Entry #029): splitting a partition can't help when all the traffic is a *single item*.

The mitigations:

**Split a hot key artificially.** Add a random prefix/suffix (say, 2 digits, 00–99) to a known hot key, turning one key into 100 keys spread across shards. Now reads must query all 100 variants and combine — extra read work, in exchange for spreading the write load. DDIA notes this is a manual, application-level bookkeeping burden (you must track *which* keys are hot and remember to fan out reads), which is why it's a targeted fix, not a default.

**The S3 insight: decorrelation and heat management.** Andy Warfield's S3 essay reframes hot spots at massive scale, and it's the most illuminating way to think about it. The physical reality: disks have grown enormously in *capacity* but barely in *IOPS* (a disk still does ~120 random operations/second — "the same in 2006"). So a modern high-capacity disk is IOPS-starved; you *cannot* serve a demanding workload from a few disks.

His key observation about workloads: "most storage workloads are completely idle most of the time and then experience sudden load peaks," so each workload's peak is far above its mean. A single-tenant system must provision for its own peak and sits mostly idle. But:

```
   ONE bursty workload:        many idle stretches, occasional huge spike
   ▁▁▁▁▁█▁▁▁▁▁▁▁█▁▁▁   peak ≫ mean

   AGGREGATE of MANY uncorrelated workloads:  the spikes don't line up
   ▄▅▄▆▅▄▅▆▅▄▅▄▆▅▄▅   peak ≈ mean  ← smooth, predictable
```

This is **decorrelation**: when many *independent* workloads share one huge fleet, "individual requests become decorrelated," and "as we aggregate millions of workloads a really cool thing happens: the aggregate demand smooths and becomes way more predictable." Because independent peaks don't coincide, it becomes "difficult or impossible for any given workload to influence the aggregate peak at all." It's the law of large numbers applied to load.

S3's **heat management** applies this: "heat" is requests-per-disk, and the goal is to "spread heat as evenly as possible." The mechanism is to **spread each object/customer thinly across a huge fleet** — "a customer's data only occupies a very small amount of any given disk, which helps achieve workload isolation," so "individual workloads can't generate a hotspot on any one disk." The beautiful part: the *same* thin-spreading mechanism gives *both*:
- **Workload isolation** — no tenant can concentrate enough requests to overheat one disk, and
- **Workload sharing** — a single customer's burst can be "served by over a million individual disks," a performance level "that just wouldn't be practical to build as a stand-alone system."

And **redundancy doubles as a heat tool**: with replication or erasure coding, S3 can "read from any of the disks" (or any k-of-(k+m) shards), steering reads to the *least-loaded* copy. Redundancy isn't just for durability — it's a read-load-balancing lever.

The transferable lesson — and why this belongs in a sharding note — is that **scale is the enabling ingredient, not just a cost.** The diversity of many uncorrelated tenants is precisely what lets a shared system spread heat smoothly and offer any single tenant more performance than it could ever build alone. Fine-grained spreading across a large fleet is the deepest answer to hot spots.

---

## Key Takeaways

- **Sharding = different data on different nodes** (vs replication = same data on many nodes); real systems combine both, so a shard is usually itself a replication group. Shard to scale storage + write throughput and to isolate workloads — but you pay in lost cross-shard operations, hot-spot risk, and operational weight. Shard *earlier than overload forces you to*, but not so early you lock in the wrong data model (Notion).
- **Multitenancy** makes tenant the natural shard key. **Isolated** (DB-per-tenant: easy isolation, poor scale) vs **pooled** (shared DB + `tenant_id`: cheap, but a missing `WHERE` = leak). **Cell-based architecture** replicates the whole stack into cells for **blast-radius containment**. Challenges: wildly uneven tenants, noisy neighbors, hard per-tenant ops, and isolation that must extend through the whole pipeline (Nile).
- **Three key-value sharding schemes:** **key-range** (sorted; great range scans, but boundary hot spots like timestamp keys); **hash-of-key** (even spread, but the **mod-N trap** reshuffles everything when N changes — fix with a **fixed large number of shards**, "pick a number with many factors"); and **hash-range** (hash then range-partition the *hash space*) — the modern default (DynamoDB, MongoDB, Yugabyte) giving even spread *and* elastic dynamic splitting, at the cost of range scans.
- **Consistent hashing** solves mod-N directly: **ring/Karger** (arbitrary add/remove, but high memory for vnodes to get balance), **jump** (perfect distribution + zero memory, but shard-number-only and top-of-range-only), **multi-probe** (ring flexibility, O(n) memory, slower lookups), **rendezvous** (simple + weightable, but O(n) lookups → small clusters), **Maglev** (O(1) lookup, low memory, but slow rebuild). Plus **bounded loads** (cap per-node load) and **shuffle sharding** (assign each tenant a random node *subset* → probabilistic blast-radius reduction). Caveat: databases often use fixed-shards/hash-range, not the ring — the ring shines in caching/load-balancing.
- **Hot spots** persist even with even *key* distribution because *access* is skewed (one celebrity key). Fixes: artificially **split a hot key** (random suffix + fan-out reads), and — the deep answer — **spread thinly across a huge fleet** so that **decorrelation** (S3) smooths aggregate demand: the same mechanism gives both workload isolation and the ability for one tenant to burst across a million disks. Scale is the enabling ingredient, not just a cost.
