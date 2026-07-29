# Single-Leader Replication: Fundamentals

**Date:** 2026-07-29
**Sources:**
- DDIA Chapter 5 (Replication) — the leader-based model, synchronous vs asynchronous replication, setting up followers

**Related entries:**
- [005-facebook-tao.md](005-facebook-tao.md) — TAO is single-leader replication at scale: one leader per shard serializes writes, many follower tiers serve reads, MySQL async replication carries data between regions
- [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) — the write-leader / read-replica split here is the infrastructure-level version of CQRS's command/query split; both accept eventual consistency on reads
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — "moderate scale, read-heavy → Aurora with read replicas" is a single-leader replication decision

> **Note on scope:** This is the first of four replication notes. It covers the *fundamentals* — the model, the consistency knob, and how you add a follower. How changes physically travel (replication logs) is Entry #017; what happens when nodes fail (failover, split-brain) is Entry #018; and databases built on object storage are Entry #019.

---

## Why Replicate at All

Replication means keeping a copy of the same data on multiple machines connected over a network. Before getting into *how*, it's worth being clear about *why*, because the three reasons pull in slightly different directions and shape every later decision.

The first reason is **reducing latency**: put a copy of the data geographically close to your users, so a reader in Paris hits a European replica instead of crossing the ocean to Virginia. The second is **increasing availability**: if one machine dies, another copy can keep serving, so the system survives hardware faults and outages. The third is **scaling read throughput**: many copies mean many machines that can answer read queries in parallel, which matters enormously because most applications read far more than they write (TAO in Entry #5 runs at 99.8% reads).

It's important to separate replication from **partitioning** (also called sharding), because they're often confused and often combined. Replication is *the same data on several nodes*; partitioning is *different slices of the data on different nodes*. Replication buys you redundancy and read scaling; partitioning buys you the ability to hold more data and handle more writes than one machine can. Real systems usually do both — TAO shards the graph across MySQL servers *and* replicates each shard.

```
   REPLICATION (same data, many copies)     PARTITIONING (different data, split up)
   ┌────────┐ ┌────────┐ ┌────────┐         ┌────────┐ ┌────────┐ ┌────────┐
   │ A B C  │ │ A B C  │ │ A B C  │         │  A     │ │   B    │ │   C    │
   └────────┘ └────────┘ └────────┘         └────────┘ └────────┘ └────────┘
   redundancy + read scaling                more data + more write capacity
```

The genuinely hard part of replication isn't storing the copies — disks are cheap. It's handling *changes* to replicated data, and doing so in the face of nodes that fail and networks that misbehave. Everything difficult about replication flows from that.

---

## Replication Is Not a Backup (and Why You Need Both)

A tempting mistake is to think "I have three replicas, so I don't need backups." This is wrong, and understanding why sharpens what each mechanism is actually for.

Replication propagates writes *as fast as it can* to keep copies in sync. That's exactly the problem: if you run `DROP TABLE users` by accident, or a bug corrupts a record, or ransomware encrypts your data, replication faithfully and near-instantly copies that destruction to every replica. Replicas protect you against a *machine* failing. They do nothing against a *logical* error — a bad command, a bug, a malicious actor — because from the system's point of view those are legitimate writes to be replicated.

A backup is a *point-in-time* copy, deliberately not kept in sync. Its value is precisely that it lags: it lets you go back to how things were *before* the mistake. Backups protect against logical errors, accidental deletion, corruption, and attacks — the things replication is helpless against.

```
   Accidental `DROP TABLE users`
            │
            ├─► Replication:  copies the DROP to every replica in milliseconds. ✗ gone everywhere.
            │
            └─► Backup:       yesterday's snapshot still has the table.        ✓ restore from it.
```

So they are **complementary, not substitutes**. Replication gives you high availability and read scaling against hardware faults; backups give you recoverability against logical faults. A serious system has both, and treats "can we actually restore from this backup?" as a question to test regularly rather than assume.

---

## The Single-Leader Model

The most common and most fundamental replication scheme is **single-leader** (also called master–slave, primary–replica, or active–passive; this repo uses leader/follower). The rule is simple and is the source of all its properties:

- One replica is designated the **leader**. All *writes* must go to the leader.
- The other replicas are **followers**. When the leader processes a write, it also sends the change to every follower via a **replication log** (the subject of Entry #017).
- *Reads* can be served by the leader **or** any follower.

```
                        writes only
        Client ─────────────────────────►  ┌──────────┐
                                            │  LEADER   │
        Client ◄──── reads ─────────────►   └────┬─────┘
                                                 │ replication log (change stream)
                            ┌────────────────────┼────────────────────┐
                            ▼                     ▼                     ▼
                      ┌──────────┐          ┌──────────┐          ┌──────────┐
        reads ───────►│ Follower │   reads─►│ Follower │   reads─►│ Follower │
                      └──────────┘          └──────────┘          └──────────┘
```

Why funnel all writes through one leader? Because it makes ordering trivial. A single leader sees all writes in one sequence, so there's a single authoritative order of changes, and every follower applies that same sequence. This sidesteps the enormous complexity of resolving conflicting concurrent writes to the same data — the problem that multi-leader and leaderless systems must confront. TAO (Entry #5) leans on exactly this: one leader per shard *serializes* writes so cache state can never disagree with the database. The cost of the single leader is that it's a bottleneck for writes and a single point of failure for writes — which is what makes failover (Entry #018) such a critical topic.

This model is everywhere: PostgreSQL, MySQL, Oracle, SQL Server in their standard replicated setups; MongoDB replica sets; and managed services like Amazon RDS and Aurora (one writer, up to 15 read replicas — the "read-heavy → Aurora replicas" branch of Entry #4).

---

## The Central Knob: Synchronous vs Asynchronous Replication

The single most consequential decision in leader-based replication is *when the leader considers a write "done"* relative to its followers. Does it wait for followers to confirm, or not? This is the synchronous/asynchronous choice, and it's a direct trade of **durability and consistency against latency and availability**.

**Synchronous replication:** the leader waits for the follower to confirm it received the write *before* reporting success to the client.

```
   Client ──write──► Leader ──replicate──► Follower
                        │                     │
                        │ ◄──── "got it" ─────┘   (leader WAITS for this)
                        │
   Client ◄─ "success"─┘   (only now)
```

The upside is a strong guarantee: the follower has a copy that is definitely up to date, so if the leader dies, that follower can take over with no data loss. The downside is severe: if the synchronous follower is slow or unreachable, the write *blocks*. The leader must stop accepting writes until the follower recovers. A single stalled follower can freeze the whole system.

**Asynchronous replication:** the leader sends the change to followers but does *not* wait — it reports success to the client immediately.

```
   Client ──write──► Leader ─ ─ ─replicate─ ─ ─► Follower  (fire and forget)
                        │
   Client ◄─ "success"─┘   (immediately, without waiting)
```

The upside is that writes are fast and the leader keeps working even if followers lag or fail. The downside is the mirror image of sync's upside: if the leader fails *before* a write has reached the followers, that write is **lost**, even though the client was told it succeeded. This is the fundamental durability weakness of async replication — and it's why TAO's cross-region async replication (Entry #5) can lag seconds behind, and why an async setup can lose recently-acknowledged writes during failover.

### Semi-Synchronous: The Practical Middle Ground

Making *all* followers synchronous is impractical — any one of them stalling would halt writes, and the more followers you have the more likely one is slow. Making all of them asynchronous risks data loss on failover. The common compromise is **semi-synchronous**: exactly *one* follower is synchronous, and the rest are asynchronous.

```
        Client ──write──► ┌──────────┐
                          │  LEADER   │
                          └──┬────┬───┘
              synchronous ───┘    └─── asynchronous ───┐
              (leader waits)              (fire & forget)
                    ▼                        ▼        ▼
              ┌──────────┐            ┌──────────┐ ┌──────────┐
              │ Follower │            │ Follower │ │ Follower │
              │  (sync)  │            │ (async)  │ │ (async)  │
              └──────────┘            └──────────┘ └──────────┘
```

This guarantees that at least **two nodes** (the leader and the synchronous follower) have every acknowledged write — so no data is lost if the leader alone dies — while only paying the latency cost of waiting for one follower rather than all of them. If the synchronous follower becomes slow or unavailable, one of the asynchronous followers is *promoted to synchronous* in its place, so the system keeps its "at least two copies" guarantee without permanently blocking. This dynamic swap is what keeps semi-sync both safe and available.

### The Trade at a Glance

| | Fully synchronous | Semi-synchronous | Fully asynchronous |
|---|---|---|---|
| Write latency | Highest (wait for all) | Moderate (wait for one) | Lowest (wait for none) |
| Durability on leader failure | No data loss | No data loss (≥2 copies) | Recent writes may be lost |
| Availability of writes | Fragile (any follower stalls writes) | Robust (swap the sync follower) | Most robust |
| Typical use | Rare (too fragile) | Common default for HA setups | Common where some loss is tolerable |

The honest summary DDIA gives: **fully synchronous is usually impractical, so in practice replication is very often asynchronous**, and teams accept the small risk of losing the most recent writes in exchange for performance and availability. Semi-synchronous is the middle path when that risk is unacceptable. There is no free lunch here — it's the same "pick your trade-off" reality as the RUM conjecture (Entry #14), just applied to durability versus latency.

---

## Setting Up a New Follower

A practical question falls straight out of this model: how do you add a brand-new follower — or replace a dead one — *without taking the system offline*? You can't just copy the leader's files, because the leader is constantly being written to; a naive file copy would capture an inconsistent, half-updated state. And you can't lock the whole database to get a clean copy, because that would mean downtime, defeating the point of having replicas.

The solution hinges on the same replication log that keeps followers in sync, and it works because that log has a well-defined position — every change has a place in the sequence, which PostgreSQL calls a log sequence number (LSN) and MySQL calls a binlog coordinate. The steps:

```
   1. SNAPSHOT        Take a consistent snapshot of the leader's database at
                      some instant — without locking it. Note the EXACT log
                      position that corresponds to this snapshot.
                            │
                            ▼
   2. COPY            Copy that snapshot to the new follower.
                            │        (meanwhile the leader keeps taking writes,
                            │         which pile up in the replication log)
                            ▼
   3. CONNECT & ASK   The follower connects to the leader and requests all
                      changes that happened SINCE the snapshot's log position.
                            │
                            ▼
   4. CATCH UP        The follower applies that backlog of changes. It's now
                      a moving target chasing the leader; the backlog shrinks
                      until it's processing changes in near-real-time.
                            │
                            ▼
   5. CAUGHT UP       The follower is in sync and can now serve reads. From
                      here it just keeps applying new changes as they arrive.
```

The elegance is in step 1 and step 3 fitting together: because the snapshot is tagged with an exact log position, the follower knows *precisely* where to resume — not one change too early (which might double-apply) or too late (which would skip data). It replays the backlog and converges. This "snapshot + note the position + replay from there" pattern is the same idea that powers change data capture and, in a different guise, the log-position bookkeeping in Entry #017.

The identical mechanism recovers a *crashed* follower that comes back after being down for a while: it knows the last log position it processed, reconnects, and asks the leader for everything since — this is **catch-up recovery**, covered in Entry #018.

---

## Key Takeaways

- **Replication keeps copies of the same data on multiple nodes** for three reasons: lower latency (data near users), higher availability (survive node failure), and read scaling (many nodes answer reads). It's distinct from partitioning, which splits *different* data across nodes.
- **Replication is not a backup.** It faithfully copies mistakes and corruption everywhere; backups (deliberately lagging, point-in-time copies) protect against the logical errors replication can't. You need both — they're complementary.
- **Single-leader replication** routes all writes through one leader that serializes them into a replication log; followers replay that log, and reads can hit any node. The single order sidesteps write-conflict resolution, at the cost of a write bottleneck and a single point of failure.
- **Synchronous vs asynchronous is the central knob**, trading durability/consistency against latency/availability. Fully synchronous is fragile; fully asynchronous risks losing recent writes on failover; **semi-synchronous** (one sync follower, promoted-on-failure) is the common middle ground guaranteeing ≥2 copies of every acknowledged write.
- **New followers are added without downtime** by snapshotting the leader at a known log position, copying it, then replaying the change log from that exact position until caught up — the same mechanism that recovers crashed followers.
