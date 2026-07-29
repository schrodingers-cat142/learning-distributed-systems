# Replication Logs: How Changes Are Shipped

**Date:** 2026-07-29
**Sources:**
- DDIA Chapter 5 (Replication) — the four implementations of replication logs
- [Evolution of Logical Replication — Amit Kapila (PostgreSQL committer, 2023)](https://amitkapila16.blogspot.com/2023/09/evolution-of-logical-replication.html)

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — this entry details the "replication log" that connects a leader to its followers, referenced throughout #016
- [008-lsm-trees.md](008-lsm-trees.md) — the write-ahead log (WAL) that LSM trees use for durability is the same log that "WAL shipping" replication streams to followers
- [012-b-trees-modern.md](012-b-trees-modern.md) — B-tree databases rely on a WAL for crash recovery; that WAL is what physical replication ships
- [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) — change data capture turns a database's replication log into an event stream, which is exactly the event-sourcing/CQRS integration mechanism
- [015-encoding-and-dataflow.md](015-encoding-and-dataflow.md) — CDC is a form of dataflow through async messages; logical log formats face the same schema-evolution concerns as any encoded data

> **Note on scope:** This is the second of four replication notes. Entry #016 established *that* a leader ships a "replication log" to followers. This note is about *what that log actually contains* and the trade-offs between the four ways of building it. Failover and split-brain are Entry #018; object-storage-backed databases are Entry #019.

---

## The Question This Note Answers

In Entry #016 we said the leader sends every change to its followers via a "replication log," and followers replay it to stay in sync. That left a crucial question unanswered: **what, precisely, is in that log?** When the leader executes `UPDATE accounts SET balance = balance - 100 WHERE id = 42`, what does it actually send down the wire?

The answer is not obvious, and there are four genuinely different approaches, each with real consequences. Getting this wrong causes subtle, dangerous bugs — replicas that silently drift out of sync in ways that only surface later. The four approaches, from most intuitive to most powerful, are: statement-based, write-ahead-log shipping, logical (row-based), and change data capture built on top of logical logs. We'll walk through each, and the theme that ties them together is a familiar one from Entry #015: **the more the log couples to physical storage details, the faster and simpler it is, but the less flexible it becomes** — especially across versions and systems.

---

## Approach 1: Statement-Based Replication

The most obvious idea: the leader logs every write *statement* it executes — the literal SQL — and sends that statement to followers, which execute the same SQL against their own copy.

```
   Leader executes:   UPDATE accounts SET balance = balance - 100 WHERE id = 42
        │
        │  ships the STATEMENT itself
        ▼
   Follower executes:  UPDATE accounts SET balance = balance - 100 WHERE id = 42
```

It's appealing because the log is compact (one line of SQL can affect thousands of rows) and human-readable. But it has a nasty collection of failure modes, all stemming from one root cause: **the same statement can produce different results on different machines or at different times.** DDIA lists the dangerous cases, and they're worth internalizing because they're the reason this approach fell out of favor:

- **Nondeterministic functions.** `NOW()` returns a different timestamp on the leader than when the follower replays it milliseconds later; `RAND()` returns a different random number entirely. The follower diverges silently.
- **Autoincrement and ordering dependencies.** A statement that depends on existing data (`INSERT ... SELECT`, or one relying on an autoincrement column) must run in *exactly* the same order relative to every other statement, or it produces different rows. This constrains concurrency severely.
- **Side effects.** Triggers, stored procedures, and user-defined functions may not be perfectly deterministic, so they can produce different effects on each replica.

You can work around these — for instance, the leader can replace `NOW()` with the concrete timestamp it used before logging the statement — but the workarounds are numerous and there are many edge cases. MySQL used statement-based replication by default before version 5.1, and switched to row-based when nondeterminism was detected precisely because of these hazards. The lesson: **shipping the instruction is fragile because instructions aren't guaranteed to mean the same thing everywhere.**

---

## Approach 2: Write-Ahead Log (WAL) Shipping

The opposite extreme: instead of the high-level statement, ship the low-level *physical* changes the storage engine made to the bytes on disk.

Recall from Entries #008 (LSM trees) and #012 (B-trees) that virtually every storage engine already maintains a **write-ahead log** for crash recovery — an append-only sequence recording every change *before* it's applied to the main data structure. For a B-tree, the WAL records which disk pages changed and how (down to the byte level); for an LSM tree, it records every write before it enters the memtable. That log already exists for durability. **WAL shipping** simply reuses it: the leader streams its WAL to followers, and each follower applies those exact byte-level changes to build a copy identical to the leader's, disk page for disk page.

```
   Leader's storage engine writes to its WAL:
        [page 17: bytes 40-52 changed to ...]  [page 88: new entry inserted at offset ...]
        │
        │  ships the raw WAL (physical, byte/page-level changes)
        ▼
   Follower replays the identical byte-level changes onto its own pages.
```

This is exactly what PostgreSQL's streaming replication does, and it's often called **physical replication** because it operates on the physical storage layout. It's efficient and robust against the nondeterminism problems of statement-based replication — there's nothing to re-evaluate, just bytes to copy. TAO's underlying MySQL (Entry #5) and most standard Postgres HA setups use physical log shipping.

But physical replication has one significant drawback, and it's a big one: **the WAL is tightly coupled to the storage engine and its exact version.** Because the log describes changes at the level of physical disk pages, the follower must use the *same storage format* as the leader. This makes it normally **impossible to run different database versions on leader and followers.** That sounds like a footnote until you try to do a zero-downtime upgrade: the usual trick for upgrading a database without downtime is to upgrade the followers first, then fail over to a newly-upgraded follower (Entry #018). Physical replication blocks this, because a follower on a new version can't read the old version's WAL format. You're forced into a full-stop upgrade. This limitation is the direct motivation for the third approach.

---

## Approach 3: Logical (Row-Based) Replication

The insight behind logical replication is to **decouple the replication log from the storage engine** by using a log format that describes changes *logically* — in terms of rows and their values — rather than physically in terms of disk pages. This is sometimes called a "logical log" to distinguish it from the storage engine's physical (WAL) representation.

A logical log records, for each write, the affected rows and what happened to them:

- **Insert:** the new values of all columns of the inserted row.
- **Delete:** enough information to uniquely identify the deleted row (typically the primary key, or all old column values if there's no key).
- **Update:** the identifying information plus the new values of the changed columns.

```
   Leader logs, per affected ROW (not per statement, not per page):
        INSERT accounts: {id: 43, balance: 500, owner: "Bob"}
        UPDATE accounts: id=42  →  {balance: 400}
        DELETE accounts: id=17
        │
        │  ships row-level logical changes
        ▼
   Follower applies these row changes to its own tables.
```

Notice this sidesteps *both* prior problems. It's **deterministic** — the follower applies concrete row values, so there's no `NOW()` or `RAND()` to re-evaluate (fixing statement-based replication's flaw). And because it describes rows rather than disk pages, it is **decoupled from the physical storage format** — a leader and follower can run different storage engines or different database versions and still understand each other (fixing WAL shipping's flaw). This is MySQL's **row-based binlog** (the `binlog_format = ROW` setting), and it's what makes MySQL's logical replication so versatile.

A statement that touches a thousand rows generates a thousand row-change entries, so the logical log can be larger than a statement-based log — but that verbosity is the price of correctness and flexibility, and it's usually well worth it. This logical log has another superpower: because it's a clean, storage-independent description of *what changed*, it can be parsed by external systems that aren't databases at all — which is exactly what change data capture exploits (Approach 4).

### PostgreSQL's Logical Replication: An Evolution Worth Tracing

PostgreSQL's journey to logical replication is a useful case study in how this capability gets built, because it was added incrementally over many releases (per Amit Kapila's history). The foundation was laid in **PostgreSQL 9.4** with three building blocks that are worth naming because they recur across systems:

- **Logical decoding** — the mechanism that reads the physical WAL and translates it into a clean, logical stream of changes in a customizable output format. This is the bridge from physical to logical: rather than maintaining a separate logical log, Postgres *decodes* its existing WAL into logical changes on demand.
- **Replication slots** — bookkeeping that tracks how far each consumer has read, and (crucially) prevents the leader from deleting WAL files that a follower or consumer still needs. This is the "know your log position" idea from Entry #016 made durable — a slot remembers exactly where a consumer is.
- **Replica identity** — a per-table setting that controls what identifying information is logged for updates and deletes (e.g., just the primary key, or all old values), so downstream consumers can locate the right row.

Full **publish/subscribe logical replication** arrived in **PostgreSQL 10** — you declare a *publication* (a set of tables to publish) on the leader and a *subscription* on the follower, and changes flow between them. This immediately enabled things physical replication couldn't: **replication between different major versions** (so you can do a rolling upgrade), and **selective replication** of just some tables rather than the whole database. Later releases filled in the gaps: replication of `TRUNCATE` (11), partitioned tables (13), **streaming of in-progress transactions** so a huge transaction doesn't have to fully commit before its changes start flowing (14), and row filters and column lists for even more selective replication (15). PostgreSQL 16 added the ability to **decode logically from a standby** (offloading that work from the primary) and guards against infinite loops in bidirectional setups via origin filtering.

Two items on the project's roadmap are telling about where the hard problems remain, and both connect to other notes:
- **Failover slots** — synchronizing logical replication slots from the leader to its physical standbys, so that when a standby is promoted (Entry #018), logical subscribers can reconnect and resume rather than losing their place. Slot state living only on the leader is a real gap when the leader dies.
- **DDL replication** — logical replication historically replicates *data* changes (rows) but not *schema* changes (`ALTER TABLE`). If you change the schema on the leader, subscribers don't automatically learn about it, which can break replication. This is the schema-evolution problem from Entry #015 reappearing inside the database itself.

---

## Approach 4: Change Data Capture (CDC)

The final approach isn't a fourth kind of log so much as a *reuse* of the logical log for a broader purpose. **Change data capture** is the practice of taking a database's stream of logical changes and making it available to *other systems* — not just replica databases, but search indexes, caches, data warehouses, and event-driven consumers.

The realization is powerful: a database's logical replication log is, in effect, **a real-time stream of every change to the data.** If you can tap that stream, you can keep any number of derived systems continuously up to date without the application having to explicitly write to each one.

```
                            ┌──────────────► Replica DB (traditional follower)
                            │
   Leader DB ──logical──────┼──────────────► Search index (e.g. Elasticsearch)
   (logical log /  changes  │
    binlog)                 ├──────────────► Cache (invalidate/update on change)
                            │
                            ├──────────────► Data warehouse (analytics)
                            │
                            └──────────────► Event consumers (EDA, Entry #015)
```

In practice CDC is implemented by a connector that reads the database's logical log — MySQL's row-based binlog or PostgreSQL's logical decoding stream — and publishes those changes onto a message broker like Kafka. Debezium is the best-known open-source tool for this. Downstream consumers subscribe and react. This is precisely the "dataflow through asynchronous messages" pattern from Entry #015: the database becomes an event producer, and its change log becomes the event stream.

The connection to **event sourcing** (Entry #007) is deep and worth making explicit. Event sourcing *stores* the log of changes as the source of truth and derives current state from it. CDC works the other way around — the current-state database is authoritative, and the change log is *derived* from it — but the two converge on the same architectural payoff: an append-only stream of changes from which you can build any number of read-optimized views, catch up new consumers by replaying from a known position (the snapshot-plus-log-position idea from Entry #016 yet again), and integrate independent systems without tight coupling. CDC is, in a sense, how you get most of event sourcing's integration benefits from an ordinary database you already have.

---

## Comparing the Four Approaches

The four approaches form a spectrum from "ship the instruction" to "ship the physical bytes," with logical replication as the flexible middle that CDC then builds on:

| | Statement-based | WAL shipping (physical) | Logical (row-based) | CDC |
|---|---|---|---|---|
| What's in the log | SQL statements | Byte/page-level changes | Row-level changes | Row-level changes, exported |
| Coupled to storage engine/version? | No | **Yes — tightly** | No | No |
| Deterministic? | **No** (NOW(), RAND(), ordering) | Yes | Yes | Yes |
| Cross-version replication | Possible | **No** | **Yes** | Yes |
| Log size | Small (per statement) | Medium (per page) | Larger (per row) | Larger (per row) |
| Consumable by non-databases? | No | No | Somewhat | **Yes (the whole point)** |
| Examples | MySQL < 5.1 | PostgreSQL streaming replication | MySQL row binlog, PG logical replication | Debezium → Kafka |

The trajectory across the table is the same trade-off from Entry #015 restated: coupling to physical detail (WAL shipping) buys efficiency and simplicity but forfeits flexibility and cross-version/cross-system portability; describing changes logically (row-based, CDC) costs some verbosity but unlocks version-independent replication, zero-downtime upgrades, and integration with the entire rest of your data infrastructure. Modern systems overwhelmingly trend toward logical formats for exactly these reasons — the same way binary schema-driven encodings won over ad-hoc formats.

---

## Key Takeaways

- **The "replication log" from Entry #016 can be built four ways**, differing in what they actually put on the wire.
- **Statement-based replication** ships the SQL itself — compact but *nondeterministic* (`NOW()`, `RAND()`, ordering, side effects can make replicas silently diverge). Largely abandoned for this reason.
- **WAL shipping (physical replication)** streams the storage engine's byte/page-level write-ahead log — efficient and deterministic, but **tightly coupled to the exact storage version**, which blocks running different DB versions on leader vs. follower and thus blocks zero-downtime upgrades. This is standard PostgreSQL streaming replication.
- **Logical (row-based) replication** describes changes as row-level inserts/updates/deletes — deterministic *and* decoupled from physical storage, enabling cross-version and selective replication. This is MySQL's row binlog and PostgreSQL's publish/subscribe logical replication (built on logical decoding + replication slots + replica identity, evolved incrementally from PG 9.4 onward).
- **Change data capture (CDC)** exports the logical log to *other* systems (search indexes, caches, warehouses, event consumers) via tools like Debezium → Kafka, turning the database into an event producer — the integration backbone that connects replication to event-driven architecture (#015) and event sourcing (#007).
- **The governing trade-off:** the more the log couples to physical storage, the faster and simpler but less portable; logical formats cost verbosity but win flexibility — the same lesson as data encoding in Entry #015.
