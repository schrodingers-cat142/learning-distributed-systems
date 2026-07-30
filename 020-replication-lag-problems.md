# Problems with Replication Lag

**Date:** 2026-07-30
**Sources:**
- DDIA Chapter 5 (Replication) — "Problems with Replication Lag": read-after-write, monotonic reads, consistent prefix reads, and solutions
- [Eventually Consistent — Werner Vogels (CACM 2009)](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — the inconsistency window and why async replication produces these anomalies

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — asynchronous replication (the common default there) is *precisely* what creates replication lag; this note is the price you pay for it
- [017-replication-logs.md](017-replication-logs.md) — the lag is the delay between a change entering the leader's log and a follower applying it
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — the formal menu of consistency guarantees these anomalies map onto (session guarantees, baseball, quorums); this note is the anomalies, that note is the taxonomy
- [005-facebook-tao.md](005-facebook-tao.md) — TAO's read-after-write via synchronous changesets, and its "read from a different tier and miss your write" caveat, are live instances of the problems and fixes here

> **Note on scope:** This note is the *practical* half of DDIA's replication-lag material — the concrete anomalies a user hits and how to fix them. The *conceptual* half — the formal catalogue of consistency guarantees (session guarantees, the baseball framing, quorum math) that these anomalies slot into — is Entry #021. Read this one for "what breaks and how to patch it"; read #021 for "the full menu of guarantees and why you'd choose each."

---

## Where This Fits

Entry #016 introduced a deliberate compromise: because fully synchronous replication is fragile, real systems very often replicate **asynchronously** — the leader acknowledges a write to the client immediately and lets followers catch up in their own time. That single decision buys performance and availability, and it hands you a bill. The bill is **replication lag**: the window during which a follower has *not yet* applied a write that the leader already has.

```
   t=0   Client writes X=1 to LEADER.  Leader acks immediately.
   t=0   Leader's log:    [... X=1]        ← has the write
         Follower's log:  [...    ]        ← doesn't have it YET
                                              └── this gap is the LAG
   t=?   (some milliseconds or seconds later) follower applies X=1.
```

Most of the time this lag is tiny — sub-second — and nobody notices. The trouble is that "eventually" is the only promise on offer. This is **eventual consistency** (Vogels): *if writes stop, all replicas eventually converge to the same value* — but it says nothing about what you see *during* the window Vogels calls the **inconsistency window**. And when lag occasionally spikes — a follower under load, a slow network, a follower catching up after a restart — that window stretches to seconds or minutes, and users hit genuinely confusing anomalies.

DDIA identifies three specific anomalies, each a distinct broken expectation, each with its own fix. They form a natural progression: the first is about *your own* writes, the second about *time appearing to move backward*, and the third about *cause appearing after effect*. We'll take them in that order.

---

## Anomaly 1: Reading Your Own Writes

The most jarring anomaly is submitting a write, then immediately reading it back — and seeing your write missing. You post a comment, the page reloads, and your comment is gone. You *know* you wrote it; the system seems to have swallowed it.

The mechanism is exactly the lag above: your write went to the leader, but your subsequent read was served by a follower that hasn't received it yet.

```
   1. You WRITE your comment  ──────────────►  LEADER   (accepted, acked)
                                                  │  async replication (lagging)
   2. Page reloads, you READ  ◄──────────────  FOLLOWER (hasn't got it yet)
        → your comment is MISSING.  "Did my post fail??"
```

The guarantee that fixes this is **read-after-write consistency** (also called *read-your-writes*): a user must always see updates *they themselves* submitted. Note the precise, limited promise — it says nothing about seeing *other* users' writes promptly. It only guarantees you're not confused about your *own* actions. That narrowness is what makes it cheap to provide.

### How to Provide It

DDIA gives several techniques, in rough order of increasing generality:

- **Read your own profile from the leader.** If a piece of data is something only its owner can edit (your profile, your comment), then read *that specific data* from the leader, and everything else from followers. This works because "things I might have just written" is a small, identifiable slice.
- **Track the time of the last write.** After a write, remember its timestamp (or the leader's log position). For a short window afterward, either serve that user's reads from the leader, or only serve them from a follower that has caught up *past* that position. The log-position idea is the same "know where you are in the log" mechanism from Entries #016–#017, used here for consistency rather than recovery.
- **Client remembers a version/timestamp.** The client carries the timestamp of its most recent write and the system ensures any replica serving it is at least that current — refusing or waiting on a stale replica.

TAO (Entry #5) is a real-world instance: when you write via a follower tier, the leader returns a **synchronous changeset** that the follower applies to its own cache *before* replying to you — so your very next read hits a cache that already contains your write. Read-after-write, engineered directly into the caching layer.

### The Cross-Device Wrinkle

There's a subtlety that trips up real systems: **the guarantee often needs to span devices.** You update your profile on your phone, then open your laptop expecting to see the change. From the system's perspective these are two different clients, and "read your writes" now has to recognize they're the *same user* across devices.

```
   PHONE   ── writes profile ──►  LEADER
                                     │  (lag)
   LAPTOP  ── reads profile  ──►  FOLLOWER  → stale?  "But I just changed this on my phone!"
```

This breaks the simpler fixes in two ways. First, "remember the timestamp of my last write" is stored *on the phone* — the laptop doesn't know about it, so the client-side approach needs **centralized** tracking of the user's latest write, not per-device. Second, if your two devices are routed to *different datacenters or replicas* (phone via one, laptop via another), you must route *both* to a replica that's current for that user — which requires the routing layer to key on the *user*, not the connection. Cross-device read-after-write is strictly harder, and it's a case worth remembering because it's easy to ship a fix that works on one device and silently fails across two.

---

## Anomaly 2: Monotonic Reads — Time Appearing to Move Backward

The second anomaly appears when a user makes *several* reads in a row and is served by *different* followers with *different* amounts of lag. They can see data, then see *older* data — time appearing to run backward.

The canonical example: you load a page and see a comment thread with a new reply. You refresh, and the reply *vanishes* — because the first read hit an up-to-date follower and the second hit one that's further behind.

```
   Read #1  ──►  Follower A (fresh)      → sees the new reply       "Oh, a reply!"
   Read #2  ──►  Follower B (lagging)    → reply is GONE            "...where'd it go?"
                                                                     time went BACKWARD
```

This is a *weaker* violation than reading your own writes — the data will still converge — but it's deeply disorienting because it contradicts the basic intuition that time only moves forward. The guarantee that prevents it is **monotonic reads**: if a user makes a sequence of reads, they will never see time go backward — a later read never returns *older* data than an earlier read. (Note it's a weaker promise than strong consistency: you can still be reading stale data, just never data that's *staler than what you already saw*.)

### How to Provide It

The standard technique is **sticky routing**: ensure each user always reads from the *same* replica, chosen deterministically — e.g., by hashing the user ID to a replica rather than picking one at random per request. If you always read from the same follower, you can't jump to a more-lagged one, so you can't go backward. (If that replica fails, the user must be rerouted, and care is needed so the new one isn't behind the old.) This is exactly why Vogels notes that achieving these session-scoped guarantees "depends largely on client stickiness to a given server."

---

## Anomaly 3: Consistent Prefix Reads — Cause After Effect

The third anomaly is the most conceptually interesting because it's about **causality and ordering**, not staleness. It shows up when different pieces of data replicate at different speeds, so an observer sees an *effect* before its *cause* — a violation of the order in which things actually happened.

DDIA's example is a conversation. Mr. Poons asks, "How far into the future can you see, Mrs. Cake?" and she answers, "About ten seconds usually." If these two writes land on partitions that replicate at different rates, a third-party observer can see the *answer* before the *question*:

```
   Actual order:   Q: "How far can you see?"  →  A: "About ten seconds."

   Observer sees:  A: "About ten seconds."     ← arrives first (fast partition)
                   Q: "How far can you see?"    ← arrives second (slow partition)

   → The answer precedes the question. Cause after effect. Nonsense.
```

The guarantee that prevents this is **consistent prefix reads**: if a sequence of writes happens in a certain order, then anyone reading them sees them in that same order — you always see a *prefix* of the true history, never a version with a later write but a missing earlier one. You might be behind, but what you *do* see is a coherent point-in-time snapshot, not a scrambled mix.

### Why This One Is Different — and Why Partitioning Causes It

The critical detail is that this problem specifically arises when data is **partitioned (sharded)**. In a single-leader system with *one* partition, writes go through the leader in a definite order, and that order is preserved in the log every follower replays — so a consistent prefix comes for free. But when different pieces of data live on *different partitions*, each partition has its own independent leader and its own log, and there's **no global ordering** across them. Partition A (holding the question) and partition B (holding the answer) replicate independently, so their relative order can be scrambled at an observer.

```
   Partition A (question)  ──log──►  followers   ┐
                                                  ├─ replicate INDEPENDENTLY,
   Partition B (answer)    ──log──►  followers   ┘  so cross-partition order can invert
```

The fix is genuinely harder than the previous two. There's no cheap sticky-routing trick; you need writes that are *causally related* to be either kept on the same partition (so one log orders them) or tracked with explicit causal metadata so the reader can enforce the order. This is a first taste of the **causality** problem that runs much deeper in distributed systems — and it's the anomaly that most motivates the stronger consistency models in Entry #021.

---

## Solutions for Replication Lag: The Bigger Picture

Stepping back from the three individual fixes, DDIA makes a broader point worth internalizing. All the techniques above — read-from-leader, sticky routing, tracking timestamps, carrying versions — share an awkward property: **they push the burden of dealing with lag up into the application.** The application developer has to remember which reads need which guarantee, route accordingly, and get all the edge cases right (the cross-device wrinkle being a prime example of one that's easy to miss).

```
   THE MENU, from "cheap but you do the work" to "expensive but it just works":

   Eventual consistency + hand-rolled fixes                  Stronger guarantees
   ───────────────────────────────────────────────────────────────────────────►
   read-your-own from leader                                 transactions /
   sticky routing per user                                   strong consistency /
   track write timestamp / version                           "just use a single-leader
   keep causal writes on one partition                        DB with real guarantees"
   (you manage each anomaly by hand)                         (the DB handles it for you)
```

This leads to the pragmatic conclusion. These patches are fine when applied carefully to the few places that need them, and they preserve the performance and availability that made you choose async replication in the first place (Entry #016). But if your application needs these guarantees pervasively, hand-rolling them everywhere is fragile and error-prone — and at that point the honest move is to **let the database provide stronger guarantees** (transactions, or a system offering the consistency level you need) rather than reimplementing consistency badly in application code. This is the same lesson as "What Goes Around" (Entry #3): people who abandon a strong guarantee for performance often end up re-implementing it, worse, in their app layer.

The deeper framing — *which* guarantee each situation actually needs, and the full spectrum between "eventual" and "strong" — is exactly what Entry #021 lays out. The three anomalies here are the symptoms; that note is the diagnostic manual.

---

## Key Takeaways

- **Replication lag is the cost of asynchronous replication** (Entry #016): the window where a follower hasn't yet applied a write the leader already has. **Eventual consistency** promises only that replicas *eventually* converge — nothing about what you see during Vogels' **inconsistency window**.
- **Read-after-write (read-your-writes):** you must see *your own* writes. Broken when your read hits a follower that lacks your just-submitted write. Fixed by reading your own data from the leader, tracking your last write's timestamp/log-position, or carrying a version. The **cross-device** case is harder — tracking must be centralized (not per-device) and routing must key on the user.
- **Monotonic reads:** you must never see time go *backward* across successive reads. Broken when consecutive reads hit followers with different lag. Fixed by **sticky routing** — always read from the same replica (e.g., hash the user ID).
- **Consistent prefix reads:** you must never see an *effect before its cause*. Broken specifically when data is **partitioned**, because independent partition logs have no global order. Fixed by keeping causally-related writes on one partition or tracking causal metadata — a first taste of the deeper causality problem.
- **The meta-lesson:** these fixes push lag-handling into application code and are fine in moderation, but if you need these guarantees everywhere, stop hand-rolling consistency and **let the database provide stronger guarantees** (transactions / stronger consistency). The formal menu of those guarantees is Entry #021.
