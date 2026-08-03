# Leaderless Replication (Dynamo-style)

**Date:** 2026-08-03
**Sources:**
- DDIA Chapter 5 (Replication) — leaderless replication, quorums, quorum-consistency limits, performance, multi-region
- [Shared-Nothing Architectures for Server Replication and Synchronization — Colin Breck (2023)](https://blog.colinbreck.com/shared-nothing-architectures-for-server-replication-and-synchronization/)

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — the model this departs from: no leader means no single write order, so conflicts and quorum reasoning move to the foreground
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — the N/W/R quorum model was introduced there with a light touch and explicitly deferred; this note is that promised deep dive
- [022-multi-leader-replication.md](022-multi-leader-replication.md) — leaderless is the third replication family; like multi-leader it must detect and resolve concurrent writes (the how is Entry #028)
- [028-time-clocks-concurrent-writes.md](028-time-clocks-concurrent-writes.md) — leaderless replication *detects concurrent writes* using the happens-before relation and version vectors, which that note covers in depth
- [014-rum-conjecture.md](014-rum-conjecture.md) — the quorum knobs (N, W, R) are a tunable tradeoff, the same "pick your point, pay elsewhere" shape

> **Note on scope:** This note covers the *mechanics* of leaderless replication — writing during failures, catching up, and the N/W/R quorum model (the deep dive promised in #021). *How concurrent writes are detected and ordered* (happens-before, version vectors) is the foundational Entry #028; this note leans on it and points there.

---

## The Third Replication Model

We've now seen two ways to replicate. **Single-leader** (Entry #016): one node accepts all writes and orders them; simple, no conflicts, but the leader is a bottleneck and a failover hazard. **Multi-leader** (Entry #022): several leaders accept writes; better for multi-datacenter and offline, but concurrent writes conflict and must be reconciled.

**Leaderless replication** is the third model, and it throws out the very idea of a leader. *Any* replica can accept a write directly from a client, and the client (or a coordinator acting on its behalf) sends each read and write to *several* replicas at once. It was a niche idea until Amazon's 2007 **Dynamo** paper revived it, after which it became the architecture of **Cassandra, Riak, and Voldemort** — often called "Dynamo-style" databases. (Note: Amazon's *DynamoDB* is a different, later system — we cover it separately. "Dynamo-style" refers to the open-source lineage Dynamo inspired.)

```
   SINGLE-LEADER          MULTI-LEADER              LEADERLESS
   ──────────────         ─────────────             ──────────
   one write order        several leaders,          NO leader; client writes to
   (no conflicts)         conflicts to merge        MANY replicas at once, reads
                                                     from MANY replicas at once
        │                       │                          │
   leader is a            conflict resolution        client/coordinator sends to
   bottleneck & SPOF      is the hard part           several nodes; quorum decides
```

The defining move: with no leader to impose an order, **the client stops relying on any single node.** Instead of "write to the leader, trust it," the rule becomes "write to *n* replicas, wait for *w* to confirm; read from *n* replicas, wait for *r* to respond." The overlap between those sets is what gives you consistency — and reasoning about that overlap is the heart of this note.

---

## Writing to the Database When a Node Is Down

Start with the scenario that makes leaderless shine. In single-leader replication, if the leader is down, **no writes can happen** until failover completes (Entry #018) — that whole fraught failover dance exists precisely because losing the leader stops writes. Leaderless replication sidesteps it entirely.

Suppose data is replicated across **three** nodes, and one is down for a reboot. A client writing a value simply sends the write to all three; two accept it, one (the down node) doesn't. If we've declared that **two out of three confirmations is enough**, the write succeeds — the client just ignores the fact that one node missed it. No failover, no interruption.

```
   Client writes X=6 to all 3 replicas:

         ┌─────────┐   ✓ ack
   ─────►│ Node A  │────────►
         └─────────┘
         ┌─────────┐   ✓ ack       client got 2 acks → WRITE SUCCEEDS
   ─────►│ Node B  │────────►        (it required only 2 of 3)
         └─────────┘
         ┌─────────┐   ✗ (down)
   ─────►│ Node C  │
         └─────────┘   ← missed the write; now STALE
```

But this creates an obvious problem: **Node C now holds a stale value.** When it comes back and a client reads from it, it will return the old data. If reads went to just one node, you'd sometimes get the stale answer. The fix is the reason reads *also* go to multiple nodes: a client reads from several replicas at once and uses **version numbers** attached to each value to determine which response is newest, discarding the stale ones. That's the first half of the solution. The second half is repairing the stale node so it doesn't stay wrong forever.

---

## Catching Up on Missed Writes

A node that missed writes (because it was down, partitioned, or slow) must eventually be brought back into sync. Dynamo-style systems use up to three mechanisms, and the *tradeoffs between them* — not just their definitions — are what matter operationally.

### Read Repair

When a client reads from several replicas in parallel and notices that one returned a stale value (an older version number than the others), it **writes the newer value back** to the stale replica. Repair happens *as a side effect of reading*.

```
   Client reads X from all 3:
      Node A → X=6 (version 3)   ┐
      Node B → X=6 (version 3)   ├─ client sees C is behind
      Node C → X=5 (version 2)   ┘
                    │
                    └──► client writes X=6 back to Node C  (READ REPAIR)
```

This is simple and self-healing, but it only fixes values that are *actually read*. A value nobody reads never gets repaired — so read repair alone lets rarely-read data drift.

### Hinted Handoff

When a write can't reach its intended node (it's down), a *different, reachable* node accepts the write on its behalf, storing it as a **hint** — a note saying "this really belongs to Node C; deliver it when C returns." Once C recovers, the hint-holder forwards the buffered writes to it, then discards the hints. The write is never lost, and the intended node catches up without a full re-sync.

```
   Node C is down. Client writes X=6.
      A neighbor (say Node D) accepts it with a HINT: "for C, deliver later."
              │
      C recovers ──► D hands off the buffered writes to C ──► C is caught up
```

**But hinted handoff has a nasty operational shape**, and this is where Colin Breck's hard-won experience is illuminating. He calls it **"inverted load-shedding"**: when a node fails, the *surviving* nodes must now work *harder* — they serve the failed node's share of traffic *and* buffer (journal) writes on its behalf. Exactly when the cluster is degraded and least able to cope, you pile *more* load on the healthy nodes. Worse, recovery causes an **IO storm**: when the failed node returns and the buffered writes flood back to it, the cluster is under *maximum* stress precisely while it's trying to heal. And the signal is noisy — hinted-handoff backlog growth "draws a lot of attention without always being actionable" (it might mean catastrophe, or just a routine restart backfilling), so operators learn to ignore it, which is its own hazard.

Breck's preferred alternative is worth knowing as the modern counterpoint: push writes to a **shared, durable, distributed journal** (Kafka-style) *before* database ingestion. Then a recovered node catches up by reading the journal **independently**, at its own pace, burdening no peer. The journal has its own scalable resource pool, can be shared across many consumers, and decouples the healthy nodes from the recovery. "It leaves the essential complexity of state replication to one system, rather than many." Breck credits DynamoDB and Cassandra for proving hinted handoff can be "incredibly reliable despite outright neglect," while noting InfluxDB deliberately *moved away* from it toward the shared-journal (shared-nothing) model. So: hinted handoff works, but it's operationally spiky, and there are cleaner designs for some workloads.

### Anti-Entropy

Read repair and hinted handoff both have gaps (unread data; hints that never get delivered if the holder itself fails). **Anti-entropy** is the background process that closes them: replicas periodically compare their full datasets with each other and copy over any differences, converging toward identical state. It's typically implemented with **Merkle trees** — hash trees that let two replicas efficiently find *just* the ranges that differ without transferring everything. Anti-entropy is slow and runs continuously in the background, but it's the safety net that guarantees eventual convergence even for data no one reads.

```
   Read repair    → fixes data that is READ        (fast, but misses cold data)
   Hinted handoff → fixes a node that was DOWN      (fast, but operationally spiky)
   Anti-entropy   → fixes EVERYTHING, eventually    (slow background sweep, the safety net)
```

Colin Breck's own system, incidentally, is a cautionary tale of what happens *without* these: it had "no native quorum reads, read-repair, or anti-entropy," so its replicas could "permanently diverge" — a deliberate simplicity tradeoff acceptable for his static industrial topology, but a reminder that these three mechanisms are what *earn* eventual consistency.

---

## Quorums for Reading and Writing

Now the core theory — the N/W/R quorum model, introduced with a light touch in Entry #021 and deferred to here. This is the machinery that decides "how many confirmations is enough."

Three numbers govern a leaderless system:

- **n** — the number of replicas each piece of data is stored on.
- **w** — the **write quorum**: how many replicas must acknowledge a write for it to count as successful.
- **r** — the **read quorum**: how many replicas must respond to a read for it to count as successful.

The single rule that makes it work:

```
   ┌──────────────────────────────────────────────────────────────┐
   │   w + r > n   ⟹   the write set and read set always OVERLAP   │
   │                                                                │
   │   → every read touches at least one node that saw the latest  │
   │     write → the read is guaranteed to see up-to-date data     │
   └──────────────────────────────────────────────────────────────┘
```

The intuition is pure pigeonhole. If a write reached `w` nodes and a read consults `r` nodes, and `w + r > n`, then the two sets *cannot* be disjoint — at least one node is in both, so the read is guaranteed to include at least one copy of the latest value. The reader then uses version numbers to pick the newest among the responses.

```
   n = 3, w = 2, r = 2  →  w + r = 4 > 3  ✓  (the common default)

        write set {A,B}          read set {B,C}
              ▼                        ▼
        ┌───┐ ┌───┐ ┌───┐
        │ A │ │ B │ │ C │        B is in BOTH sets → the read sees the write
        └───┘ └───┘ └───┘
              └── overlap ──┘
```

The classic configuration is **n=3, w=2, r=2**: it tolerates one node being down for either reads or writes while still guaranteeing overlap. You tune the knobs for your workload (the RUM-conjecture tradeoff shape, Entry #014):
- **Read-heavy:** small `r` (even `r=1`) so reads are cheap and fast — but then `w` must be large (`w=n`) to keep `w+r>n`, making writes need every replica.
- **Write-heavy / write-available:** small `w` so writes succeed even with nodes down — but `r` grows correspondingly.
- **`w + r ≤ n`:** you've given up the overlap guarantee — reads *may* miss the latest write — in exchange for lower latency and higher availability. This is a legitimate choice (it's just weaker consistency), and it's the regime where the leaderless-vs-single-leader consistency story from #021 lives.

---

## The Limitations of Quorum Consistency

Here's the part that trips people up, and where DDIA is emphatic: **`w + r > n` does not actually guarantee the strong consistency it appears to.** The pigeonhole argument is correct, but its guarantee is narrower than intuition suggests, and several edge cases quietly break it. It's worth walking through them, because "we set `w+r>n`, so we're consistent" is a dangerous half-truth.

**1. Sloppy quorums destroy the overlap.** To stay *available* during network partitions, many systems use a **sloppy quorum with hinted handoff**: if the client can't reach `w` of the *designated* `n` home nodes, it writes to `w` *other, reachable* nodes instead (which hold the writes as hints). This keeps writes succeeding — but now the write went to nodes *outside* the read set's home nodes, so a subsequent read of the home nodes may **not** overlap with it. The `w+r>n` math assumed the same `n` nodes for both; sloppy quorums violate that assumption. Availability is bought at the cost of the consistency guarantee.

**2. Concurrent writes still conflict.** If two clients write the same key *concurrently* (neither saw the other), the quorum doesn't order them — the nodes may receive them in different orders, and there's no leader to serialize. You're back to the concurrent-write problem of multi-leader (Entry #022), needing version vectors and conflict resolution (Entry #028). A quorum guarantees you *read* a recent value; it does *not* guarantee there's a single unambiguous "latest."

**3. A read concurrent with a write is ambiguous.** If a read happens *while* a write is still propagating, the read might see the new value on some replicas and the old on others — it's undefined which it returns.

**4. Partial write failure leaves things murky.** If a write succeeds on fewer than `w` nodes, it's reported as *failed* to the client — but the nodes that *did* accept it **don't roll back.** So a "failed" write may still be visible on some replicas, and a later read may or may not see it.

**5. Timing edge cases with node recovery.** If a node carrying a new value fails and is restored from a replica carrying an *old* value, the count of replicas holding the new value can drop below `w`, silently breaking the overlap condition.

```
   The trap:  "w + r > n"  looks like  "strong consistency"
   The truth: it's a PROBABILISTIC improvement, not a guarantee. Sloppy quorums,
              concurrent writes, read/write races, partial failures, and recovery
              timing all punch holes in it. Dynamo-style stores are EVENTUALLY
              consistent — quorums make staleness less likely, not impossible.
```

The takeaway DDIA drives home: leaderless quorum systems are designed to *tolerate* eventual consistency and to make stale reads *less frequent*, not to eliminate them. If you need real strong consistency, quorums alone won't give it to you — you need the stronger machinery of later chapters (linearizability, consensus). This is the honest counterweight to the tidy `w+r>n` rule.

---

## Single-Leader vs Leaderless Performance

Leaderless replication has a genuine *performance* advantage that's easy to miss, and it comes from how each model handles a **slow** node (as opposed to a *dead* one).

In single-leader (or any system that waits on specific nodes), one slow replica drags down every request routed through it. The insidious case is a **gray failure**: a node isn't cleanly *dead* (which failure detectors catch), but is *degraded* — high latency, dropping some requests, GC-pausing, a failing disk. It still answers, just slowly, so it isn't failed over, and it silently poisons tail latency.

Leaderless replication is naturally robust to this because of **request hedging** (and the quorum structure itself). Since a client sends each request to `n` nodes but only needs `w` (or `r`) responses, **it doesn't have to wait for the slowest ones** — as soon as enough fast nodes respond, the request completes; the stragglers are simply ignored. The slow node's latency never enters the critical path.

```
   Read with n=3, r=2:
      Node A → responds in 2 ms   ┐
      Node B → responds in 3 ms   ├─ got 2 responses → DONE at 3 ms
      Node C → responds in 800 ms ┘   (gray failure; slow — but IGNORED)

   The slow node cannot poison latency because we never wait for it.
```

**Request hedging** generalizes this: send a request to more nodes than strictly needed, take the first responses, cancel or ignore the rest. It trades a bit of extra work for dramatically better tail latency and immunity to gray failures — one of the underappreciated strengths of the leaderless model, and a reason it's favored for latency-sensitive workloads at scale.

---

## Multi-Region Operation

Leaderless replication extends to multiple regions/datacenters more gracefully than single-leader, precisely because there's no leader whose location privileges one region. Every replica can accept writes, so a client can write to its *local* region's replicas and get a fast local acknowledgment, while the write propagates to other regions in the background.

The `n` replicas are typically spread across regions, and the quorum can be configured so that a write only needs to wait for acknowledgments from *local* replicas (`w` counts local nodes), with cross-region replication happening asynchronously — keeping the write path fast while still eventually reaching every region. Cassandra and Riak support per-request tuning of how many replicas, and in which regions, must respond. This is the leaderless answer to the multi-region problem that multi-leader (Entry #022) solved with per-datacenter leaders and single-leader (Entry #016) struggled with (all writes crossing to one region). The tradeoff is the familiar one: local-only quorums are fast but a region can read stale data written in another region until replication catches up.

---

## Key Takeaways

- **Leaderless replication** (Dynamo-style: Cassandra, Riak, Voldemort) has *no leader* — any replica accepts writes, and clients read/write *multiple* replicas at once, using version numbers to identify the newest value. It sidesteps single-leader's failover problem: a write succeeds as long as enough replicas accept it, even with a node down.
- **Three catch-up mechanisms**, with different tradeoffs: **read repair** (fix stale data as a side effect of reads — misses cold data); **hinted handoff** (a reachable node buffers writes for a down node — but it's "inverted load-shedding," piling work on healthy nodes and causing an IO storm on recovery, per Colin Breck); **anti-entropy** (a slow background Merkle-tree sweep that converges everything — the safety net). Breck's modern counterpoint: a shared durable journal (Kafka-style) lets nodes catch up independently without burdening peers.
- **Quorums (N/W/R):** `n` replicas, `w` write acks required, `r` read responses required. **`w + r > n` ⟹ read and write sets overlap ⟹ reads see the latest write** (pigeonhole). Common default **n=3, w=2, r=2**. Tune for read-heavy (small r), write-available (small w), or weaker/faster (`w+r ≤ n`) — the RUM tradeoff (Entry #014).
- **Quorum consistency is weaker than it looks:** `w+r>n` is *not* a strong-consistency guarantee. **Sloppy quorums** (write to reachable non-home nodes for availability) break the overlap; **concurrent writes** still conflict (needs version vectors, Entry #028); read/write races, partial-write non-rollback, and recovery timing all punch holes. Dynamo-style stores are **eventually consistent** — quorums make staleness *less likely*, not impossible.
- **Performance edge:** because a client needs only `w`/`r` of `n` responses, it **ignores slow nodes** — giving natural immunity to **gray failures** (degraded-but-not-dead nodes) and enabling **request hedging** (send to extra nodes, take the fastest, ignore stragglers) for excellent tail latency.
- **Multi-region:** every replica accepts writes, so clients write locally and fast with local quorums while cross-region replication happens asynchronously — the leaderless answer to the multi-region problem, at the cost of cross-region staleness.
