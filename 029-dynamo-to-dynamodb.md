# Dynamo to DynamoDB: The Complete Picture

**Date:** 2026-08-03
**Sources:**
- [Dynamo: Amazon's Highly Available Key-value Store — DeCandia et al. (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service — Elhemali et al. (USENIX ATC 2022)](https://www.usenix.org/system/files/atc22-elhemali.pdf)
- [Dynamo, DynamoDB, and DSQL — Marc Brooker (2025)](https://brooker.co.za/blog/2025/08/15/dynamo-dynamodb-dsql.html)
- [The DynamoDB paper — Marc Brooker (2022)](https://brooker.co.za/blog/2022/07/12/dynamodb.html)

**Related entries:**
- [027-leaderless-replication.md](027-leaderless-replication.md) — Dynamo (2007) is *the* canonical leaderless system; that note's quorums, hinted handoff, and anti-entropy are all Dynamo's mechanisms, explained here in their original context
- [028-time-clocks-concurrent-writes.md](028-time-clocks-concurrent-writes.md) — Dynamo's vector clocks and the shopping-cart siblings/merge are the concrete application of that note's version vectors
- [016-single-leader-replication.md](016-single-leader-replication.md) — the great irony: DynamoDB (the product) abandoned Dynamo's leaderless design for *single-leader-per-partition* via Paxos
- [009-bigtable.md](009-bigtable.md) — DynamoDB's storage replica (B-tree + write-ahead log, archived to a durable object store) echoes Bigtable's tablet-on-GFS design; contrast the leaderless-vs-single-master lineage
- [014-rum-conjecture.md](014-rum-conjecture.md) — Dynamo's tunable N/R/W knobs are a live RUM-style tradeoff dial; DynamoDB later decided those knobs weren't worth exposing

> **Note on scope:** This is a single, exhaustive note covering the whole arc — the 2007 Dynamo paper, why DynamoDB deliberately diverged from it, DynamoDB's actual internals (as of the 2022 paper), and the lessons Brooker draws out. It's long by design: read it top to bottom and you should understand how DynamoDB works and *why* every piece is the way it is.

---

## The Three Names Problem

Three Amazon systems share confusingly similar names, and untangling them is the first step to understanding any of them (Brooker's 2025 post exists mostly to do this):

- **Dynamo** — a 2007 research paper describing a *leaderless, eventually-consistent* key-value store Amazon built for internal services like the shopping cart. It was never a public product.
- **DynamoDB** — the *public AWS product*, launched 2012. It "shared most of the name of the previous Dynamo system but little of its architecture." It is *not* leaderless and *not* (by default) eventually consistent.
- **Aurora DSQL** — a newer serverless distributed *SQL* database, architecturally distinct from both. (Out of scope here; mentioned only where it sharpens a contrast.)

The single most important thing to internalize up front: **DynamoDB is not Dynamo.** It took Dynamo's *goals* (incremental scalability, predictable performance, always-on availability) and Dynamo's *name*, but threw away most of Dynamo's *mechanisms* (leaderless quorums, vector clocks, sloppy quorums) in favor of nearly the opposite design (single-leader-per-partition via Paxos, strong consistency available). Understanding *why* that reversal happened is the spine of this note. So we start where it started: 2007.

---

# Part 1 — Dynamo (2007): The Always-Writeable Store

## The Problem Dynamo Solved

Amazon in the mid-2000s ran hundreds of services on tens of thousands of machines across many datacenters, and had learned a brutal operational truth the paper states plainly: at that scale, "small and large components fail continuously," so failure handling must be "the normal case." The motivating example, repeated throughout the paper, is the **shopping cart**: a customer must *always* be able to add or remove items — "even if disks are failing, network routes are flapping, or data centers are being destroyed by tornados." Rejecting a write because a replica is unreachable is an unacceptable lost sale and a broken customer experience.

Traditional relational databases couldn't deliver this. They favored *consistency* over *availability* — when uncertain (say, during a network partition), they'd rather make data unavailable than risk returning a wrong answer. Dynamo's designers made the opposite bet, grounded in a principle they took as given (citing the CAP intuition): with network partitions inevitable, "strong consistency and high data availability cannot be achieved simultaneously." So Dynamo chose availability. Its defining goal: an **"always writeable" data store** where "no updates are rejected due to failures or concurrent writes."

Two design consequences flow directly from "always writeable," and they shape everything:

1. **Conflicts must be resolved on reads, not writes.** Most databases resolve conflicts at write time (reject a write that can't reach a quorum) to keep reads simple. Dynamo can't — rejecting writes is forbidden. So it pushes the complexity to reads: accept every write, allow multiple conflicting versions to coexist, and reconcile them when someone reads.
2. **The application often resolves conflicts, not the database.** A database resolving conflicts alone can only use dumb rules like "last write wins" (which loses data). But the *application* knows the semantics — the shopping-cart service can *merge* two divergent carts (union the items) so an "add to cart" is never lost. Dynamo exposes this choice to the app.

Two more requirements set the tone. Everything is measured at the **99.9th percentile** of latency, not the average — Amazon's "relentless focus on performance from the perspective of the customer's experience," because tail latency is what actually breaks user-facing pages (a single page could fan out to 150+ services, so every dependency needs a tight latency contract, an **SLA**). And Dynamo must be a **zero-hop DHT**: unlike Chord/Pastry which route requests through multiple nodes (adding latency variance that kills the 99.9th percentile), every Dynamo node knows enough to route directly to the right node in one hop.

## The Interface: get and put

Dynamo's API is deliberately minimal — just two operations on opaque blobs (< 1 MB), identified by a key that's MD5-hashed to a 128-bit id:

```
   get(key)                → returns the object (or MULTIPLE conflicting versions)
                             plus a "context"
   put(key, context, obj)  → writes a new version; the context (from a prior read)
                             tells Dynamo which version this write is based on
```

The **context** is the clever part — it's opaque system metadata (carrying the version/vector-clock info) that the client got from a read and hands back on the next write. This is how Dynamo tracks causality (which write descends from which), and it's exactly the "read-return-context, write-with-context" protocol from the version-vectors discussion (Entry #028).

## The Techniques (and What Each Solves)

The paper's own summary table pairs each problem with its technique. This is the heart of Dynamo, so we take them one at a time.

```
   Problem                          Technique                        Why
   ──────────────────────────────   ──────────────────────────────  ──────────────────────
   Partitioning                     Consistent hashing               incremental scalability
   High availability for writes     Vector clocks + reconcile-on-read decouple version size
                                                                      from update rate
   Handling temporary failures      Sloppy quorum + hinted handoff   availability + durability
                                                                      when replicas are down
   Recovering from permanent        Anti-entropy with Merkle trees   sync divergent replicas
     failures                                                         in the background
   Membership & failure detection   Gossip-based protocol            symmetry, no central registry
```

### Consistent Hashing + Virtual Nodes (partitioning)

Data must spread across nodes and rebalance smoothly as nodes come and go. Dynamo uses **consistent hashing**: the hash output is a fixed circular "ring." Each node gets a random position on the ring; a key is placed by hashing it to a point and walking *clockwise* to the first node. A node owns the ring segment between it and its predecessor. The beauty: adding or removing a node only affects its *immediate neighbors*, not the whole ring.

```
        (hash ring, clockwise)
              A
          G       B ← key K hashes here, walks clockwise → lands on B
        F           C     so B is K's coordinator; C, D are the next replicas
          E       D
```

Two problems with naive consistent hashing — random positions give *uneven* load, and it ignores that some machines are more powerful. Fix: **virtual nodes.** Each physical node is assigned *many* positions ("tokens") on the ring rather than one. This makes load distribution even (a failed node's load disperses across *many* neighbors, not one), lets a recovered/new node absorb a little load from everyone, and lets a beefier machine hold *more* virtual nodes — accounting for hardware heterogeneity.

### Replication + Preference Lists

Each key is replicated on **N** nodes (typically N=3). The coordinator (the key's first node) stores it locally and replicates to the **N-1 clockwise successors**. The list of nodes responsible for a key is its **preference list** — and it deliberately contains *more* than N nodes (to have healthy fallbacks) and skips positions so the list has N *distinct physical* nodes (virtual nodes could otherwise put two replicas on one machine). The preference list is also arranged to span *multiple datacenters*, so Dynamo survives an entire datacenter loss.

### Vector Clocks + Reconciliation (versioning)

This is the mechanism that makes "always writeable + resolve on read" work. Because writes propagate asynchronously, and failures + concurrent writes can create genuinely divergent versions, Dynamo needs to tell *"this version supersedes that one"* apart from *"these two versions conflict."* It uses **vector clocks** — a list of `(node, counter)` pairs, one clock per version (Entry #028 covers the mechanism in full).

The rule: comparing two versions' vector clocks, if every counter in clock A is ≤ the corresponding counter in clock B, then A is an *ancestor* of B and can be discarded (B descends from A). Otherwise the versions are *concurrent* — a real conflict that must be reconciled.

```
   D1 [(Sx,1)]                          client writes; node Sx
      │
   D2 [(Sx,2)]                          same client updates; D2 descends D1 → D1 discardable
      ├───────────────┐
   D3 [(Sx,2),(Sy,1)]  D4 [(Sx,2),(Sz,1)]    two nodes handle concurrent updates
      └───────┬────────┘                       D3 ∥ D4: neither's clock dominates → CONFLICT
   D5 [(Sx,3),(Sy,1),(Sz,1)]            client reads BOTH, merges, writes back → D5 subsumes both
```

The **shopping cart** is the canonical use: concurrent "add to cart" writes create sibling versions; on the next read the app receives all siblings and *merges* them (unions the items), so no "add" is ever lost. (The paper is honest about the downside: since deletes are also just writes, a merge can cause **deleted items to resurface** — a real, known quirk of the model.) Vector clocks can grow if many nodes coordinate writes to one key; Dynamo truncates the oldest `(node, counter)` pair past a threshold (~10), accepting rare reconciliation inaccuracy.

Applications that don't want to write merge logic can fall back to **last-write-wins** (the session-state service does this) — simpler, but it discards concurrent writes.

### Sloppy Quorum + Hinted Handoff (temporary failures)

Dynamo uses quorum-style **R** and **W** (from Entry #027): a read waits for R responses, a write for W acknowledgments, and `R + W > N` gives read/write overlap. But a *strict* quorum would make Dynamo unavailable whenever nodes are down — violating "always writeable." So Dynamo uses a **sloppy quorum**: reads and writes go to the first N *healthy* nodes from the preference list, which "may not always be the first N nodes encountered while walking the ring."

**Hinted handoff** makes this work. If the intended node (say A) is down, the write goes to another reachable node (say D) instead, tagged with a **hint** saying "this really belongs to A." D stores it in a separate local area and, when A recovers, hands it off and deletes its copy. This means a write succeeds as long as *any* N healthy nodes exist — set W=1 and a write survives if a *single* node is up. (Entry #027 covers the operational pain of hinted handoff, Colin Breck's "inverted load-shedding" critique — that's the practitioner counterpoint to this mechanism.)

### Anti-Entropy with Merkle Trees (permanent failures)

Hinted handoff can fail (the hint-holder itself dies before handoff). The safety net is **anti-entropy**: replicas periodically compare full datasets and copy over differences. To do this *efficiently* — without shipping everything — Dynamo uses **Merkle trees**: a hash tree where leaves hash individual keys and parents hash their children. Two replicas compare roots; if equal, they're identical and done. If not, they descend only the differing branches, exchanging hashes until they pin down exactly which keys are out of sync. This minimizes both data transferred and disk reads. (The weakness: when nodes join/leave, key ranges shift and trees must be recomputed — a problem Dynamo's refined partitioning later addressed.)

### Gossip Membership + Failure Detection

Consistent with its symmetry/decentralization principles (no special nodes, no central registry), Dynamo spreads membership via a **gossip protocol**: each node contacts a random peer every second and they reconcile their membership histories, converging to an eventually-consistent view. Membership changes are *explicit* (an admin adds/removes a node) — a node outage is treated as transient and does *not* trigger rebalancing, because outages are usually temporary. **Seed nodes** (known to all) prevent logical partitions where two nodes each think they're alone. **Failure detection is purely local**: node A considers B failed if B doesn't answer A's requests, and just routes around it — no global agreement on who's dead is needed.

## Dynamo's Results and Honest Limits

Dynamo delivered: it scaled through Amazon's holiday peak with no downtime, served tens of millions of cart requests, and let each service tune N/R/W for its needs (R=1, W=N for read-heavy catalog data; W=1 for maximum write availability). The main research contribution, in the authors' words, was demonstrating "that an eventually-consistent storage system can be used in production with demanding applications."

But — and this matters for the rest of the story — Dynamo also carried heavy costs that its *own users* felt:

- **Eventual consistency is hard for application developers.** Every reader must be prepared to receive multiple conflicting versions and merge them. Most developers don't want to write merge logic.
- **It was single-tenant and self-managed.** Each team ran its *own* Dynamo installation and had to become experts in consistent hashing, quorum tuning, and gossip. As the 2022 paper says bluntly, "the resulting operational complexity became a barrier to adoption." Amazon engineers *preferred* the managed simplicity of S3 and SimpleDB even when Dynamo fit their needs better.

That tension — Dynamo's power vs its operational burden — is exactly what the product DynamoDB was created to resolve. Which requires understanding one system in between.

---

# Part 2 — The Pivot: Why DynamoDB Is Not Dynamo

Between Dynamo (2007) and DynamoDB (2012) came **SimpleDB** — Amazon's *first* database-as-a-service. It was fully managed and elastic (no setup, patching, or configuration), which developers loved. But it had crippling limits: tiny tables (10 GB), *unpredictable* latency (it indexed *all* attributes, so every write updated every index), and a restricted query model. Developers ended up splitting data across many tables to work around the limits — a new operational burden.

So Amazon had two systems, each with half the answer:

```
   DYNAMO gave:                          SIMPLEDB gave:
   • incremental scalability             • ease of a managed cloud service
   • predictable high performance        • a richer table data model
   • always-on availability              • consistency
   BUT: self-managed, single-tenant,     BUT: tiny capacity, unpredictable
        eventually consistent, hard            latency, restrictive queries
```

**DynamoDB (2012) was the deliberate synthesis:** take Dynamo's scalability + predictable performance, and SimpleDB's managed-cloud ease + richer table model + consistency. The 2022 paper states this explicitly. And crucially, achieving *predictable performance* and *strong consistency as an option* meant abandoning Dynamo's leaderless, eventually-consistent architecture. DynamoDB kept the *hash-based partitioning* idea and the *name*, and reversed almost everything else. This is Brooker's central point: Dynamo's foundational assumptions "haven't aged well" — the mid-2000s belief that ACID/strong-consistency systems inherently have poor availability was true *then* but isn't now (better hardware, networking, and replication understanding let DynamoDB deliver strong consistency *and* high availability *and* low latency, tolerating host and even whole-AZ failures).

---

# Part 3 — DynamoDB Internals (the 2022 paper)

Now the actual product, as it works today. DynamoDB's guarantee is **consistent, predictable single-digit-millisecond latency at any scale** — latency that stays flat whether a table is megabytes or hundreds of terabytes, whether traffic is 100 ops/sec or (as on Prime Day 2021) **89.2 million requests/second**. It is fully managed, multi-tenant (many customers' data on shared machines for efficiency, with isolation), and offers **99.99%** availability for regional tables, **99.999%** for global tables. Everything below serves that predictability promise.

## The Data Model and Partitions

A **table** is a collection of **items** (each a bag of attributes, no fixed schema), each identified by a **primary key** — either a partition key alone, or a partition key + sort key (composite). The partition key is *hashed*; the hash (plus sort key) determines where the item lives.

A table is split into **partitions**, each owning a contiguous, disjoint slice of the key range. Partitions are *the* central abstraction — they're how DynamoDB scales both storage and throughput, by splitting and migrating them across a fleet of storage nodes. Each partition is replicated for durability into a **replication group**.

## The Replication Group: Single-Leader via Multi-Paxos

Here is the sharpest break from Dynamo. Each partition's replicas form a **replication group** that runs **Multi-Paxos** for leader election and consensus. This is *single-leader replication* (Entry #016) at the per-partition level — the exact opposite of Dynamo's leaderless design:

```
   A partition's replication group (spread across 3 Availability Zones):

          ┌──────────────┐   Multi-Paxos leader election
          │ LEADER (AZ1) │◄──── only the leader serves WRITES
          └──────┬───────┘      and STRONGLY CONSISTENT reads
       WAL record│ replicated to peers
          ┌──────┴───────┬──────────────┐
          ▼              ▼               ▼
   ┌────────────┐ ┌────────────┐  ┌────────────┐
   │ replica AZ2│ │ replica AZ3│  │ log replica│
   └────────────┘ └────────────┘  └────────────┘
   any replica serves EVENTUALLY consistent reads
```

- **Only the leader serves writes and strongly consistent reads.** On a write, the leader creates a **write-ahead log (WAL) record** and sends it to peers; the write is acknowledged once a **quorum** persists the log record. This gives *strong consistency* — a total order of writes per key — with **no vector clocks, no sibling versions, no application-side conflict merging.** The complexity Dynamo pushed onto every application developer simply vanishes.
- **Any replica serves eventually consistent reads** (cheaper, possibly slightly stale) — so DynamoDB lets the *reader* choose strong or eventual per request, rather than baking eventual consistency into the whole system.
- **The leader holds a lease.** It stays leader by periodically renewing a time-based lease; if it's suspected dead, a peer proposes a new election but *cannot serve* until the old leader's lease expires (preventing two leaders — the split-brain problem of Entry #018).

### Storage Replicas vs Log Replicas (a key trick)

A normal **storage replica** holds *both* the write-ahead log *and* a **B-tree** storing the actual key-value data (echoing Bigtable's tablet design, Entry #009). But DynamoDB adds a lightweight variant: the **log replica**, which holds *only* recent WAL entries — no B-tree, like a Paxos acceptor.

Why this matters enormously: when a storage replica fails, the group drops to two copies, and *healing a full storage replica takes minutes* (you must copy the whole B-tree + logs). But **adding a log replica takes only seconds** (copy just the recent logs). So the moment a replica looks unhealthy, the leader instantly adds a log replica to restore the write quorum's durability, buying time to heal the full replica in the background. This one idea — cheap, fast, partial replicas — is central to DynamoDB's durability *and* availability, and the formally-proven Paxos implementation gave the team confidence to introduce it safely.

## Request Routing and Metadata: The MemDS Story

How does a request find the right partition's leader? Three services cooperate (plus the **autoadmin** control plane, "the central nervous system," which handles health, scaling, and repairs):

```
   Client ──► Request Router ──► looks up routing info in Metadata ──► Storage Node (leader)
                                  (which partition? which nodes?)
```

The **request router** needs one piece of metadata: the mapping from a key to the storage nodes hosting its partition. The story of how DynamoDB stores that mapping is one of the paper's best lessons — and Brooker devotes his whole 2022 post to it, because it's a masterclass in **avoiding metastable failures**.

**The original design (and its trap):** routers cached a table's full routing info on first access. Since partition maps rarely change, the cache hit rate was ~99.75% — which *sounds* excellent. But it created **bimodal behavior**, a latent catastrophe:

```
   Normal:      99.75% cache hits → metadata service handles only 0.25% of traffic
   Cold caches: 0% hits (e.g., a fleet of new routers deployed, or a restart)
                → EVERY request hits the metadata service
                → it must suddenly handle ~400× its normal load
                → it can't → cascading failure
```

A system that runs at 0.25% load normally but needs 100% capacity in a cold-start is a **metastable failure** waiting to happen — a small trigger (adding routers) flips it into a state it can't recover from. In practice, adding new routers spiked metadata traffic up to 75%.

**The fix — MemDS (and the deeper principle):** DynamoDB built an in-memory distributed metadata store, **MemDS**, that holds all metadata compressed in RAM and scales horizontally to absorb DynamoDB's *entire* request rate. (Internally it uses a "Perkle" tree — a Patricia + Merkle hybrid — supporting prefix and range lookups.) The genius is in how the router cache interacts with it: **a cache hit *still* triggers an asynchronous call to MemDS to refresh.** That sounds wasteful, but it means MemDS always receives a *constant* volume of traffic regardless of cache hit rate — so there's no bimodality, no 400× cold-start spike, no cliff.

This embodies the note's biggest lesson (Brooker's theme): **design for predictability over efficiency.** A cache that *hides* work is dangerous because when it fails, the hidden work reappears all at once. DynamoDB deliberately does *more* constant work (always hitting MemDS) to guarantee the system is *always* provisioned for the load it would face without the cache. "Do not allow [caches] to hide the work that would be performed in their absence." This is the **constant-work** principle, and it's arguably the single most transferable idea in the whole DynamoDB story.

## The Throughput Journey: Provisioned → On-Demand

DynamoDB's other great arc is how it manages *throughput* — and it's a chain of "our assumption was wrong, here's the fix" stories. This is where the value of a managed service is most visible: hard-won lessons baked in so customers never hit them.

**Provisioned throughput (the launch model).** Customers specified capacity in **RCUs** (read capacity units) and **WCUs** (write capacity units). Each partition got a static slice of the table's capacity, enforced by **admission control** — token buckets on each storage node (per-partition allocated + burst buckets, plus a node-level bucket) throttling requests beyond the allocation. Admission control was *local* to each storage node, which nicely avoided distributed-coordination complexity.

**The wrong assumption.** The whole scheme assumed applications access keys *uniformly*, so splitting a partition splits its load evenly. But real workloads are **non-uniform** — traffic concentrates on some keys. Two painful failure modes resulted:

```
   HOT PARTITION:      traffic hammers a few items → that partition hits its
                        allocation and THROTTLES, even though the table overall
                        has plenty of unused capacity elsewhere.

   THROUGHPUT DILUTION: split a partition for SIZE → its throughput is halved
                        into each child → each child now has LESS capacity than
                        the parent → throttling, again despite adequate table total.
```

From the customer's view, this was **unavailability** even though DynamoDB was "working as designed." Their workaround — massively *over-provision* — was expensive and hard to estimate. So DynamoDB evolved through four fixes:

1. **Bursting.** Observation: not all partitions use their allocation at once. So let a partition temporarily borrow *unused* capacity (its own, saved up to 300 s, gated by node-level capacity) to absorb short spikes. Short-lived relief only.
2. **Adaptive capacity.** For *longer* skew: monitor tables, and if one throttles while under its table-level total, automatically *boost* the hot partitions' allocation (relocating them to nodes with room). This eliminated 99.99% of skew-throttling — but was **reactive** (kicked in only *after* throttling was observed, so customers still saw brief unavailability).
3. **Global Admission Control (GAC).** The real fix: decouple admission control from static per-partition allocation. GAC is a service that centrally tracks a *table's* total token consumption; each request router keeps a local token bucket and tops up from GAC every few seconds. Now a non-uniform workload can drive traffic to a subset of items up to the *partition's* max capacity, drawing on the whole table's budget — no artificial per-partition ceiling. (Per-partition buckets are *retained* as defense-in-depth, capped so one tenant can't monopolize a node.)
4. **Splitting for consumption.** With GAC letting partitions burst freely, DynamoDB also *splits partitions based on consumed throughput* (not size), choosing the split point from the *observed key distribution* (a better proxy for the access pattern than splitting the range in the middle). It detects the un-splittable cases too — a single hot item, or a sequentially-accessed range — and avoids futile splits.

**On-demand (the destination).** Finally, DynamoDB removed capacity planning entirely. **On-demand tables** watch actual consumption and provision automatically — instantly accommodating up to *double* the previous peak, and scaling further as traffic grows, by splitting partitions for consumption. GAC protects the system from any one app consuming everything, and consumption-based balancing places partitions to avoid node limits. The customer just uses the table; DynamoDB figures out the capacity. This is the culmination of the whole journey: the operational burden that made *Dynamo* hard to adopt is now fully absorbed by the *service*.

There's a broader colocation challenge behind all this: since a storage node hosts thousands of unrelated partitions from different customers, DynamoDB must **proactively rebalance** partitions across nodes based on throughput and storage, so that always-bursting partitions don't collectively overload a node (each node monitors its own load and asks autoadmin to move replicas off when it's over threshold).

## Durability: Never Lose a Committed Write

DynamoDB's durability story is defense-in-depth, layered against three enemies: hardware failures, silent data corruption, and software bugs.

**Write-ahead logs, everywhere.** WALs are the foundation. Each partition's WAL lives in all three replicas *and* is periodically **archived to S3** (which itself has 11 nines of durability). Each replica keeps only the most recent (few-hundred-MB) unarchived logs. Combined with the log-replica trick above, this means recent writes are extremely well protected.

**Checksums on everything.** Silent data errors (bad storage media, CPU, memory) can corrupt data anywhere and are hard to detect. DynamoDB "makes extensive use of checksums" — every log entry, every message between nodes, every log file carries a checksum that's verified at each hop, so corruption can't silently propagate. Log files archived to S3 carry a manifest (table, partition, sequence markers) that's validated for gaps and correctness before archival.

**Continuous verification (scrub).** The most striking durability mechanism: DynamoDB continuously verifies *data at rest*. A **scrub** process checks (1) that all three replicas of a group hold identical data, and (2) that the live data matches a replica *rebuilt independently from the archived WAL* from the table's inception. If the live storage and the log-derived replica ever diverge, scrub catches it. This "defense in depth" against bit rot and unanticipated errors is, the paper says, "the most reliable method of protecting against hardware failures, silent data corruption, and even software bugs."

**Formal methods against software bugs.** The core replication protocol is specified in **TLA+** and model-checked; new features affecting replication are added to the spec and re-checked, catching "subtle bugs that could have led to durability and correctness issues before the code went into production." (Marc Brooker is an author on the formal-methods work — this is a recurring AWS practice, also used by S3.) Beyond the data plane, formal methods verify the control plane and distributed transactions too.

**Backups and point-in-time restore.** Against *logical* corruption (a bug in the *customer's* app), DynamoDB offers backups (built from the S3-archived WALs, so zero performance impact, consistent across partitions to the nearest second) and **point-in-time restore** to any second in the last 35 days (via periodic partition snapshots + WAL replay).

## Availability: Staying Up Through Everything

Availability rests on the replication group being spread across **three AZs**, so it survives node, rack, and even whole-AZ failure (regularly tested with power-off chaos experiments). A partition stays *writeable* as long as it has a leader and a healthy **write quorum** (two of three replicas across AZs). If a replica goes unhealthy, the leader instantly adds a **log replica** to restore the quorum — the fastest possible mend (seconds, not minutes).

**Gray failures and smarter failure detection.** The subtle availability killer is the **gray failure** (Entry #027): a node isn't cleanly dead, but a *link* between a specific follower and the leader is degraded. That follower stops hearing heartbeats and tries to trigger a *new election* — even though the leader is fine and serving everyone else. Spurious elections hurt availability (the new leader must wait out the old lease). DynamoDB's fix is elegant: before a follower triggers failover, **it asks the other replicas "can you still talk to the leader?"** If they say yes, it stands down. This one change slashed false-positive elections.

**Deployments without downtime.** DynamoDB deploys continuously with no maintenance windows. The hard part in a distributed system: deployments aren't atomic, so old and new code run simultaneously, and new code might send messages old code can't parse. The solution is **read-write deployment** — a two-phase rollout: first deploy software that can *read* the new message format (but still sends the old); once the whole fleet can read it, deploy software that *sends* it. This guarantees old and new coexist safely, including during rollbacks (which are explicitly tested — the rolled-back state can differ from the original, a case "often missed in testing"). Deployments go to small node sets first, with automatic rollback if availability alarms trip.

**Static stability against dependencies.** DynamoDB depends on IAM and KMS for auth on every request — but those must never be able to take DynamoDB down. So DynamoDB uses **static stability** (Entry-adjacent AWS principle): it caches IAM/KMS results in the request routers and refreshes asynchronously; if IAM or KMS become *unavailable*, the routers keep using cached results for an extended period, so "everything before the dependency became impaired continues to work." The system is designed to keep running on stale-but-safe cached auth rather than fail when a dependency does.

---

# Part 4 — The Lessons (What to Actually Remember)

Both papers and Brooker converge on a set of transferable principles. These are the *why-it-matters* takeaways that outlast any specific mechanism:

**1. Predictability over efficiency.** The recurring theme. A system that's *usually* fast but occasionally catastrophic is worse than one that's *always* predictable. Flat, boring latency at any scale is DynamoDB's product. Caches, adaptive tricks, and clever optimizations are dangerous exactly when they *hide* work that reappears under stress.

**2. Beware caches and metastability.** The MemDS story is the cautionary tale: a 99.75%-hit cache is a *bimodal bomb*. The fix — **constant work** (always hit MemDS regardless of cache state) — trades a little steady overhead for immunity to cascading, metastable failure. When you add a cache, ask: "what happens when it's empty?" If the answer is "the backend gets 400× load," you've built a metastable failure.

**3. Admission control is subtle and workloads are never uniform.** The entire provisioned→on-demand journey is one long lesson that "assume uniform access" is *always* wrong, and that coupling a rigid resource allocation to a partition creates hot-partition and dilution failures. The evolution toward global admission control + consumption-based splitting is how you decouple them.

**4. Durability is layered and *verified*, not assumed.** WALs + S3 archival + checksums-on-everything + **continuous scrub against log-rebuilt replicas** + formal methods. Notably, DynamoDB doesn't *trust* that data is intact — it *continuously proves* it by rebuilding from an independent source and comparing.

**5. Formal methods pay off at scale.** TLA+ model-checking of the replication protocol caught subtle bugs pre-production and — just as importantly — gave the team *confidence to safely change* a running system that millions depend on (e.g., introducing log replicas).

**6. The value of "managed" is embedded operational wisdom.** Every failure mode above — hot partitions, metadata bimodality, gray-failure elections, silent corruption — is a lesson learned the hard way and now baked into the service so customers never relearn it. That embedded wisdom, not just the code, is what you buy.

---

# Part 5 — The Lineage: Dynamo vs DynamoDB (vs DSQL)

To close the loop, here's Brooker's side-by-side of how the *same-named* systems actually differ — the clearest summary of "DynamoDB is not Dynamo":

| Property | **Dynamo (2007)** | **DynamoDB (product)** |
|---|---|---|
| Leadership | **Leaderless** — any node coordinates | **Single-leader per partition** (Multi-Paxos) |
| Consistency | **Eventual only** (tunable N/R/W) | **Strong or eventual**, reader's choice |
| Conflict handling | Vector clocks + **app-side merge** (siblings) | Total order per key — **no conflicts, no merging** |
| Durability | N replicas placed on ring successors | Replication group across 3 AZs; write = quorum persists WAL; WAL → S3 |
| Scaling | Add nodes to the ring (churn recomputes Merkle trees) | Split/merge/migrate **partitions**; no durability drop while scaling |
| Programming model | Key-value, single-key, no isolation | Rich items, secondary indexes, **ACID transactions** |
| Operations | **Self-managed, single-tenant** | **Fully managed, multi-tenant, serverless** |

Brooker's meta-point: several of Dynamo's 2007 assumptions were true *then* but are obsolete now. The claim that ACID/strong-consistency systems "tend to have poor availability" no longer holds (DynamoDB, DSQL, and Aurora Postgres all offer strong availability tolerating host or full-AZ failures). Dynamo's tunable N/R/W performance knobs — a real RUM-style tradeoff dial (Entry #014) — made sense on 2000s hardware but have become "uninteresting" thanks to SSDs, better networks, and better replication techniques; DynamoDB delivers *lower* latency *without* exposing those knobs. (And DSQL goes further still — physical-time + MVCC for strong consistency on *all* reads/writes including interactive SQL transactions, with reads never blocking writers — but that's another note.)

The arc, then, is one of *progress*: Dynamo proved eventual consistency could work in production and pioneered the techniques (consistent hashing, quorums, hinted handoff, anti-entropy) that Entry #027 catalogues. DynamoDB kept the *goals* and the *name*, learned that better engineering could deliver strong consistency, predictable latency, *and* availability together, and wrapped it all in a managed service that absorbs the operational complexity that made Dynamo itself hard to live with. The paper's classic status is well-earned — but, as Brooker says, "much of it no longer reflects modern reality," and that's the good news.

---

## Key Takeaways

- **Three systems, one confusing name family:** *Dynamo* (2007 leaderless eventually-consistent research system), *DynamoDB* (2012 managed product that kept the name but reversed the architecture), *DSQL* (a distinct SQL system). **DynamoDB is not Dynamo.**
- **Dynamo (2007)** chose *availability over consistency* to be **"always writeable"** for the shopping cart. Its techniques: **consistent hashing + virtual nodes** (partitioning), **N replicas + preference lists**, **vector clocks + reconcile-on-read** with app-side merge (the siblings model, Entry #028), **sloppy quorum + hinted handoff** (temporary failures), **Merkle-tree anti-entropy** (permanent failures), and **gossip membership + local failure detection**. Tunable **N/R/W** with `R+W>N`. Its cost: hard for developers (merge logic), and self-managed/single-tenant — a barrier to adoption.
- **DynamoDB (2012)** synthesized Dynamo's scalability + SimpleDB's managed ease, and to get **predictable low latency + strong consistency** it went **single-leader-per-partition via Multi-Paxos** (Entry #016) — the opposite of Dynamo. Only the leader serves writes and strong reads (WAL replicated to a quorum); any replica serves eventual reads. **No vector clocks, no siblings, no app-side merging.**
- **Log replicas** (WAL only, no B-tree) can be added in *seconds* to restore a write quorum's durability, vs *minutes* to heal a full storage replica — central to both durability and availability.
- **MemDS + constant work:** the original 99.75%-hit routing cache was a **metastable bomb** (cold start → ~400× metadata load → cascading failure). The fix: a cache hit *still* asynchronously refreshes from MemDS, so metadata traffic is *constant* regardless of hit rate. Lesson: **predictability over efficiency; never let a cache hide work it can't reappear to do.**
- **Throughput journey:** provisioned (static per-partition, hit **hot partitions** and **throughput dilution** because access is never uniform) → **bursting** (short spikes) → **adaptive capacity** (reactive boost) → **Global Admission Control** (table-level tokens, decoupled from partitions) → **splitting for consumption** → **on-demand** (no capacity planning at all).
- **Durability is layered and *verified*:** WALs in all replicas + archived to S3 (11 nines) + checksums on every entry/message/file + **continuous scrub** (compare live data against a replica rebuilt from archived logs) + **TLA+** model-checking + backups/PITR.
- **Availability:** 3-AZ replication groups, write quorum = 2 of 3, instant log-replica healing, **gray-failure-aware failover** (ask peers before triggering an election), **read-write (two-phase) deployments**, and **static stability** against IAM/KMS via cached auth.
- **The meta-lesson:** the value of a managed database is the decade of embedded operational wisdom — every failure mode learned once, so customers never relearn it. Dynamo's assumptions ("ACID means poor availability"; tunable N/R/W matter) were true in 2007 but obsolete now — the whole arc is a story of progress.
