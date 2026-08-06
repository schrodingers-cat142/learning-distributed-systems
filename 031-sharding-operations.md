# Operating a Sharded System: Rebalancing, Routing, and Secondary Indexes

**Date:** 2026-08-06
**Sources:**
- DDIA Chapter 6 (Sharding/Partitioning) — rebalancing, request routing, secondary indexes
- [ScyllaDB's Safe Topology and Schema Changes on Raft (2024)](https://www.scylladb.com/2024/06/18/scylladbs-safe-topology-and-schema-changes-on-raft/)
- [Sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion) — the migration mechanics
- [Things Databases Don't Do… But Should — Nile](https://www.thenile.dev/blog/things-dbs-dont-do)

**Related entries:**
- [030-sharding-strategies.md](030-sharding-strategies.md) — the design half: *how to divide* the data. This note is the operational half: once divided, *how to run it*
- [018-node-outages-failover-split-brain.md](018-node-outages-failover-split-brain.md) — the "automatic vs manual" tension here mirrors failover's: automation is a loaded gun, and split-brain is why coordination needs consensus
- [028-time-clocks-concurrent-writes.md](028-time-clocks-concurrent-writes.md) — ScyllaDB's move to Raft for metadata is "use a linearizable coordinator" — the strong-consistency answer to the concurrency problems there
- [017-replication-logs.md](017-replication-logs.md) — Notion's migration used change-capture/audit-logs (CDC-style) to keep old and new in sync during the move
- [013-vector-search-ann.md](013-vector-search-ann.md) — global secondary indexes over shards are "scatter-gather," the same fan-out-and-merge pattern as ANN queries there

> **Note on scope:** Second of two sharding notes. Entry #030 covered *designing* the sharding scheme (schemes, consistent hashing, multitenancy, hot spots). This note covers *operating* the sharded system: how shards move between nodes (rebalancing), how requests find the right shard (routing + coordination services), and how you query by something other than the shard key (secondary indexes).

---

## The Problem This Note Solves

Entry #030 ended with a working sharding *scheme* — you can compute which shard any key belongs to. But a scheme on paper isn't a running system. Three operational questions remain, and each is genuinely hard:

```
   1. REBALANCING — nodes get added, removed, or die. Shards must move
      between nodes. How, and who decides — automatically or by a human?

   2. ROUTING     — a client has a request. Which node holds the shard it
      needs? How does the client (or the system) find out?

   3. SECONDARY INDEXES — the shard key answers "find record by key." But
      "find all records where color = red" doesn't align with the shard key.
      How do you query by a non-key attribute across shards?
```

All three share a theme that ties this note to the failover material (Entry #018): **the system's *metadata* — who owns which shard — is itself distributed state that must stay consistent, or everything breaks.** Get the metadata wrong and requests go to the wrong node, two nodes both think they own a shard (split-brain for data), or a rebalance corrupts data. So a recurring answer will be: a strongly-consistent coordination service.

---

## Rebalancing: Moving Shards Between Nodes

Over time, the assignment of shards to nodes must change — you add a node for capacity, a node dies, or one node gets hot and needs relief. **Rebalancing** is the process of moving shards (and their data) between nodes to restore balance. Two requirements make it delicate:

- It should move **as little data as possible** (moving data is slow and consumes bandwidth that competes with serving requests).
- The system should **keep serving reads and writes** *during* the rebalance — you can't take an outage every time you add a node.

Entry #030 already covered *which* scheme makes rebalancing cheap: the **mod-N trap** (never use it — it reshuffles everything), the **fixed-number-of-shards** approach (move whole shards, never recompute the key→shard map), and **hash-range** (split a hash range in two). Those are the *mechanics*. The harder question is *who triggers a rebalance, and how much do you trust automation.*

### Automatic vs Manual Rebalancing

This is a direct echo of the failover dilemma (Entry #018): full automation is convenient but dangerous.

```
   FULLY AUTOMATIC                       FULLY MANUAL
   ───────────────                       ────────────
   system detects imbalance and          a human decides when to rebalance
   moves shards on its own               and the system executes it
   + no human in the loop                + human judgment prevents cascades
   − DANGEROUS: can misread load and     − slow; needs an operator; doesn't
     trigger a rebalance at the worst      react to sudden failures on its own
     moment, cascading the problem
```

DDIA's warning is emphatic and worth internalizing: **fully automatic rebalancing is a loaded gun.** The failure mode is a feedback loop. Suppose a node is slow because it's overloaded. An automatic rebalancer sees the overload and decides to *move shards off it* — but moving data is itself expensive and consumes the node's already-scarce resources, making it *slower*, which looks like *more* overload, triggering *more* rebalancing... a cascade. Worse, automatic rebalancing combined with automatic failure detection is doubly dangerous: a node that's merely slow (a gray failure, Entry #027) can be mistaken for dead, triggering a needless, expensive data migration that degrades the whole cluster.

The pragmatic middle ground most systems adopt: **the system computes a proposed rebalancing plan automatically, but a human approves it before it executes.** You get the automation's analysis without letting it pull the trigger during an incident. This is the same "keep a human in the loop for the dangerous action" conclusion as failover (Entry #018, where GitHub made failover manual-only after an automated one caused an outage).

### The Worked Example: Notion's Migration

Notion's Postgres sharding (Entry #030 covered *why* and *what* — workspace ID, 480 logical shards) is the best real-world illustration of *executing* a large rebalance — here, the biggest rebalance of all: moving from *one* shard (the monolith) to 480, with zero data loss and near-zero downtime. The framework is worth memorizing because it's how *any* live data migration works:

```
   DOUBLE-WRITE  →  BACKFILL  →  VERIFY  →  CUTOVER
```

1. **Double-write.** Start writing every new change to *both* the old monolith *and* the new sharded cluster simultaneously, so the new cluster stays current from this moment on. Notion built this with an **audit log + catch-up script** rather than Postgres logical replication (which "struggled to keep up with `block` table write volume") or direct dual-writes from the app (too flaky). They also built a *reverse* audit log so they could roll back to the monolith if needed (they didn't have to). This is a CDC-flavored technique (Entry #017).
2. **Backfill.** Copy all the *existing* (pre-double-write) data into the new cluster in the background. Notion ran this on a 96-CPU `m5.24xlarge`, and it "took around three days." The backfill compared record versions and skipped rows already newer in the destination, so it could coexist with the live double-writes without clobbering fresher data.
3. **Verify.** Confirm old and new agree. A full scan was too expensive, so they *sampled* UUIDs and checked adjacent ranges, plus used **dark reads** — reads served from *both* stores with discrepancies logged (at some latency cost). Notably, the migration and verification logic were written by *different people* as a cross-check.
4. **Cutover.** Switch reads to the new cluster. Notion took **five minutes of planned downtime** — the bottleneck was waiting for the catch-up script to fully drain after taking writes offline. Their retrospective: with a sub-30-second catch-up lag, they could have done a **hot-swap at the load balancer with zero downtime.** Lesson: *invest in making the catch-up fast enough for a live swap.*

This double-write→backfill→verify→cutover pattern is the universal template for online schema/shard migrations, and it's worth carrying beyond sharding — it's how you change *anything* underneath a live system.

---

## Request Routing

Once data is sharded and shards move around, a client with a request faces a basic question: **"I need key K — which node do I talk to?"** DDIA frames three approaches, and this is an instance of the general *service discovery* problem.

```
   Approach 1: ROUTE-ANYWHERE            Approach 2: ROUTING TIER
   ────────────────────                  ──────────────────
   client → any node → that node          client → dedicated router → correct node
   forwards to the right one              (a load-balancer-like layer that knows
   (nodes gossip who owns what)            the shard map)

   Approach 3: SHARD-AWARE CLIENT
   ──────────────────────────────
   client knows the shard map itself and connects directly to the right node
```

1. **Route to any node (it forwards).** The client contacts any node; if that node doesn't own the key, it forwards the request (or tells the client where to go). Requires every node to know the full shard map — historically spread via **gossip** (Dynamo/Cassandra style, Entry #027). Simple for the client; puts the routing burden on the nodes.
2. **A routing tier.** A dedicated middle layer (a partition-aware load balancer) sits between clients and nodes, knows the shard map, and forwards. MongoDB's **mongos** works this way. Clean separation, but it's another tier to run and scale.
3. **A shard-aware client.** The client itself holds the shard map and connects directly to the right node — no forwarding hop, lowest latency. This is what Notion did (application-side routing: workspace ID → logical shard → physical DB). Fastest, but every client must be kept up to date with the map.

**The common hard problem across all three:** *how does whoever routes learn the current shard map, and stay current as rebalancing moves shards?* If the map is stale, requests go to the wrong node. This is exactly the metadata-consistency problem — and it's why routing and rebalancing are really the same problem viewed from two ends. Somebody must hold the authoritative "who owns what" map and propagate changes reliably.

### Coordination Services: ZooKeeper, etcd, mongos

The standard answer is a **strongly-consistent coordination service** that holds the authoritative shard map. **ZooKeeper** and **etcd** are the canonical choices: they're small consensus-backed key-value stores (Raft or ZAB — the consensus foundation from Entry #018) whose entire job is storing a little bit of critical metadata *reliably and consistently*. Nodes register their shard ownership there; routers subscribe and get notified when the map changes.

```
   ┌─────────────────────────────┐
   │  ZooKeeper / etcd            │  ← authoritative shard map, consensus-backed
   │  "shard 7 → node B"          │     (strongly consistent, notifies on change)
   └──────────┬──────────────────┘
              │ routers/clients subscribe
       ┌──────┴───────┐
       ▼              ▼
   [ router ]     [ router ]  ──► correct storage node
```

The mapping to the components: many systems (HBase, Kafka historically, SolrCloud) use ZooKeeper for exactly this. MongoDB's **mongos** routers read the shard map from config servers. DynamoDB's MemDS (Entry #029) is a purpose-built version of the same idea. The principle: **the shard map is small but critical, so store it in a system optimized for reliable, consistent small-metadata — not in the sharded data store itself.**

### ScyllaDB: From Gossip to Raft (the case study)

ScyllaDB's 2024 migration is a superb illustration of *why* eventually-consistent routing metadata is dangerous and *why* systems move to a consensus-backed coordinator. ScyllaDB is a Cassandra-compatible, Dynamo-style database (Entry #027) that originally managed **topology** (which node owns which token range) and **schema** via **gossip** — eventually-consistent, decentralized, no coordinator. That works for *user data* (availability-first) but is a poor fit for *metadata*, and it caused real pain:

- **Slow convergence:** nodes had to wait for gossiped state to "settle" before proceeding, slowing every topology operation.
- **Metadata availability tied to node availability:** auth data was scattered across the cluster; if the node holding a role definition was down, that user "couldn't connect to the cluster at all" — a self-inflicted denial of service.
- **Unsafe concurrent changes:** the joining node itself "drove the topology change forward," so if it failed mid-join, "the operator had to intervene and restart from scratch." There was no single authority serializing concurrent add/remove/replace operations, so they could race and corrupt the topology. Schema changes had to "rehash all the system tables" on every change — quadratic in table count, and racy under concurrency.

The fix: move topology and schema metadata onto **Raft** (Entry #018's consensus algorithm) — a dedicated Raft group ("group 0") holding the metadata in a replicated log, applied on every node "in exactly the same order." A **centralized topology-change coordinator** runs alongside the Raft leader and drives all topology changes through a **work queue**, giving "an illusion of concurrency while preserving operation safety." The payoffs map directly onto this note's themes:

```
   GOSSIP (before)                       RAFT + coordinator (after)
   ──────────────                        ──────────────────────────
   eventually-consistent metadata        linearizable metadata
   joining node drives its own change    a coordinator serializes ALL changes
   → failed join = manual restart        → coordinator resumes automatically,
   → concurrent ops race                   "no human intervention"
   → schema change every 10-20s          → schema change ~1/sec, safe & concurrent
```

Crucially, because a node's view of topology "is not dependent on the availability of the owner of the tokens" — it's replicated by Raft and available locally — a starting node gets the shard map "without reaching out to the majority of the cluster." And this coordinator is what makes ScyllaDB's automatic **tablet** load balancer possible at all: it "could not exist without" the linearizable coordinator, because safe automatic rebalancing *requires* a single authority to serialize migrations. That's the whole lesson in one line: **automatic rebalancing requires a strongly-consistent coordination service.** The tradeoff ScyllaDB accepts — giving up gossip's extreme availability for metadata — is the right one *because metadata changes are infrequent*, so strong consistency costs little and buys automation, safety, and elasticity.

---

## Local and Global Secondary Indexes

The final operational problem. The shard key answers *one* question efficiently: "find the record with key K" (you compute the shard, go there). But applications also query by *other* attributes: "find all products where `color = red`," "all users in `city = Seattle`." A **secondary index** maps an attribute value → the records with it. In a sharded system, there are two fundamentally different ways to shard the index itself, and each has a sharp tradeoff.

### Local Secondary Indexes (document-partitioned)

Each shard maintains a secondary index over *only its own data*. The index is co-located with the data it describes.

```
   Shard 1 (products A–M)          Shard 2 (products N–Z)
   ├─ data                         ├─ data
   └─ local index:                 └─ local index:
        red   → [items on shard 1]      red   → [items on shard 2]
        blue  → [items on shard 1]      blue  → [items on shard 2]
```

- **Writes are cheap.** Writing a record touches only *its own* shard — the data and its index entry are on the same shard, one write, no coordination. 
- **Reads are expensive — "scatter-gather."** To answer "all red products," you have no idea which shards hold red products (they're spread by the *primary* key, not color), so you must query *every shard*, wait for all of them, and merge the results. This is a **scatter-gather** query, and it's slow and prone to tail-latency amplification (you wait for the slowest shard — the same fan-out pattern as ANN search in Entry #013). DDIA notes this is nonetheless the more common choice (used by MongoDB, Cassandra, Elasticsearch, DynamoDB local secondary indexes), because cheap writes matter and scatter-gather is often acceptable.

### Global Secondary Indexes (term-partitioned)

Instead of each shard indexing its own data, the index itself is a separate sharded structure, partitioned by the *indexed term* (the attribute value), independent of how the primary data is sharded.

```
   Global index, sharded BY COLOR (the term), independent of data sharding:

   Index shard X (colors a–m)              Index shard Y (colors n–z)
   └─ red  → [all red items, from ANY data shard]
   └─ blue → [all blue items, from ANY data shard]
                                           └─ yellow → [all yellow items ...]
```

- **Reads are cheap.** "All red products" goes to the *one* index shard owning `red` and reads the whole list — no scatter-gather.
- **Writes are expensive and cross-shard.** Writing a single product now must update *its data shard* AND the index shard(s) for *each* of its indexed attributes — and those live on *different* shards. So one logical write becomes multiple cross-shard writes, which raises the specter of consistency: keeping the global index in sync with the data requires cross-shard coordination (a distributed transaction, or accepting that the index is *asynchronously updated* and thus can lag the data — which is what most systems do). DynamoDB's Global Secondary Indexes are exactly this: fast queries, but **eventually consistent** with the base table because the index is updated asynchronously.

### The Tradeoff, and How Systems Resolve It

```
                        Writes          Reads (query by attribute)
   ──────────────────   ─────────       ───────────────────────────
   LOCAL (per-shard)    cheap (1 shard)  expensive (scatter-gather ALL shards)
   GLOBAL (by-term)     expensive        cheap (1 index shard)
                        (cross-shard,
                         async/lagging)
```

It's the same shape as every tradeoff in this repo (the RUM conjecture again, Entry #014): you optimize writes *or* attribute-reads, and pay on the other. The practical resolutions:
- **Local indexes** when writes dominate or scatter-gather reads are rare/tolerable (the common default).
- **Global indexes** when attribute-queries are frequent and hot, and you can tolerate the index being asynchronously (eventually) consistent with the data.
- This is also why Nile argues these responsibilities *should be native* — hand-rolling a consistent global secondary index across shards in application code is exactly the kind of hard distributed-systems plumbing databases ought to provide (and why many teams simply avoid cross-shard queries by choosing the shard key to match their access pattern, as Notion did with workspace ID).

---

## Key Takeaways

- **Operating a sharded system has three problems** — rebalancing (moving shards), routing (finding the right shard), and secondary indexes (querying by non-key attributes) — unified by one theme: the **shard-map metadata is critical distributed state that must stay consistent.**
- **Rebalancing** should move minimal data and stay online. **Fully automatic rebalancing is a loaded gun** (Entry #018's lesson again): misreading a slow/overloaded node can trigger a data-migration *cascade*, and combined with automatic failure detection, a gray failure can cause a needless, harmful rebalance. The pragmatic norm: **system proposes, human approves.**
- **The migration template** (Notion's monolith→480 shards): **double-write → backfill → verify → cutover.** Double-write (via audit-log/CDC) keeps the new store current; backfill copies old data in the background (version-checking to coexist); verify via sampling + **dark reads**; cutover with minimal downtime (make catch-up fast enough for a live load-balancer hot-swap). This is the universal pattern for changing anything under a live system.
- **Request routing** has three forms: **route-to-any-node** (nodes gossip the map), a **routing tier** (mongos-style), or a **shard-aware client** (Notion; fastest, no hop). All share the hard problem: keeping the shard map current. The answer is a **strongly-consistent coordination service** (ZooKeeper/etcd; DynamoDB's MemDS) holding the authoritative small-but-critical map.
- **ScyllaDB gossip→Raft** is the case study: eventually-consistent metadata caused slow convergence, availability tied to node liveness, and unsafe concurrent topology changes. Moving metadata to **Raft + a centralized topology coordinator** made membership changes **linearizable, safe, concurrent, and automatic** — and is what makes their automatic tablet rebalancer possible. **Automatic rebalancing requires a strongly-consistent coordinator.**
- **Secondary indexes:** **local (document-partitioned)** = cheap writes but **scatter-gather** reads across all shards; **global (term-partitioned)** = cheap single-shard reads but expensive **cross-shard, asynchronously-consistent** writes (DynamoDB GSIs). The RUM-shaped tradeoff (Entry #014): optimize writes or attribute-reads, pay on the other. Many teams sidestep it by choosing a shard key that matches their access pattern.
