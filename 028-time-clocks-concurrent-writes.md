# Time, Clocks, and Detecting Concurrent Writes

**Date:** 2026-08-03
**Sources:**
- DDIA Chapter 5 (Replication) — detecting concurrent writes, the happens-before relation, version vectors
- [Time, Clocks, and the Ordering of Events in a Distributed System — Leslie Lamport (CACM 1978)](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/Time-Clocks-and-the-Ordering-of-Events-in-a-Distributed-System.pdf)
- [Clocks and Causality — Ordering Events in Distributed Systems — Giridhar Manepalli (2022)](https://www.exhypothesi.com/clocks-and-causality/)

**Related entries:**
- [027-leaderless-replication.md](027-leaderless-replication.md) — leaderless quorums *detect* concurrent writes but don't order them; this note is the how (version vectors), which #027 deferred here
- [022-multi-leader-replication.md](022-multi-leader-replication.md) — multi-leader's write-conflict problem is exactly "which writes are concurrent?"; LWW and version vectors both appear there and are grounded here
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — the happens-before relation underpins causal consistency and the session guarantees there
- [024-conflict-resolution-crdts-ot.md](024-conflict-resolution-crdts-ot.md) — CRDTs use logical clocks as their foundation; "strong eventual consistency" rests on the causality machinery here
- [026-eg-walker.md](026-eg-walker.md) — Eg-walker's event graph *is* a happened-before DAG; it cites Lamport directly

> **Note on scope:** This is a *foundational* note. The happens-before relation and version vectors have been quietly assumed since #021, #022, #024, #026, and #027 — this note is finally their home. It builds from Lamport's 1978 partial order → logical clocks → vector clocks → version vectors → detecting concurrent writes in a database.

---

## Why "Time" Is the Wrong Tool

Every replication model that lets more than one node accept writes (multi-leader #022, leaderless #027) hits the same question: given two writes to the same key, **did one happen before the other, or were they concurrent?** If one genuinely happened after the other, the later one should win — it's an *update*. If they were concurrent — neither author knew about the other — it's a *conflict* that needs merging or sibling-keeping, and silently discarding one is data loss.

The tempting answer is "just use timestamps — whichever has the later time wins." This is **last-write-wins (LWW)**, and Entry #022 already flagged why it's dangerous. The root problem, which Lamport's paper opens with and the Clocks & Causality article hammers, is that **wall-clock time cannot reliably order events in a distributed system:**

- Clocks on different machines **drift** and are never perfectly synchronized (NTP helps but leaves millisecond-to-second skew).
- So timestamps from different nodes are "not always mutually comparable" — a write that *actually* happened later can carry an *earlier* timestamp and wrongly lose.

Lamport's foundational insight (1978): if a system's correctness is specified in terms of events, that specification "must be given in terms of events observable within the system," not physical time — because real clocks aren't accurate. So he defines ordering **without physical clocks at all**, using only the one thing the system can actually observe: which events could have influenced which.

---

## The Happens-Before Relation (Lamport, 1978)

Lamport defines a relation "happened before," written **`→`**, as *the smallest relation satisfying three conditions*:

```
   1. If a and b are events in the SAME process, and a comes before b,  → a → b
   2. If a is the SENDING of a message and b is its RECEIPT,            → a → b
   3. If a → b and b → c, then                                          → a → c  (transitive)
```

And the definition that everything hinges on:

> Two distinct events `a` and `b` are **concurrent** (written `a ∥ b`) if `a ↛ b` **and** `b ↛ a` — neither happened before the other.

The deep reading Lamport gives: **`a → b` means it is *possible* for `a` to causally affect `b`.** Two events are concurrent when *neither can causally affect the other*. This is causality expressed purely through what information could have flowed — no clocks required. (Lamport notes the analogy to special relativity, where event ordering is defined by which events *could* exchange light signals.)

He visualizes it with a **space-time diagram** — vertical lines are processes, dots are events, wavy lines are messages. `a → b` iff you can trace a path from `a` to `b` moving upward along process lines and forward along message lines:

```
   time
    ↑        P              Q              R
    │        │              │              │
    │       p3 ─────msg────► q4            │        p3 → q4 (message)
    │        │              │              │        q4 → q5 → q6 (same process)
    │       p2             q3 ────msg────► r3       q3 → r3 (message)
    │        │              │              │
    │       p1             q2             r2        p2 ∥ q3 : CONCURRENT
    │        │              │              │        (no path connects them —
    │        └──            └──            └──        neither could affect the other)
```

Crucially, `→` is only a **partial order**: many pairs of events are simply incomparable (concurrent). Lamport's whole point in the opening is that "problems often arise because people are not fully aware of this fact" — they assume a global total order of events that doesn't exist.

---

## Lamport Logical Clocks

Lamport then asks: can we assign a *number* `C(a)` to each event that respects `→`? He wants the **Clock Condition**:

```
   Clock Condition:   if a → b   then   C(a) < C(b)
```

Note the one-directional arrow: `a → b` implies `C(a) < C(b)`, but **not the converse** — `C(a) < C(b)` does *not* imply `a → b` (they might be concurrent and just happen to get different numbers). Hold onto that; it's the source of the clock's key limitation.

He satisfies it with two implementation rules — a clock `Ci` is just a counter at each process `Pi`:

```
   IR1:  Each process increments its own clock between successive events.
   IR2:  (a) When Pi sends a message m, it stamps it with Tm = Ci (its current time).
         (b) When Pj receives m, it sets Cj = max(Cj, Tm) + 1
             (advance past both its own clock and the message's timestamp).
```

A worked trace across three processes, all starting at 0:

```
   A:  a1[1]      a2[2] ──msg(T=2)──┐
   B:  b1[1]                        └─► b2 = max(1,2)+1 = [3]   b3[4]
   C:                    c1[1]  c2[2]

   Learning of A's event (T=2) bumped B to 3 — B now "knows" it's causally after a2.
```

This is O(1) space (one number) and beautifully simple. **But it has a fatal limitation for conflict detection**, spelled out by the Clocks & Causality article: because the converse doesn't hold, **a Lamport clock cannot distinguish causality from concurrency.** If you see event `[5]` on one node and event `[4]` on another, you *cannot tell* whether the `[5]` causally followed that `[4]` or whether they were concurrent and just landed on different numbers. Two concurrent events can even get the *same* number.

### Total Order (and where LWW quietly comes from)

Lamport shows you can *extend* the partial order to a **total order** `⇒` by breaking ties with an arbitrary ordering of process IDs:

```
   a ⇒ b  iff  either  C(a) < C(b),
               or       C(a) = C(b) and Pi < Pj  (tie-break by process ID)
```

This gives a single, consistent, system-wide ordering of *all* events — which he uses to build a **distributed mutual-exclusion algorithm** (every process keeps a request queue ordered by `⇒`; a process enters the critical section when its request is at the head and it has heard from everyone with a later timestamp — no central coordinator). It's the ancestor of state-machine replication.

But notice: this total order is **arbitrary for concurrent events** — it *invents* an order for events that were genuinely concurrent, purely to have a deterministic tiebreak. That's exactly what LWW does in a database: it forces a total order on concurrent writes (by timestamp, then node ID) and calls the "loser" overwritten. It converges, but it **silently discards concurrent data** — because a Lamport-style total order can't tell "genuinely later" from "concurrent, tiebroken." This is the theoretical root of why LWW loses writes (Entry #022).

### Anomalous Behavior and the Strong Clock Condition

Lamport is honest about a further gap. His logical clock only captures causality *visible inside the system*. His example: you issue request A on computer A, then phone a friend who issues request B on computer B. In reality A caused B — but that causal link traveled *outside the system* (the phone call), so the system's clocks might order B before A. He calls this **anomalous behavior**.

Two fixes: (1) push the external ordering info into the system (have the user carry A's timestamp to B), or (2) use real physical clocks satisfying the **Strong Clock Condition** (`a → b ⟹ C(a) < C(b)` for *all* events including external ones) — which requires physical clocks synchronized to within a bounded skew. Lamport derives the synchronization bound in the paper's second half. The lesson for us: **logical clocks capture only in-system causality**; genuine external causality needs either explicit propagation or synchronized physical clocks. (This is also why modern systems like Google's Spanner use tightly-synchronized physical clocks — TrueTime — for stronger guarantees; a thread we'll pick up later.)

---

## Vector Clocks: Detecting Concurrency

The limitation of Lamport clocks — can't distinguish causality from concurrency — is exactly what a database needs to fix, because that distinction *is* "is this a conflict or an update?" **Vector clocks** solve it, at the cost of more space.

Instead of one number, each node keeps a **vector** — one entry per node — tracking the latest event it knows about from *every* node:

```
   Vector clock rules (n nodes, each entry starts at 0):
   • On a local event, a node increments ITS OWN entry.
   • On sending, it attaches its whole vector.
   • On receiving vector V, it sets each entry to max(own[i], V[i]),
     then increments its own entry.
```

The comparison rule is where the magic is — compare **entry by entry**:

```
   Given vectors V and W:
   • V = W in every entry              → same event
   • V ≤ W in every entry (and V ≠ W)  → V happened-before W   (V → W)
   • W ≤ V in every entry (and V ≠ W)  → W happened-before V   (W → V)
   • some entries V<W AND some V>W      → CONCURRENT  (V ∥ W)   ← the key case!
```

A worked example (3 nodes, from the Clocks & Causality article). Node B produces `[3, 4, 0]` — meaning "I've seen A's first 3 events, this is my 4th, and I've seen nothing from C":

```
   [3, 4, 0]  vs  [4, 5, 2]  → every entry of the first is ≤ the second
                               → [3,4,0] happened-before [4,5,2]   (causal)

   [3, 4, 0]  vs  [0, 2, 2]  → first has entry1=3 > 0, but entry3=0 < 2
                               → some greater, some smaller
                               → CONCURRENT  (C did something B never saw,
                                              and vice versa) — a real conflict
```

That "some-greater-and-some-smaller" pattern is impossible to detect with a single Lamport number, and it's precisely the signal a replicated database needs: **it can now tell a stale-and-superseded write (causally dominated → safe to discard) apart from a genuinely concurrent write (a conflict → must be kept/merged).** The cost is O(n) space (a vector sized by the number of nodes) and O(n) comparison — usually a fine trade for correctness. (Refinements exist: *dotted version vectors* track the last event separately for O(1) comparison; the article covers these, but the entry-by-entry idea is the core.)

---

## Version Vectors: Applying This to Database Writes

Now the payoff for replication. DDIA's "detecting concurrent writes" is exactly vector clocks applied to *data versions* rather than *events* — a usage the Clocks & Causality article is careful to distinguish (and notes people wrongly conflate the two). This is the **version vector**.

The setup, for a leaderless (#027) or multi-leader (#022) store:
- Each replica keeps a **version vector** per key — a version number per replica.
- On a write, a client must send back the **version(s) it read** (the "context"/causal history it's basing its write on).
- The server compares: is the incoming write causally *after* everything it has (an update), or *concurrent* with some existing version (a conflict)?

```
   Client reads key K → gets value + version vector context {A:2, B:1}
   Client writes K back WITH that context {A:2, B:1}
        │
        ▼
   Server checks the write's context against current versions:
     • context ≥ all current versions → this write SUPERSEDES them → replace
     • context is CONCURRENT with some version → CONFLICT → keep BOTH as siblings
```

The **shopping-cart example** DDIA uses makes it concrete. Two devices concurrently add different items to a cart. Neither write's context includes the other, so the version vectors are concurrent → the server keeps **both versions as siblings**. On the next read, the client receives both and must **merge** them (union the carts) and write back the merged value with a context covering both — which then supersedes both siblings. This is why "add to cart" rarely loses items even under concurrency: concurrent writes become siblings, not overwrites.

```
   Phone:   cart = {milk}        writes with context {}
   Laptop:  cart = {eggs}        writes with context {}     ← concurrent, disjoint contexts
        │
        ▼
   Server keeps siblings: {milk} AND {eggs}
        │
   Next read returns both → client MERGES → {milk, eggs} → writes back
        │
        ▼
   merged write's context covers both siblings → supersedes them → cart = {milk, eggs} ✓
```

The two resolution strategies, now grounded:
- **LWW** — force a total order (Lamport-style tiebreak), keep one, discard the rest. Simple, converges, **loses data**. Fine only when losing concurrent writes is acceptable.
- **Keep siblings + merge** — use version vectors to *detect* concurrency, preserve all concurrent versions, and merge them (in the app, or automatically via CRDTs, Entry #024). No data loss. This is what version vectors *enable* and what LWW throws away.

Version vectors are the connective tissue across this whole repo: they're the "concurrency detection" behind multi-leader conflict handling (#022), the sibling mechanism in leaderless stores (#027), the causal foundation of CRDTs (#024), the machinery in Bayou's session guarantees (#021), and conceptually the parents-list in Eg-walker's event graph (#026).

---

## The Ladder of Clocks (Summary)

```
   Wall-clock time      → NOT comparable across nodes (drift). LWW on it loses data.
        │
   Lamport clock O(1)   → respects happens-before, BUT can't tell causality from
        │                 concurrency (converse fails). Total order = arbitrary
        │                 tiebreak on concurrent events = the root of LWW.
        │
   Vector clock O(n)    → DETECTS concurrency (entry-by-entry: some-up-some-down).
        │                 Distinguishes "stale, discard" from "concurrent, conflict."
        │
   Version vector O(n)  → vector clocks applied to DATA VERSIONS. The database
                          mechanism for detecting concurrent writes → keep siblings,
                          merge, no data loss.
```

Each rung buys more discrimination at more cost. Lamport clocks are enough to *order* events into a consistent sequence (mutual exclusion, state-machine replication); vector/version vectors are needed to *detect concurrency* and thus handle conflicts without losing data. Choosing between them is, once more, a tradeoff — the recurring shape of this entire repo.

---

## Key Takeaways

- **Wall-clock time can't order events in a distributed system** — clocks drift, so timestamps aren't reliably comparable, and LWW built on them silently loses writes. Lamport's 1978 insight: define ordering from *observable events*, not physical time.
- **Happens-before (`→`)** is a *partial* order defined by three rules (same-process order; send→receive; transitivity). `a → b` means "a could causally affect b"; **concurrent** (`a ∥ b`) means neither could affect the other. Many event pairs are simply incomparable — the mistake is assuming a global total order exists.
- **Lamport logical clocks** (IR1: increment locally; IR2: on receive, take max+1) satisfy the Clock Condition `a→b ⟹ C(a)<C(b)`. But the **converse fails**, so they **cannot distinguish causality from concurrency**. Extending to a total order requires an arbitrary process-ID tiebreak on concurrent events — which is precisely the root of LWW's data loss.
- **Anomalous behavior:** logical clocks capture only *in-system* causality; external causal links (Lamport's phone-call example) escape them. Fixes: propagate ordering explicitly, or use physical clocks meeting the **Strong Clock Condition** (bounded-skew synchronization).
- **Vector clocks** (one entry per node; compare entry-by-entry) **detect concurrency**: if some entries are greater and some smaller, the events are concurrent — a conflict; if all ≤, one causally precedes — safe to supersede. O(n) space is the price for this discrimination.
- **Version vectors** apply vector clocks to *data versions*: clients read-and-return a causal context, the server keeps **concurrent writes as siblings** rather than overwriting, and the app (or a CRDT) merges them — the shopping-cart pattern, with **no data loss**. This is the concurrency-detection machinery under multi-leader (#022), leaderless (#027), CRDTs (#024), and session guarantees (#021).
