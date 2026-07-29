# Object-Storage-Backed Databases and Zero-Disk Architecture

**Date:** 2026-07-29
**Sources:**
- [Leader Election With S3 Conditional Writes — Gunnar Morling (2024)](https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/)
- DDIA Chapter 5 framing (replication) applied to a storage model DDIA predates

**Related entries:**
- [018-node-outages-failover-split-brain.md](018-node-outages-failover-split-brain.md) — this entry's S3 leader election is a concrete instance of the fencing-token / eventual-leader theory there; the epoch *is* the fencing token
- [016-single-leader-replication.md](016-single-leader-replication.md) — object storage changes *where* replicated durability lives (the object store, not a leader's local disk), reframing the sync/async trade-off
- [008-lsm-trees.md](008-lsm-trees.md) — LSM engines are the natural fit for object storage: their immutable, append-only SSTables map perfectly onto write-once object stores
- [002-retry-policies.md](002-retry-policies.md) — a stale leader retrying after its lease expired is exactly the "zombie" case, and the same admission-control instincts apply

> **Note on scope:** Fourth and final replication note. Entries #016–#018 covered the classic leader/follower model, its logs, and its failure handling — all assuming databases own local disks. This note covers the modern reframing: databases built directly on object storage, where the durability and coordination story changes shape.

---

## The Shift This Note Is About

Everything in Entries #016–#018 quietly assumed a particular hardware picture: each database node owns a **local disk**, replication exists to get copies of the data onto *other* nodes' local disks, and the whole apparatus of leaders, followers, and failover exists to manage those independent copies. That assumption held for decades. Cloud object storage — Amazon S3, Google Cloud Storage, Azure Blob Storage — quietly dissolves it, and a new class of database architecture has grown up around the change.

Object storage is not a filesystem and not a block device. It's a simple, massively scalable key-value store for large blobs: you `PUT` an object under a key, you `GET` it back, you `LIST` keys, you `DELETE`. What makes it transformative for databases is the bundle of properties it offers essentially for free: it is **already replicated and highly durable** (S3 advertises eleven nines of durability, storing each object redundantly across multiple facilities), it is **effectively infinite** in capacity, and it is **cheap** — often an order of magnitude cheaper per byte than the block storage (EBS-style volumes) that databases traditionally run on.

The catch, historically, was that object storage was **slow** (tens of milliseconds per request, versus microseconds for local NVMe) and offered only **weak consistency and no atomic operations**, which made it useless as the primary store for a database that needs fast, coordinated writes. The story of this note is how newer designs work *around* the latency and — thanks to a recent feature — *exploit* a new atomic operation to solve coordination problems that used to require a separate cluster of servers.

---

## Databases Backed by Object Storage

The core idea is to make object storage the **source of truth** for a database's data, rather than a node's local disk. The database nodes become largely **stateless compute** that reads and writes objects; the durable, replicated data lives in S3.

```
   CLASSIC (Entries #016–#018)              OBJECT-STORAGE-BACKED
   ─────────────────────────               ──────────────────────
   ┌────────┐  ┌────────┐                  ┌────────┐  ┌────────┐
   │ Leader │  │Follower│                   │Compute │  │Compute │  ← stateless-ish
   │ +local │  │ +local │  replicate         │ node   │  │ node   │    nodes
   │ disk   │─►│ disk   │  between disks     └───┬────┘  └───┬────┘
   └────────┘  └────────┘                       │           │
   durability = N local copies                  ▼           ▼
   kept in sync by the leader              ┌──────────────────────┐
                                           │   Object store (S3)   │  ← the durable,
                                           │  durable + replicated │    already-replicated
                                           │  source of truth      │    source of truth
                                           └──────────────────────┘
```

Why does this map so naturally onto certain database designs? Because **LSM-tree storage engines** (Entry #008) already produce exactly the kind of files object storage wants. Recall that an LSM tree writes **immutable, sorted SSTable files** and never modifies them in place — it only creates new files and later merges them during compaction. Object storage is a **write-once** medium (you create an object; you don't edit it byte-by-byte). These fit together perfectly: an LSM engine can flush each SSTable as an S3 object and treat compaction as "read some objects, write a new object, delete the old ones." A B-tree (Entry #012), which relies on in-place page updates, is a far worse fit. This is a big part of why the object-storage-database wave is built on LSM-style engines.

The write-ahead log poses the harder problem. A database needs durable, low-latency writes (Entry #016's whole sync/async discussion is about *when* a write is safely persisted), but object storage is too slow to put every individual write to S3 synchronously. The common pattern is to **buffer recent writes** — in memory and often in a small, fast, genuinely-durable layer — batch them, and flush to object storage periodically, accepting that the object store holds slightly-behind-but-massively-durable data while the newest writes live in the faster tier. This is the sync/async trade-off from Entry #016 reincarnated: the object store is your durable "follower," and how aggressively you flush to it is exactly the durability-versus-latency knob.

---

## Tiered Storage

A gentler, extremely common version of this idea doesn't move the *whole* database to object storage — it moves the *cold* part. This is **tiered storage**, and it's now standard in systems like Apache Kafka, ClickHouse, and many data warehouses.

The observation is that data has a temperature. Recently-written data is **hot**: read often, latency-sensitive, worth keeping on fast local disk. Older data goes **cold**: rarely read, but must be retained (for compliance, replay, or the occasional historical query). Keeping years of cold data on expensive local SSD is wasteful. Tiered storage keeps hot data local and transparently offloads cold data to cheap object storage:

```
   Hot tier (local NVMe/SSD)            Cold tier (object storage)
   ┌───────────────────────┐           ┌──────────────────────────┐
   │ recent segments       │  age out  │ older segments            │
   │ (fast reads/writes,   │ ────────► │ (cheap, deep, slower;     │
   │  latency-sensitive)   │           │  read on demand)          │
   └───────────────────────┘           └──────────────────────────┘
```

In Kafka's tiered storage, for example, recent log segments stay on the broker's disk while older segments move to S3; a consumer reading old data transparently fetches from the object store. The payoff is decoupling **retention** from **local disk cost** — you can keep data effectively forever without provisioning ever-larger local volumes, and brokers hold far less local state (which, not coincidentally, makes them faster to move and recover). Tiered storage is the pragmatic on-ramp to the fuller object-storage-backed model.

---

## Zero-Disk Architecture (ZDA)

Push the idea to its logical conclusion and you get **zero-disk architecture**: the database's compute nodes hold *no durable local disk at all*. All persistent state lives in object storage; the nodes keep only ephemeral in-memory caches and buffers. If a node dies, nothing durable is lost with it, because it never held anything durable in the first place.

This inverts the entire premise of Entries #016–#018 in a way worth stating plainly:

```
   Classic replication:  durability comes from having MANY NODES each hold a copy.
                         The scary question is "did the write reach enough disks
                         before the leader died?" (Entry #016's sync/async problem,
                         Entry #018's data-loss-on-failover problem).

   Zero-disk:            durability comes from the OBJECT STORE, which is already
                         replicated to eleven nines. Nodes are disposable. The scary
                         question mostly disappears — a dead node loses nothing,
                         because it owned nothing durable.
```

The benefits are substantial and interlocking. **Elasticity:** because nodes are stateless, you can add or remove them almost instantly — there's no multi-hour "copy a snapshot and catch up the log" dance from Entry #016 to bring a new node online, because there's no local dataset to copy. **Cheaper and simpler durability:** you outsource the hard, dangerous parts of replication (getting copies onto enough disks, not losing recent writes on failover) to the object store, which already solved them at massive scale. **Operational simplicity:** far fewer stateful-node failure modes to reason about.

The costs are equally real, and they're the mirror image. **Latency:** every genuine read-through or flush to object storage pays that tens-of-milliseconds tax, so ZDA systems live or die by their caching and write-buffering. **The write-durability gap:** between "client got an ack" and "data is safely in S3" there's a window living in a faster tier, and closing that window safely is the central engineering problem (it's Entry #016's durability question, just relocated). And **coordination** — which node is currently in charge of writing, so two nodes don't stomp on each other in the shared object store — becomes the crux. Which brings us to the feature that makes this tractable.

---

## The Enabling Primitive: Conditional Writes (Compare-and-Swap)

The thorniest problem in a shared-storage design is coordination: if many stateless nodes can all reach the same object store, **how do you ensure only one of them acts as the writer/leader** (Entry #016's single-leader requirement) without standing up a separate ZooKeeper/etcd cluster — which would reintroduce exactly the stateful coordination service ZDA was trying to avoid?

The answer is an **atomic conditional write** on the object store itself. Google Cloud Storage and Azure Blob Storage have long supported compare-and-swap style operations. Amazon S3 added a limited but sufficient form in **August 2024**: `PutObject` now accepts an **`If-None-Match: *`** header, meaning *"only create this object if no object with this key already exists; otherwise fail."* On conflict, S3 returns **`412 Precondition Failed`**. It's weaker than a full CAS (it's only "create-if-absent," not "swap-if-value-equals"), but it's an atomic test-and-set, and that's enough to build a lock.

The significance is that the object store — the thing you already depend on — becomes your coordination primitive. No separate consensus cluster required. As the leader-election theory in Entry #018 anticipated: *if your storage provides atomic conditional writes, you may be able to skip dedicated leader election entirely.*

---

## Leader Election on S3, Step by Step (Morling)

Gunnar Morling's write-up builds a working leader election on top of S3 conditional writes, and it's a beautiful concrete instance of everything in Entry #018 — fencing tokens, leases, eventual-only correctness, all made tangible. Let me walk through it, because each refinement patches a specific failure mode we already met theoretically.

**The basic race.** Nodes compete to create a lock object; whoever's conditional `PUT` succeeds is the leader. But `If-None-Match` only prevents overwriting an *existing* key — so a single fixed key can never be re-acquired once created. The fix is a **per-epoch lock file**: an epoch is a strictly increasing integer embedded in the key name (`lock_0000000001.json`, `lock_0000000002.json`, …). To acquire leadership you `LIST` the lock files, find the highest epoch, and try to `PUT` the *next* epoch with `If-None-Match: *`.

```
   1. LIST lock files.  Highest is lock_0000000007.json.
   2. Is it expired / absent?  If yes → try to create lock_0000000008.json
        with If-None-Match:*
   3. PUT succeeds (no 412) → I am leader for epoch 8.  Begin work.
   4. PUT fails with 412    → someone else won the race for epoch 8.  Back to step 1.
   5. Current lock still valid & held by another → wait, retry later.
```

Because the `PUT` is atomic, if two nodes race to create epoch 8, **exactly one wins** and the other gets a `412` — guaranteeing a single leader *per epoch*.

**The epoch IS the fencing token.** This is the payoff that ties directly to Entry #018. The epoch is a strictly-increasing number identifying each leadership term — which is precisely the definition of a fencing token. When the leader goes off to do work against some *other* system (write to a downstream API, say), it passes its epoch along. That downstream system tracks the highest epoch it has seen and **rejects any request carrying a lower epoch** — fencing off a stale "zombie" leader whose term has passed. As Entry #018 stressed, fencing only works if the downstream resource *cooperates* by checking the token; S3 itself doesn't natively check epochs, so this cooperation must be built into whatever the leader acts upon.

**Liveness needs a lease.** If a leader simply crashes without releasing its lock, no one can ever take over — a liveness disaster. A graceful leader sets `"expired": true` in its lock file so others promptly elect a successor. But a *crashed* leader can't do that. So the lock file carries a **`validity_ms`** (e.g., 60 s), turning it into a **lease**: leadership is valid only for that window unless renewed. S3 exposes each object's last-modified timestamp, so a would-be usurper can check whether the current lease has lapsed, and the leader can "touch"/rewrite its lock file to renew. This is the standard lease pattern — leadership that must be continually re-earned, so a dead leader's grip automatically expires.

**Correctness is only *eventual* — clock drift guarantees it.** Here Morling's write-up lands exactly on Entry #018's deep result. Lease expiry depends on time, and **you can never trust clocks to agree across nodes.** The dangerous cases: a leader whose clock runs *slow* believes its lease is still valid after it has actually expired; a challenger whose clock runs *fast* declares the lease expired too early. Either produces **two nodes that both believe they are leader** — the split-brain of Entry #018, arising here purely from clock skew. Morling is explicit that this is inherent: leader election "will only ever be eventually correct," and checking the lease and doing the work aren't atomic (a GC pause between the two — the very scenario from Entry #018 — can strand a leader acting on an expired lease). The mitigations are best-effort (compare against an S3 timestamp, use a time-sync service) and the *real* safety net is the fencing token, not the clock. This is the leader-election-vs-consensus lesson from Entry #018 made concrete: **you cannot guarantee exactly one leader; you can only guarantee that a stale one gets fenced off.**

**The connection to object-storage databases.** Morling ties it back to systems like SlateDB (an LSM key-value store built on object storage), which historically detected competing writers by having them create serially-named files and noticing conflicts — essentially the epoch idea by hand. With native conditional writes, that coordination becomes trivial and, crucially, **eliminates any external stateful coordination service**. That's the final piece that makes zero-disk architecture practical: the object store provides durability *and* the atomic primitive needed for coordination, so a ZDA database needs nothing but stateless compute plus a bucket.

---

## How It All Fits Together

```
   Object storage gives you, for cheap:
       • durability + replication (eleven nines)  ─┐
       • effectively infinite capacity            ─┼─► so you can put the DB's
       • an atomic conditional write (Aug 2024)   ─┘    source of truth here
              │                         │
              │                         └────────────► and use it to elect a leader
              ▼                                          (epoch = fencing token; lease
   Tiered storage: offload COLD data only              for liveness; eventually correct
        (Kafka, ClickHouse) — the on-ramp              — Entry #18 made concrete)
              │
              ▼
   Zero-Disk Architecture: NO durable local disk; nodes are stateless compute.
        • elastic (no snapshot-and-catch-up to add a node — cf. Entry #16)
        • durability outsourced to the object store (cf. Entry #16 sync/async)
        • pays a latency tax → survives on caching + write-buffering
        • coordination solved by conditional writes, not a ZooKeeper cluster
```

The elegant closing observation is that object storage doesn't so much *break* the replication story of Entries #016–#018 as **relocate** it. The durability question ("did enough copies get the write?") moves into the object store, which already answers it. The coordination question ("who is the single leader?") gets a new, cheaper tool (conditional writes) but the *same* unavoidable theory (Entry #018: only eventual leadership, fence the zombies). Nothing fundamental was repealed — the hard problems moved to where they're already solved, or got a better primitive while keeping the same guarantees.

---

## Key Takeaways

- **Object storage (S3/GCS/Azure Blob)** offers cheap, effectively-infinite, already-replicated (eleven-nines) durable storage, but with high latency and — until recently — no atomic operations. A new class of databases makes it the *source of truth* instead of local disk.
- **LSM engines fit object storage** because their immutable, append-only SSTables map onto a write-once medium; in-place-update B-trees fit poorly. The write path buffers recent writes in a faster tier and flushes batches to the object store — Entry #016's sync/async durability knob, relocated.
- **Tiered storage** (Kafka, ClickHouse) is the common on-ramp: keep hot data on local disk, transparently offload cold data to object storage, decoupling retention from local-disk cost.
- **Zero-disk architecture (ZDA)** removes durable local disk entirely — nodes are stateless compute, durability comes from the object store. Benefits: instant elasticity (no snapshot-and-catch-up), outsourced durability, operational simplicity. Costs: a latency tax (mitigated by caching), a write-durability gap to close, and coordination as the crux.
- **Conditional writes are the enabling primitive.** S3's `If-None-Match: *` (Aug 2024; GCS/Azure had CAS earlier) is an atomic create-if-absent that returns `412` on conflict — enough to build a lock without a separate ZooKeeper/etcd cluster.
- **S3 leader election (Morling)** makes Entry #018's theory concrete: per-**epoch** lock files give one leader per epoch (atomic PUT), the **epoch is the fencing token** (downstream must check it), a **`validity_ms` lease** provides liveness against crashed leaders, and correctness is **only eventual** because clock drift can transiently produce two leaders — so the fencing token, not the clock, is the real safety net. This is exactly "you can't guarantee one leader; you can only fence the stale one" from Entry #018.
