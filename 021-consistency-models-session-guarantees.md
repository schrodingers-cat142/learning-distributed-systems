# Consistency Models: Session Guarantees, Baseball, and Quorums

**Date:** 2026-07-30
**Sources:**
- [Eventually Consistent — Werner Vogels (CACM 2009)](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — client-side vs server-side consistency, the model taxonomy, CAP, and the N/W/R quorum model
- [Session Guarantees for Weakly Consistent Replicated Data — Terry, Demers, Petersen, Spreitzer, Theimer & Welch (Xerox PARC / Bayou, PDIS 1994)](https://csis.pace.edu/~marchese/CS865/Papers/SessionGuaranteesPDIS.pdf) — the four session guarantees and their version-vector implementation
- [Replicated Data Consistency Explained Through Baseball — Doug Terry (Microsoft Research, 2011)](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/10/ConsistencyAndBaseballReport.pdf) — six read guarantees and the "who is reading matters" lesson

**Related entries:**
- [020-replication-lag-problems.md](020-replication-lag-problems.md) — the three anomalies there (read-your-writes, monotonic reads, consistent prefix) are *symptoms*; this note is the formal *catalogue* of guarantees they map onto
- [016-single-leader-replication.md](016-single-leader-replication.md) — the sync/async knob there is one point on the consistency spectrum this note lays out in full
- [014-rum-conjecture.md](014-rum-conjecture.md) — the consistency/performance/availability tradeoff here is the same "pick your point, pay elsewhere" shape as RUM
- [005-facebook-tao.md](005-facebook-tao.md) — TAO deliberately sits at "read-after-write within a tier, eventual across tiers," an explicit choice from this menu

> **Note on scope:** This is the *conceptual* half of DDIA's replication-lag material — the full menu of consistency guarantees the Entry #020 anomalies live inside. It pulls together three classic sources into one map. On **quorums (N/W/R)** it stays deliberately *light* — just enough to see how the server-side knob produces these guarantees; the deep quorum treatment (sloppy quorums, read repair, anti-entropy) belongs with **leaderless (Dynamo-style) replication**, a later note, and is intentionally deferred here.

---

## Why a Whole Note on "Consistency Models"

Entry #020 walked through three concrete anomalies caused by replication lag and patched each one. But it left a bigger question hanging: those three aren't a random list — they're points on a *spectrum* of formally-defined guarantees that stretches from "you see nothing coherent" (eventual consistency) to "you always see the latest truth" (strong consistency). Practitioners and papers have been mapping this spectrum for thirty years, and understanding the map is what lets you answer the real design question: *which* guarantee does this particular read need, and what does it cost me?

Three classic sources, taken together, draw the whole map. **Vogels** gives the vocabulary and the split between how *clients* perceive consistency and how *servers* provide it. **Terry et al.'s Bayou paper** provides the rigorous foundation — it's literally where "read your writes" and "monotonic reads" were defined — plus two guarantees about *writes* that Entry #020 didn't touch, and a beautiful implementation using version vectors. And **Terry's baseball paper** delivers the punchline that ties it to practice: on a database of just two numbers, *which participant is reading matters as much as what they're reading.* This note weaves the three into one picture.

---

## Vogels' First Cut: Client-Side vs Server-Side Consistency

Vogels' most useful contribution is insisting that "consistency" is really *two* questions wearing one word, and confusing them is the source of endless muddle.

**Client-side consistency** is about what an *observer* sees: if process A updates a value, when and whether do processes B and C see it? This is the world of read-your-writes, monotonic reads, and the rest — it's about the *guarantees experienced by readers*.

**Server-side consistency** is about the *mechanics* underneath: how updates flow between replicas and what the replication protocol can promise. This is the world of quorums and N/W/R.

```
   CLIENT-SIDE ("what do I, the reader, observe?")
        A updates X.  When do B and C see it?
        → strong / eventual / causal / read-your-writes / monotonic / ...
                                │
                                │  is produced by
                                ▼
   SERVER-SIDE ("how do replicas coordinate?")
        N replicas, W must ack a write, R are read.
        → the W + R > N knob that dials the client-side guarantee up or down
```

The point of the split: the client-side guarantee you *experience* is a *consequence* of the server-side knobs the operators *set*. Keep them separate and the whole topic clarifies.

---

## The Backdrop: CAP and Why Any of This Is Necessary

Vogels grounds the entire discussion in the **CAP theorem** (Brewer's 2000 PODC keynote, formalized by Gilbert & Lynch in 2002): of the three properties **Consistency, Availability, and Partition-tolerance**, a distributed system can guarantee only *two* at once.

The catch is that in any real large-scale system, **network partitions are not optional** — links fail, datacenters get isolated, packets drop. So partition-tolerance isn't something you choose; it's a fact you must live with. That collapses the choice to a stark either/or during a partition:

```
   A partition happens (it will). You must pick:

   ┌─ Prioritize CONSISTENCY ──►  refuse operations on the wrong side of the
   │                              partition → reduced AVAILABILITY (writes fail)
   │
   └─ Prioritize AVAILABILITY ──► keep serving on both sides → replicas diverge,
                                  reads may not reflect recent writes (relax CONSISTENCY)
```

This is *why* the whole spectrum of weaker consistency models exists. If you refuse to sacrifice availability (as Amazon famously chose for the shopping cart — a "write-always" system that merges diverging carts after the partition heals), then you are *forced* to accept some form of weak/eventual consistency, and the interesting question becomes: how much can I claw back? That is exactly the same "you can't have everything, pick your point and pay elsewhere" shape as the RUM conjecture (Entry #14) — here the three-way tension is consistency vs availability vs performance rather than read vs update vs memory.

Vogels is also careful to separate this **CAP-consistency** (replicas agreeing) from **ACID-consistency** (a transaction leaving the database in a valid state) — same word, unrelated meaning. Worth keeping straight.

---

## The Client-Side Menu (Vogels' Taxonomy)

With the split and the CAP backdrop in place, here's Vogels' catalogue of client-side models, from strongest to weakest-plus-refinements:

- **Strong consistency:** after an update completes, *every* subsequent access (by anyone) returns the updated value. The gold standard; the most expensive.
- **Weak consistency:** no guarantee that subsequent reads see the update; there's a period — the **inconsistency window** — before it's visible.
- **Eventual consistency:** the specific, useful form of weak consistency — *if no new updates are made, eventually all replicas converge* to the last value. Vogels' canonical example is **DNS**: propagate updates through caches with time-controlled expiry until everyone eventually sees them.
- **Causal consistency:** if A tells B "I updated X," then B's later reads see the new X (and B's own write supersedes it); a process C with *no* causal link to the update follows ordinary eventual-consistency rules. This is the guarantee that would have prevented the consistent-prefix "answer before question" anomaly in Entry #020.
- **Read-your-writes consistency:** A always sees its own updates, never an older value. Vogels frames it as a *special case of causal consistency* (you are causally linked to your own writes). This is Entry #020's Anomaly 1.
- **Session consistency:** read-your-writes, but scoped to a **session** — the practical version. As long as the session lasts, you see your own writes; if the session ends (e.g., a failure), a new session's guarantees don't extend across the boundary. This is the bridge to the Bayou paper below.
- **Monotonic read consistency:** once a process has seen a value, it never later sees an *earlier* one. Entry #020's Anomaly 2.
- **Monotonic write consistency:** the system serializes writes from the *same* process in order. Vogels notes systems lacking this are "notoriously hard to program" — you can't reason about your own writes landing out of order.

His practical verdict, worth memorizing: **monotonic reads + read-your-writes together are the most desirable pair in practice** — they cover the anomalies that most confuse users, while still letting the storage system relax consistency elsewhere for availability. And a deflating aside: eventual consistency isn't exotic — any ordinary RDBMS doing *asynchronous* primary-backup replication (Entry #016) and allowing reads from the backup is *already* eventually consistent. You've probably been running it for years.

---

## The Rigorous Foundation: Terry et al.'s Four Session Guarantees (Bayou, 1994)

Vogels' taxonomy is the popular vocabulary; the Bayou paper is the peer-reviewed bedrock it rests on. Written at Xerox PARC for a system supporting *mobile* users who read and write from **different servers over time** (a laptop hopping between replicas), it defines the notion of a **session** — the sequence of reads and writes an application performs — and four guarantees that make a session behave *as if* it were talking to one consistent central server, even though it's bouncing between inconsistent replicas.

The framing insight is that inconsistency is most confusing *precisely when a single user is moving between servers*: you write to server S1, then read from server S2 that hasn't synced yet, and your own action seems to vanish. The four guarantees each close one such gap. Two concern **reads**, two concern **writes** — and the two write-side ones are exactly what Entry #020 (a reads-focused note) left out.

```
                    │  affects the SESSION's own view   │  also affects OTHER users
   ─────────────────┼───────────────────────────────────┼──────────────────────────
   about READS      │  Read Your Writes                  │  Monotonic Reads
                    │  (see your own writes)             │  (never see time go backward)
   ─────────────────┼───────────────────────────────────┼──────────────────────────
   about WRITES     │  Monotonic Writes                  │  Writes Follow Reads
                    │  (your writes apply in order)      │  (writes ordered after the
                    │                                     │   reads they depend on)
```

**1. Read Your Writes (RYW).** A read reflects all previous writes *in the same session*. If you change your password then log in, the login must see the new password. Formally: if read R follows write W in a session and R runs at server S at time t, then W must be in that server's database DB(S,t). The paper's own example is the Grapevine bug where users changed a password, immediately tried it, and got "invalid" — because the login hit a server that hadn't received the change. This is Entry #020's Anomaly 1, defined precisely.

**2. Monotonic Reads (MR).** Successive reads reflect a *non-decreasing* set of writes — later reads never show *less* than earlier ones. Their example: a calendar app refreshing its display shouldn't have meetings *appear and disappear* as it reads from replicas at different sync levels. This is Entry #020's Anomaly 2.

**3. Writes Follow Reads (WFR).** A write is ordered *after* any writes whose effects the session already read. This preserves *write-after-read dependencies* — the causal ordering of "I read this, and based on it, I wrote that." Their example: a bulletin board where a *reply* must be ordered after the *article* it replies to, for *everyone*, so nobody sees the reply before the original. This is the write-side sibling of consistent-prefix / causal consistency, and it's the guarantee Entry #020's "answer before question" anomaly really needed.

**4. Monotonic Writes (MW).** Writes from a session are applied in the order they were submitted, at every replica. Their example: a text editor saving version N then N+1 must not have N+1 land on one server and N on another and get reordered so N *overwrites* N+1. This is Vogels' "monotonic write consistency," the one whose absence makes systems "notoriously hard to program."

### Why "session" guarantees, and how they're implemented

The genius of the design is that these are provided *without any coordination between servers* and without changing the underlying weakly-consistent, read-any/write-any system. All the work happens in a **session manager** (part of the client stub) that mediates which server each operation is allowed to use.

The mechanism: every write gets a unique ID (a **WID**). The session manager tracks two sets of WIDs:
- a **write-set** — the writes this session has performed
- a **read-set** — the writes *relevant* to what this session has read

Then, before each operation, it enforces the guarantee by *only using a server that's caught up enough*:

```
   Before a READ  (for Read-Your-Writes):  pick a server whose database
                                            already contains the session's WRITE-SET.
   Before a READ  (for Monotonic Reads):   ...contains the session's READ-SET.
   Before a WRITE (for Writes-Follow-Reads):...contains the session's READ-SET.
   Before a WRITE (for Monotonic Writes):  ...contains the session's WRITE-SET.

   If no available server is caught up enough → the guarantee cannot be met
   (the availability cost — you may have to wait or fail).
```

Tracking raw WID sets would balloon, so the practical implementation compacts each set into a **version vector** — a sequence of ⟨server, logical-clock⟩ pairs, where entry V[S] means "seen all writes from server S up to clock value V[S]." Checking "is this server caught up enough?" becomes a cheap check that the server's version vector **dominates** (is ≥ in every component) the session's vector. The entire per-session state collapses to just **two version vectors** (one for reads, one for writes). This is the same version-vector machinery used elsewhere to detect conflicting writes — reused here to enforce session consistency.

The crucial tradeoff the paper is explicit about: restricting *which* servers a session may use **reduces availability** — if the only caught-up server is unreachable, you're stuck. So guarantees are requested **individually, per session**, and only where needed. That's the whole philosophy: don't buy consistency you don't need, because you pay for it in availability. (Underneath, updates still spread between servers by lazy **anti-entropy** — the gossip/rumor-mongering propagation we'll meet again in leaderless replication.)

---

## The Intuition Pump: Consistency Explained Through Baseball (Terry, 2011)

Seventeen years after Bayou, Doug Terry wrote a short, wonderful paper making all of this concrete with a database of *two integers*: the visitors' and home team's baseball scores. It defines **six** read guarantees (Vogels' models, repackaged for readers) and then asks the question that reframes everything: *which guarantee does each person watching the game actually need?*

### The six read guarantees

Each guarantee is defined simply by *which subset of past writes a read is allowed to see*:

| Guarantee | You see... |
|---|---|
| **Strong consistency** | all previous writes (the true current score) |
| **Eventual consistency** | *some* subset of previous writes (any past-or-nonsensical score) |
| **Consistent prefix** | an initial, in-order sequence of writes (a score that *really existed* at some point) |
| **Bounded staleness** | all writes older than time T (no more than, say, 15 minutes stale) |
| **Monotonic reads** | an increasing subset over successive reads (never go backward) |
| **Read my writes** | all writes *you* performed (renamed from "read your writes" to stress the reader's viewpoint) |

The **consistent prefix** definition is the sharp one: with a real score of 2-5 reached through a specific sequence, an *eventual* read could return a score like 1-0 *that never actually existed* (mixing the home count from one moment with the visitor count from another), whereas a *consistent-prefix* read only ever returns a score that genuinely occurred at some past instant — like database **snapshot isolation**. That's the difference between "stale but real" and "stale and fictional."

### The lesson: who is reading matters as much as what they read

Now the payoff. Terry walks through the people who read the score, and shows each needs a *different* guarantee:

```
   Official scorekeeper  →  Read My Writes      (only writer; reads own last score to +1 it —
                                                 gets strong-read effect cheaply, no majority quorum)
   Umpire (end of 9th)   →  Strong Consistency  (must know the exact current score to end the game)
   Radio reporter        →  Consistent Prefix   (never report a score that never existed) +
                            + Monotonic Reads    (don't report 2-5 then later 1-3 and look foolish)
   Sportswriter          →  Bounded Staleness    (dined an hour post-game → 1-hr bound guarantees final)
   Statistician          →  Strong + Read My Writes (final score + own running season total)
   Stat watcher (fan)    →  Eventual Consistency (a day-old season stat is totally fine)
```

Three deep points fall out of this:
- **The scorekeeper needs strongly-consistent *data* but not a strongly-consistent *read*.** Because he's the *only* writer, "read my writes" gives him the exact same result as a strong read — but far cheaper. A strong read must pessimistically consult a majority of servers in case *anyone* wrote; read-my-writes just needs *some* server that has *his* last write, and since the last run scored minutes ago, almost any server qualifies. **Application knowledge lets you buy strong-read correctness at weak-read cost.**
- **Sometimes you must combine guarantees** (the reporter needs consistent-prefix *and* monotonic-reads — neither alone suffices).
- **Different readers of the identical data want different consistency.** This overturns the common assumption that consistency is a property of the *data* ("bank data = strong, shopping-cart data = eventual"). Terry's two-integer database shows it depends as much on *who is reading and why* as on what the data is.

### The tradeoff table

Terry also scores each guarantee on the three axes, which is the practical decision aid:

| Guarantee | Consistency | Performance | Availability |
|---|---|---|---|
| Strong | excellent | **poor** | **poor** |
| Eventual | poor | **excellent** | **excellent** |
| Consistent prefix | okay | good | excellent |
| Bounded staleness | good | okay | poor |
| Monotonic reads | okay | good | good |
| Read my writes | okay | okay | okay |

The shape is unmistakable and is the whole point: **strong consistency sits at one corner with the worst performance and availability; eventual sits at the opposite corner with the best.** The four intermediate guarantees each occupy a distinct middle position — none strictly dominates another. Choosing a consistency model *is* choosing a point in this tradeoff space, which is precisely the RUM-conjecture-style reasoning from Entry #14, applied to consistency.

---

## The Server-Side Knob: N/W/R Quorums (Light Touch)

Finally, back to Vogels' *server-side* view — the mechanism that actually produces these guarantees. He models a replicated store with three numbers:

- **N** — the number of replicas holding a piece of data
- **W** — the **write quorum**: how many replicas must acknowledge a write before it's considered done
- **R** — the **read quorum**: how many replicas a read consults

The single rule worth carrying away:

```
   W + R > N   ⟹   the write set and read set always OVERLAP
                    → every read sees at least one replica with the latest write
                    → STRONG consistency

   W + R ≤ N   ⟹   read and write sets may MISS each other
                    → a read can hit only replicas that lack the write
                    → WEAK / EVENTUAL consistency
```

A couple of Vogels' examples make it concrete. A synchronous primary-backup RDBMS is N=2, W=2, R=1 → W+R=3 > 2 → strongly consistent. Switch the backup to *asynchronous* and read from it — N=2, W=1, R=1 → W+R=2 = N → **not** guaranteed consistent. That's the exact moment Entry #016's "async replication" becomes Entry #020's "replication lag anomalies," expressed in quorum arithmetic. Operators tune these knobs for their goal: fault tolerance (N=3, W=2, R=2), read-heavy scaling (large N, R=1), write-optimized-but-less-durable (W=1, propagate lazily), and so on.

That's as far as this note goes on quorums *by design*. The N/W/R model is really the beating heart of **leaderless (Dynamo-style) replication**, where it comes with a whole apparatus — sloppy quorums, hinted handoff, read repair, and anti-entropy — for making it work under partitions. That machinery deserves its own treatment and will get it in the leaderless replication note. Here, N/W/R is just the server-side dial that explains *how* the operators produce the client-side guarantees the rest of this note catalogued.

---

## How the Three Sources Fit Together

```
   VOGELS  ── vocabulary + the two-level split + CAP backdrop + N/W/R knob
      │        "here is the menu, and here is the server dial that produces it"
      │
   BAYOU (Terry et al. 1994) ── the rigorous foundation
      │        defines RYW, Monotonic Reads, + the two WRITE-side guarantees,
      │        implemented with version vectors, scoped to a SESSION
      │
   BASEBALL (Terry 2011) ── the intuition + the decision lens
               six read guarantees on a 2-integer DB; "WHO reads matters as
               much as what they read"; the consistency/perf/availability table
```

The through-line connecting all three (and back to Entry #020): **consistency is not one thing you have or lack — it's a spectrum of named, well-defined guarantees, and the engineering skill is choosing the *weakest* one that still satisfies each specific reader, because weaker generally means faster and more available.** Entry #020 showed the anomalies; this note is the menu you order from to avoid them, priced in performance and availability.

---

## Key Takeaways

- **Vogels splits consistency into client-side** (what a reader observes — strong / eventual / causal / read-your-writes / session / monotonic-read / monotonic-write) **and server-side** (how replicas coordinate — the N/W/R quorum knob). The client-side guarantee you experience is *produced by* the server-side settings.
- **CAP forces the issue:** partitions are unavoidable, so during one you must choose consistency *or* availability. Choosing availability (e.g., Amazon's always-writable cart) forces weak/eventual consistency — which is why the whole spectrum exists. (CAP-consistency ≠ ACID-consistency.)
- **Terry et al.'s Bayou paper is the formal foundation:** four **session guarantees** — Read Your Writes and Monotonic Reads (reads), plus **Writes Follow Reads** and **Monotonic Writes** (writes, which Entry #020 omitted) — making a session behave like one consistent server. Implemented cheaply with **version vectors** (per-session read-set + write-set), by restricting each operation to a sufficiently-caught-up server — which trades away some availability.
- **The baseball paper's lesson:** on a two-integer database, six read guarantees suffice, and *which participant is reading matters as much as the data* — scorekeeper needs read-my-writes, umpire needs strong, reporter needs consistent-prefix + monotonic, sportswriter needs bounded staleness, fan needs only eventual. Application knowledge (e.g., "I'm the only writer") can buy strong-read correctness at weak-read cost.
- **The consistency/performance/availability table** shows strong and eventual at opposite corners, with four non-dominating middle options — choosing a model is picking a point in that tradeoff space (the RUM-conjecture shape, Entry #14).
- **N/W/R quorums (light touch):** **W + R > N ⇒ strong consistency** (read and write sets overlap); **W + R ≤ N ⇒ eventual**. This is the server-side dial; its deep treatment (sloppy quorums, read repair, anti-entropy) is deferred to the leaderless-replication note.
