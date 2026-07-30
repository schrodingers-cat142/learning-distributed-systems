# Multi-Leader Replication

**Date:** 2026-07-30
**Sources:**
- DDIA Chapter 5 (Replication) — multi-leader replication, handling write conflicts, replication topologies
- [PostgreSQL bi-directional replication using pglogical — AWS Database Blog](https://aws.amazon.com/blogs/database/postgresql-bi-directional-replication-using-pglogical/)
- [If You *Must* Deploy Multi-Master Replication, Read This First — Robert Hodges (2012)](https://scale-out-blog.blogspot.com/2012/04/if-you-must-deploy-multi-master.html)
- [HBASE-7709: Infinite loop possible in Master/Master replication — Apache HBase JIRA](https://issues.apache.org/jira/browse/HBASE-7709)

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — the baseline this departs from: one leader means one write order and no conflicts; multi-leader trades that away
- [017-replication-logs.md](017-replication-logs.md) — multi-leader between different databases is built on logical replication (row-based / CDC); loop-back prevention is a logical-log feature
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — last-write-wins and version vectors here are the same conflict/causality machinery catalogued there
- [018-node-outages-failover-split-brain.md](018-node-outages-failover-split-brain.md) — split-brain is an *accidental* multi-leader situation; multi-leader is the *deliberate* version, and it inherits the same "no single authority" hazards

> **Note on scope:** This note covers multi-leader replication end to end — when it's justified, the write-conflict problem at its core, and the topology hazards. **CRDTs and operational transformation** are named here as conflict-resolution approaches but not explored; they belong to a later collaborative-editing / local-first deep dive. **Leaderless (Dynamo-style) quorum** replication remains deferred to its own note.

---

## Where This Sits

Everything from Entry #016 onward assumed a single leader: all writes funnel through one node, which stamps them into one ordered log (Entry #017) that every follower replays. That single order is the quiet hero of the whole design — because there is exactly one authority deciding "what happened and in what sequence," **write conflicts simply cannot occur.** Two clients trying to change the same row both go to the same leader, which serializes them; one wins, one loses, and everyone downstream sees the same outcome.

**Multi-leader replication** (also called *active/active* or *master-master*) throws that away on purpose. It allows **more than one node to accept writes**, with each leader simultaneously acting as a follower to the others — every leader forwards its writes to all the other leaders, and applies theirs in return.

```
   SINGLE-LEADER (Entry #016)              MULTI-LEADER (this note)
   ─────────────────────────              ────────────────────────
        writes                              writes        writes
          │                                   │              │
          ▼                                   ▼              ▼
     ┌─────────┐                         ┌─────────┐    ┌─────────┐
     │ LEADER  │                         │ Leader A│◄──►│ Leader B│
     └────┬────┘                         └────┬────┘    └────┬────┘
     ┌────┴────┐                              │  each also applies the
     ▼         ▼                              ▼  other's writes → CONFLICTS
  follower  follower                       followers        followers
   ONE write order → no conflicts       TWO write orders → conflicts are inevitable
```

The moment you have two nodes independently accepting writes, you have two independent write orders, and reconciling them is *the* problem multi-leader replication exists to manage. DDIA's framing, and every practitioner's, is blunt: the benefits are real but narrow, and the cost — conflicts — is severe enough that you should reach for it only when you genuinely need it. This note is about *when* that is, *what* goes wrong, and *how* people cope.

Note the relationship to Entry #018: **split-brain is accidental multi-leader.** When a failover goes wrong and two nodes both think they're the leader, you've stumbled into multi-leader replication without meaning to — and it corrupts data precisely because nothing was designed to reconcile the two write streams. Multi-leader is the *deliberate* version, where you accept the two streams up front and build machinery to merge them.

---

## When Multi-Leader Is Actually Justified

DDIA names three situations where the pain is worth it. The common thread: in each, a single leader is either a bottleneck you can't tolerate or literally impossible.

**1. Multi-datacenter operation.** If your users span continents and you run a datacenter in each, a single leader means every write from every region crosses the ocean to that one leader — adding hundreds of milliseconds and making all writes hostage to that one datacenter's health. With a leader *per datacenter*, each region's writes are handled locally and fast, then replicated asynchronously to the other regions. Writes tolerate inter-datacenter network problems (each region keeps working independently) and cross-region latency is hidden from the write path. This is exactly the AWS pglogical use case: application stacks in multiple regions, each with a writable regional cluster, kept in sync bi-directionally, with Route 53 geolocation routing users to the nearest one. (Contrast, as AWS notes, with Aurora Global Database, which only gives you *read-only* clusters in secondary regions — single-leader across regions.)

**2. Clients with offline operation.** Think of a calendar app on your phone and laptop. Each device must let you add appointments *while offline* — so each device is effectively a leader with its own local database, and syncing when connectivity returns is multi-leader replication with each device as a "datacenter of one." The Bayou project behind the session guarantees (Entry #021) was built for exactly this mobile, intermittently-connected world.

**3. Collaborative editing.** Real-time collaborative apps (Google Docs-style) are multi-leader in disguise: each user's local replica accepts edits immediately (for responsiveness), then propagates them to others, with conflicts reconciled continuously. This is where CRDTs and operational transformation live — the deep treatment of which is deferred to a later local-first / collaborative-editing note, as agreed.

The unifying reason multi-leader wins in all three: **the write must be accepted locally, without a round-trip to a distant single authority** — because that authority is too far (datacenters), unreachable (offline), or too slow for interactivity (collaboration).

---

## The Core Problem: Write Conflicts

Here is the whole difficulty in one example. The same calendar entry is edited on two leaders at nearly the same time:

```
   Leader A (New York)                    Leader B (London)
   ───────────────────                    ─────────────────
   User sets title = "Lunch"              User sets title = "Dinner"
        │                                      │
        │  (both accepted locally, fast)       │
        └──────────── async replicate ─────────┘
                          │
              A now receives "Dinner"; B now receives "Lunch"
                          │
              Which wins?  A thinks "Lunch", B thinks "Dinner".
              The databases have DIVERGED. Someone must reconcile.
```

In single-leader this cannot happen — both edits go to one leader, which orders them. In multi-leader, both writes are accepted *before* either leader knows about the other, so the conflict is only *detected later*, asynchronously, when the writes meet. And detecting a conflict late is fundamentally harder than preventing it early.

The coping strategies below are **DDIA's** set of conflict-handling approaches; the "best-first" ordering is my editorial lean, not a ranking DDIA imposes. Attribution matters here: only the *first* strategy (conflict avoidance) reflects Hodges' argument — in fact his whole thesis cuts *against* the idea of a resolution menu, since he insists tools cannot truly resolve conflicts at all. Strategies 2–4 are DDIA's catalogue of what systems nonetheless attempt; pglogical's settings are concrete instances of them.

### Strategy 1: Conflict Avoidance (the best option)

The most reliable way to handle conflicts is to **ensure they can't happen.** If you can arrange for all writes to a *particular* record to always go through the *same* leader, then for that record you effectively have single-leader replication — and single-leader has no conflicts. The usual technique is to **partition by some key** so that, say, all of a given user's data is "homed" to one datacenter's leader.

This is *the* recurring theme across the sources, and it's worth stating as a principle: **multi-leader systems are far better at *avoiding* conflicts than *resolving* them.** This is precisely Hodges' central argument — neither the replication tool nor the database can truly resolve conflicts for you, so the entire discipline is designing your application so conflicts don't arise. DDIA adds the important caveat that conflict avoidance breaks down exactly when a user moves (the datacenter they're homed to changes) or when the "natural" partition doesn't match the access pattern — at which point the record's home leader shifts and concurrent writes to it become possible again.

### Strategy 2: Last-Write-Wins (LWW) — simple and dangerous

Give every write a timestamp; on conflict, the write with the highest timestamp wins, the others are silently discarded. This is pglogical's `last_update_wins` (and its mirror `first_update_wins`). It's appealing because it's trivial and always produces a single converged answer.

It is also **lossy and dangerous**, and you should know exactly why:

```
   Write "Lunch"  @ t=100  (Leader A)   ┐
   Write "Dinner" @ t=101  (Leader B)   ┘  → LWW keeps "Dinner", DISCARDS "Lunch"
                                            silently. That write is just GONE.
```

Two problems compound. First, **it throws away data** — the losing write vanishes with no notification, which is unacceptable for anything where every write matters. Second, and more subtly, **it relies on timestamps to order events across machines, and clocks disagree** — this is the exact clock-drift hazard from the S3 leader-election note (Entry #019) and the "you cannot trust cross-node clocks" theme throughout. A write that happened *later* in real time can carry an *earlier* timestamp and lose. LWW converges, but to a possibly-wrong and possibly-lossy answer. It's fine when losing conflicting writes is genuinely acceptable (e.g., caching, or immutable-ish data); it's a landmine otherwise.

### Strategy 3: Version Vectors / Detecting Concurrency

Rather than blindly picking a timestamp winner, track causality explicitly with **version vectors** (the same machinery from Entry #021's session guarantees). These let the system *distinguish* two cases that LWW conflates: writes that are causally ordered (one genuinely happened after the other, so the later one should win) versus writes that are truly *concurrent* (neither knew about the other — a real conflict that needs real handling). This doesn't resolve conflicts by itself, but it tells you *which writes actually conflict*, so you can avoid discarding data that was never in conflict. It's the honest foundation on top of which smarter resolution is built.

### Strategy 4: Custom / Application Resolution

Let the application decide, either on write (run resolution logic when a conflict is detected) or on read (store all conflicting versions, hand them to the app or user next time the record is read, and let them merge — the "siblings" approach). This is where **CRDTs** (conflict-free replicated data types) and **operational transformation** come in — data structures and algorithms that merge concurrent edits automatically and sensibly (e.g., two people adding different items to a set, or editing different parts of a document). These are the right tool for collaborative editing, and they get their own deep dive in a later note; here it's enough to know they're the principled alternative to "pick a timestamp winner and lose data."

pglogical's five settings map cleanly onto this menu: `error` (stop and make a human look — refuse to guess), `apply_remote` / `keep_local` (a fixed, blunt rule), and `last_update_wins` / `first_update_wins` (timestamp-based, with the clock caveats above — and note the timestamp options require enabling commit-timestamp tracking, which itself costs performance).

---

## Replication Topologies

The second big multi-leader design question is the **topology**: which leaders replicate to which? With two leaders it's trivial (they just talk to each other), but with three or more there are choices, and each has distinct failure modes.

```
   CIRCULAR                    STAR                        ALL-TO-ALL
   A → B → C → A               B   C                       A ─── B
   (each forwards               \ /                        │ \ / │
    to the next)                 A   (hub relays            │ / \ │
                                / \   to all spokes)        C ─── D
                               D   E                     (every leader → every other)
```

**Circular** (each node forwards to the next in a ring) and **star** (one central hub relays to all others) both minimize the number of connections. But they share a fatal fragility: **a single node's failure breaks the replication path.** In a ring, if B goes down, A's writes can't reach C until the topology is manually reconfigured to route around B. Hodges is especially scathing about MySQL circular replication, warning it "results in broken systems if one of the masters fails" with three or more nodes — a node dropping out severs the ring.

**All-to-all** (every leader replicates directly to every other) avoids that: because there are redundant paths, one leader can drop out and rejoin without reconfiguring anything, since its writes still reach everyone through the others. This is why Hodges *recommends* all-to-all — its fault tolerance — while noting it doesn't scale to *many* leaders (the connection count grows quadratically). It's the topology behind most robust multi-leader setups.

### But All-to-All Has Its Own Problems

Two, in fact — and this is the "all-to-all isn't a free lunch either" point.

**Problem 1 — Causality / ordering (consistent-prefix, again).** With multiple redundant paths of *different speeds*, writes can arrive at a node **out of causal order**. A classic case: an INSERT of a row arrives at some node *after* an UPDATE to that same row — so the update targets a row that doesn't exist yet. This is exactly the consistent-prefix / "effect before cause" anomaly from Entry #020, now arising from topology rather than partition lag. Timestamps don't fix it (clocks again); the proper fix is causal tracking — e.g., version vectors — so a node knows to hold an update until the write it depends on has arrived.

**Problem 2 — Infinite replication loops.** This is the subtle one, and the reason loop-prevention is a *named feature* of every multi-leader system. If A replicates to B and B replicates to C and C replicates back to A, then a write made at A travels A→B→C→A… and, without a stopping rule, **circulates forever**, being re-applied on every lap.

The fix is to tag each write with the identity of the nodes that have already seen it, and refuse to forward a write to a node already in that list. Two references show this concretely:

- **pglogical** does it with `forward_origins := '{}'` — an empty origins list means "only forward changes that *originated locally*; never re-forward a change I received from someone else." A write thus makes exactly one hop from its origin to each peer and stops. This is a logical-replication feature (Entry #017) — the logical log records each change's *origin*, which is what makes loop-back prevention possible.

- **HBASE-7709** is the cautionary tale of getting this *wrong*. HBase tracked only a **single cluster ID** per edit. The bug: clusters A and B in master/master, plus a cluster C accidentally also replicating into A. Because an edit carried just one cluster ID (which "won't be reset" once set), the system couldn't tell that an edit from C had *already visited* a cluster — so edits from C would, in the report's words, be "bouncing between A and B. Forever!" The fix options are the general lesson in provenance tracking: maintain a **set** of all cluster IDs an edit has visited (the "cleanest approach") and drop any edit a cluster has already seen, or fall back to a **hop-count** ceiling. The single-ID scheme only ever worked for exactly two clusters; real topologies need the full visited-set.

```
   The loop-prevention rule (both pglogical and the HBASE-7709 fix):

   Each write carries the set of nodes that have already applied it.
   Before forwarding a write to node X:
        if X is already in the visited-set  →  DROP it (X has seen it).
        else                                →  forward, and add X to the set.

   Single ID  → works for 2 nodes only (HBase's bug).
   Visited-set→ works for any topology.  ← the correct general solution.
```

---

## The Practitioner's Verdict

Both real-world sources converge on the same sober advice, and it's worth ending on because it's the honest bottom line the theory builds toward.

### Hodges' Rules for Multi-Master

Hodges frames the entire discipline around one distinction: **multi-master systems cannot *resolve* conflicts, so you must *avoid* them.** His article is structured as a set of practical rules for anyone who "must" deploy it anyway, and they're concrete enough to be worth listing:

- **Use the right technology, configured properly** — he prefers an **all-to-all** topology for its fault tolerance (masters drop out and rejoin without reconfiguration), and warns explicitly against **MySQL circular replication** (breaks if one master fails with 3+ nodes) and against **synchronous replication over WAN** beyond ~50km (a "siren-like promise of consistency" that performs badly and fails when links go down).
- **Use row-based replication (RBR)** to kill non-determinism — his example: `UPDATE emp SET salary = salary * 1.1 WHERE dep_id = 35` updates a different number of rows depending on which inserts have replicated, so shipping the *statement* is ambiguous; ship the *row changes* instead. (He concedes in the comments that RBR removes ambiguity about *what* the change is but still doesn't guarantee consistent masters with 3+ nodes.)
- **Prevent key collisions on INSERTs** — use `auto-increment-offset` / `auto-increment-increment` so each server generates a disjoint key space (server 1: 1, 5, 9…); tables lacking auto-increment primary keys are "suspect" and need UUIDs or a server-ID embedded in the key. (pglogical gives the same advice as split odd/even sequences per node.)
- **Remove triggers or make them harmless** — triggers are "a bane of replication"; if they fire on the replica side under row replication they double-apply or conflict. Native MySQL disables them on slaves under RBR, but not every tool does, so you may have to guard each trigger with an `IF` that stops it running on the replica.
- **Beware semantic conflicts in the application** — the hard, *silent* ones no tool can catch: an accounting system needing an unbroken invoice-number sequence, or a month-end report needing globally-consistent balances. His verdict: *"Some applications simply do not work with multi-master topologies."*
- **Have a plan for sorting out mixed-up data** — keep consistency-checking tools (e.g. `pt-table-checksum`), be ready to quiesce all but one master to repair, and be handy with SQL (he suggests disabling binary logging on the repair session so fixes don't re-replicate and break other masters).
- **Test everything** — on realistic data, on *all* masters (not just one), resetting between runs, and checking consistency afterward by quiescing the app and comparing tables.

The unifying danger he keeps returning to: it isn't the *obvious* conflicts (duplicate keys, competing updates) — those break replication loudly and get noticed. It's the *silent* semantic ones that look fine until the data is quietly corrupt, which is why prevention beats detection.

AWS reaches the same conclusion from the vendor side: bi-directional replication "involves extra work and adds complexity," and they explicitly recommend confirming that a plain read replica (single-leader) can't meet your needs *first*. The operational fine print reinforces it — needs primary keys, sequences must be manually partitioned, DDL isn't auto-replicated (schema changes risk breaking replication and should be done during a write pause), and conflict resolution has real performance cost.

So the shape of the whole topic: **multi-leader is a genuinely-necessary tool for a narrow set of problems (multi-datacenter writes, offline clients, collaboration) and a genuinely-dangerous one everywhere else.** The core trade you're making is giving up single-leader's one authoritative write order — and with it, freedom from conflicts — in exchange for local, always-available writes. Everything hard about multi-leader is the bill for that trade.

---

## Key Takeaways

- **Multi-leader (active/active) replication** lets multiple nodes accept writes, each acting as a follower to the others. It sacrifices single-leader's one authoritative write order — and with it, single-leader's freedom from **write conflicts**. (Split-brain from Entry #018 is *accidental* multi-leader; this is the deliberate, managed version.)
- **Justified in three cases:** multi-datacenter (local fast writes per region), offline clients (each device is a leader), and collaborative editing — all cases where a write must be accepted locally without a round-trip to a distant single authority.
- **Write conflicts are the core problem**, detected late and asynchronously. Coping strategies, best-first: **avoid** conflicts (home each record to one leader — "avoid, don't resolve"); **last-write-wins** (simple but silently lossy and clock-dependent — dangerous); **version vectors** to distinguish causal from truly-concurrent writes; and **custom / CRDT / OT resolution** (deferred to a later note).
- **Topologies:** circular and star minimize connections but break when any node fails (Hodges: MySQL circular "results in broken systems if one master fails"). **All-to-all** is fault-tolerant (nodes drop out and rejoin freely) and is the recommended choice, but doesn't scale to many leaders.
- **All-to-all's own problems:** writes can arrive **out of causal order** (the consistent-prefix anomaly again — fix with causal tracking, not timestamps), and writes can **loop forever** without provenance tracking. Loop prevention = tag each write with the nodes that have seen it and never forward to one already in the set — pglogical's `forward_origins`, and the fix for **HBASE-7709** (whose single-cluster-ID bug let edits "bounce between A and B forever"; the fix tracks a *set* of visited cluster IDs).
- **The verdict (Hodges + AWS):** you cannot resolve conflicts, only avoid them; watch for *silent semantic* conflicts (invoice sequences, month-end balances) that no tool catches; and confirm a single-leader read replica won't do the job before taking on multi-leader's complexity.
