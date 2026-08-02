# Eg-walker: Escaping the OT-vs-CRDT Tradeoff

**Date:** 2026-08-02
**Sources:**
- [Collaborative Text Editing with Eg-walker: Better, Faster, Smaller — Joseph Gentle & Martin Kleppmann (EuroSys 2025)](https://arxiv.org/pdf/2409.14252)

**Related entries:**
- [024-conflict-resolution-crdts-ot.md](024-conflict-resolution-crdts-ot.md) — this note is the direct sequel: Eg-walker is a hybrid that aims for OT's cheap memory *and* CRDTs' robust merging, escaping the dilemma that note laid out
- [023-local-first-offline-first.md](023-local-first-offline-first.md) — #023 flagged "CRDTs accumulate large history → performance problems" as a major *open problem*; Eg-walker (co-authored by Kleppmann, same as the local-first essay) is a direct answer, making peer-to-peer local-first competitive with the cloud
- [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) — the "store the log of events, derive state by replay" idea is event sourcing applied to collaborative text; the event graph *is* an append-only event log with a DAG shape
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — Eg-walker guarantees strong eventual consistency; the happened-before/concurrent relation reappears here

> **Note on scope:** This note explains the Eg-walker *algorithm* in detail with fully worked examples — it assumes you've read Entry #024 (OT vs CRDTs). Read this one to understand *how* a hybrid can get the best of both, traced step by step.

---

## The One Problem Every Collaborative Editor Must Solve

Strip away everything and a collaborative text editor has exactly one hard job: **when a remote edit arrives, at what index do I actually apply it to my current text?**

The edit was created against some *older* version of the document. Since then, other people's edits may have inserted or deleted characters ahead of it, shifting everything. So the raw index inside the operation is usually *wrong* for your current text. The entire algorithm — OT, CRDT, or Eg-walker — exists to **translate that operation's index into the correct index for your current document.**

Here is the problem in its smallest form (the paper's Figure 1). Two users both start with the typo `"Helo"`:

```
   User 1                              User 2
   "Helo"                              "Helo"
   Insert(3,"l")  →  "Hello"           Insert(4,"!")  →  "Helo!"
```

These edits are **concurrent** — neither user saw the other's. Now they sync, and User 1 receives User 2's operation `Insert(4,"!")`. If User 1 applies it *literally* at index 4 of their current text `"Hello"`:

```
   "Hello"  →  insert "!" at index 4  →  "Hell!o"     ✗ WRONG
```

Broken. User 1's own insertion of "l" at index 3 pushed everything from index 3 rightward, so User 2's "!" — meant to land *after* the "o" — must now go at index **5**:

```
   "Hello"  →  insert "!" at index 5  →  "Hello!"     ✓ CORRECT
```

**That `+1` shift is "the transformation."** OT computes it by comparing operations pairwise (and pays O(n²) when lots have diverged). A CRDT sidesteps it by giving every character a permanent unique ID (and pays in memory and load time forever). Eg-walker computes it a third way — by *replaying an event graph* — and the payoff is it gets OT's low memory and fast loads *together with* CRDT-grade merging of arbitrarily diverged branches.

Recall the dilemma from Entry #024, because Eg-walker's whole reason for existing is to break it:

```
   OT     → cheap memory, fast load, BUT merging long-diverged branches is O(n²)
            (the paper shows one real merge taking 1 HOUR)
   CRDT   → merges any branches robustly, BUT unique IDs + tombstones must live
            in memory, on disk, and on the wire forever → heavy + slow to load
            (best CRDTs use 10×+ OT's memory; hundreds of MiB in the paper)

   Eg-walker → aims for OT's cheapness AND CRDT's robustness, at once.
```

---

## The Core Structure: An Event Graph

Eg-walker's first move is to stop storing just "the current text" and instead store the document's **entire editing history as an event graph** — a DAG (directed acyclic graph). Every edit is an **event**:

```
   event = ( operation , unique ID , list of parent event IDs )
                │            │              │
         Insert(i,c) or   globally      the events the author had already
         Delete(i)        unique        seen when they made this edit
```

The parents are the load-bearing part. **An event's parents define exactly the document state in which its operation must be interpreted.** When someone typed `Delete(3)`, the parents tell you precisely which characters existed — and therefore which one sat at index 3 — at the moment they hit delete.

Figure 1 as an event graph:

```
        e1: Insert(0,"H")
          │
        e2: Insert(1,"e")
          │
        e3: Insert(2,"l")
          │
        e4: Insert(3,"o")        ← document is "Helo" here
         ╱          ╲
   e5: Insert(3,"l")   e6: Insert(4,"!")
```

`e5` and `e6` share parent `e4`, and neither is an ancestor of the other → they are **concurrent** (written `e5 ∥ e6`). Notice that *only the fork* has concurrency; the chain `e1→e2→e3→e4` is perfectly linear. This matters enormously later: **transformation work is needed only around forks, never on linear stretches.**

Some vocabulary that recurs (all from Lamport's happened-before, Entry #021):
- **happened-before** (`a → b`): a directed path runs from a to b.
- **concurrent** (`a ∥ b`): neither happened before the other.
- **version / frontier**: the set of "tip" events with no children — here `{e5, e6}`. A version is a logical clock: it names exactly which events a replica has seen.

Merging two replicas is just **taking the union of their event sets** — trivial. Reconstructing the text is a pure function **`replay(G)`**: topologically sort the graph into a line, transform each event, apply in order. *Any* valid topological order yields the *same* final text — that is the convergence guarantee, and it's why this is really event sourcing (Entry #007) applied to text: store what happened, derive state by replay.

---

## The Heart of the Trick: Two Versions at Once

Here is the central idea, and if you get this you get Eg-walker. During replay, the algorithm maintains an internal structure that tracks every character at **two different versions simultaneously**:

- **prepare version** — the document as it looked *when the current event was created* (the state defined by that event's parents). This is the world the operation's index refers to.
- **effect version** — the document including *everything replayed so far*. This is the world we actually want to apply the edit to.

```
   The transformation, in one sentence:

   Take the operation's index in the PREPARE version → find that character →
   read off the same character's index in the EFFECT version. That's the
   transformed index to apply.
```

If prepare and effect are **identical** (no concurrency around this event), the index is unchanged — zero work. They diverge only near concurrent forks. That's the whole reason Eg-walker is cheap: on the linear 99% of a typical document, prepare = effect and nothing happens.

To track this, every character record carries **two status fields**:

- `s_p` = status in the **p**repare version: one of `NotInsertedYet`, `Ins` (inserted/visible), or `Del n` (deleted by n concurrent deletes)
- `s_e` = status in the **e**ffect version: `Ins` or `Del`

And three operations slide the versions around:

```
   apply(e)    → bring event e into BOTH versions (move forward).
                 Insert: add a character record. Delete: mark one deleted.
   retreat(e)  → rewind the PREPARE version to before e ("pretend e didn't happen").
                 Only flips s_p flags. Effect version untouched.
   advance(e)  → the inverse of retreat: put e back into the prepare version.
```

The mental model: `s_e` is **"the truth so far, never rewound"**; `s_p` is **"a movable cursor in time"** that we slide backward and forward so it always matches the parents of whatever event we're about to process. Crucially, records are **never removed or reordered** once inserted — `retreat`/`advance` only toggle the status flags. (Eg-walker doesn't even store the actual letters in this internal structure; I'll write them below only so the trace is readable.)

---

## Worked Example 1: The Figure 4 Trace (the one that makes it click)

Two users start with `"hi"`. One turns `"hi"` → `"hey"` (delete "i", insert "e", "y"); concurrently another capitalizes (delete "h", insert "H"). They merge to `"Hey"`, then someone appends "!" → `"Hey!"`. The event graph, traversed in subscript order e1…e8:

```
        e1: Insert(0,"h")
        e2: Insert(1,"i")          ← document "hi"
        ╱                ╲
   e3: Insert(0,"H")    e5: Delete(1)      LEFT branch {e3,e4}: capitalize
   e4: Delete(1)        e6: Insert(1,"e")  RIGHT branch {e5,e6,e7}: "hi"→"hey"
                        e7: Insert(2,"y")
        ╲                ╱
        e8: Insert(3,"!")          ← merge; parents = {e4, e7}
```

The two branches are concurrent. `e8` sees both. I'll show the internal state as a row of records, each with `id / s_p / s_e`.

### Phase A — apply e1, e2 (build "hi")

Linear chain, prepare = effect, no transformation. Two records appear:

```
   char:   h        i
   id:     1        2
   s_p:    Ins      Ins
   s_e:    Ins      Ins
```
Effect version (chars where `s_e = Ins`) = `"hi"`. ✓

### Phase B — apply e3, e4 (left branch: insert "H", delete "h")

`e3 = Insert(0,"H")`: parent is e2, prepare already = {e2}, so no rewind. Insert "H" at front.
`e4 = Delete(1)`: at e4's moment the doc is `"Hhi"`, so index 1 = "h". Mark "h" deleted.

```
   char:   H        h          i
   id:     3        1          2
   s_p:    Ins      Del 1      Ins
   s_e:    Ins      Del        Ins
```
Effect version (chars where `s_e = Ins`) = `H, i` = `"Hi"`. ✓

### Phase C — the crux: apply e5, which lives on the *other* branch

`e5`'s parent is `e2`. That means e5 was created when the document was just `"hi"` — its author knew **nothing** of e3 or e4. So before we can read e5's index, we must **rewind the prepare version to `{e2}`** — pretend e3 and e4 never happened. We call `retreat(e4)` then `retreat(e3)`:

- `retreat(e4)`: e4 deleted "h"; undo that **in the prepare version only** → "h" `s_p`: `Del 1 → Ins`.
- `retreat(e3)`: e3 inserted "H"; undo it in prepare → "H" `s_p`: `Ins → NotInsertedYet`.

```
   char:   H            h        i
   id:     3            1        2
   s_p:    NotInsYet    Ins      Ins      ← PREPARE version now = "hi"  (H invisible here)
   s_e:    Ins          Del      Ins      ← EFFECT version UNCHANGED = "Hi"
```

**Stare at this state — it's the whole idea.** The two versions now disagree on purpose:
- **prepare** (chars where `s_p = Ins`): `h, i` = `"hi"` → exactly the world e5 was authored in.
- **effect** (chars where `s_e = Ins`): `H, i` = `"Hi"` → the real current document.

Now interpret `e5 = Delete(1)` in the **prepare** version `"hi"`: index 0 = "h", index 1 = "i". So e5 targets the **"i"** record. Then `apply(e5)` marks it deleted, and we read the **transformed** index from the *effect* version:

- In the effect version, visible chars were `H`(0), `i`(1) → "i" sits at effect-index **1**.
- **Transformed operation: `Delete(1)` applied to the real doc `"Hi"` → `"H"`.** ✓

```
   char:   H            h        i
   s_p:    NotInsYet    Ins      Del 1
   s_e:    Ins          Del      Del
```

That is the payoff: a `Delete(1)` authored against `"hi"` correctly became a `Delete(1)`-of-the-"i" against `"Hi"`, *not* a delete of the "H". The retreat/advance dance is what made the index mean the right thing.

### Phase D — apply e6, e7 (insert "e", "y")

e6, e7 are on the same right branch already in the prepare version — no more retreating.
`e6 = Insert(1,"e")`: prepare-visible chars are just `h` (H is NotInsertedYet, i is Del) = `"h"`; index 1 = after "h". Insert "e".
`e7 = Insert(2,"y")`: insert "y" after "e".

```
   char:   H            h        e        y        i
   id:     3            1        6        7        2
   s_p:    NotInsYet    Ins      Ins      Ins      Del 1
   s_e:    Ins          Del      Ins      Ins      Del
```
Effect version (chars where `s_e = Ins`) = `H, e, y` = `"Hey"`. ✓ Both branches have now merged: capitalization from the left, "hey" from the right.

*(Where exactly concurrent insertions at the same spot land — e.g. if two people insert at the same index — is decided by an embedded list-CRDT ordering rule, Yjs/YATA-style. That's the single place a real CRDT rule is needed, and it guarantees every replica picks the identical order.)*

### Phase E — apply e8 (append "!", parents = {e4, e7})

e8's author saw **both** branches (the merged "Hey"), so the prepare version must include e3 and e4 again. Right now they're retreated. We call `advance(e3)` then `advance(e4)`:

- `advance(e3)`: "H" `s_p`: `NotInsertedYet → Ins`.
- `advance(e4)`: "h" `s_p`: `Ins → Del 1`.

We do **not** retreat e5, e6, e7 — they're ancestors of e8 too and already applied, so they stay.

```
   char:   H        h        e        y        i
   s_p:    Ins      Del 1    Ins      Ins      Del 1     ← prepare = "Hey" (matches e8's parents)
   s_e:    Ins      Del      Ins      Ins      Del
```

Interpret `e8 = Insert(3,"!")` in prepare `"Hey"`: index 3 = after "y". Insert "!":

```
   char:   H        h        e        y        !        i
   s_p:    Ins      Del 1    Ins      Ins      Ins      Del 1
   s_e:    Ins      Del      Ins      Ins      Ins      Del
```
Effect version = `H, e, y, !` = **`"Hey!"`**. ✓ Every replica reaches this same state regardless of the order events arrived — strong eventual consistency.

---

## Making It Fast: O(log n) Index Math

In the trace above I "counted visible characters" by scanning the row — that's O(n) per operation, too slow for big documents. Eg-walker stores the records as the leaves of a **count-augmented B-tree** (an *order-statistic tree*): every internal node stores how many records beneath it have `s_p = Ins` and how many have `s_e = Ins`.

```
   To find "the i-th char visible in the PREPARE version":
      descend the tree, summing the s_p-counts of skipped subtrees → O(log n)
   To read the transformed index in the EFFECT version:
      walk UP, summing s_e-counts of subtrees to the left        → O(log n)
```

A second B-tree maps `event ID → record`, so `retreat`/`advance` (which target a specific event, not an index) are also O(log n). Net result: each `apply`/`retreat`/`advance` is O(log n), and **merging two branches of k and m events costs O((k+m) log(k+m))** — versus OT's O(n²). The 1-hour OT merge in the paper becomes **24 ms**.

---

## Why Memory Stays Tiny: Critical Versions

Everything above builds that internal `s_p`/`s_e` structure. If Eg-walker kept it forever, it would be as heavy as a CRDT — defeating the purpose. The escape hatch is the **critical version**.

A version is **critical** when there's no concurrency straddling it — *every* event in the whole graph is either at-or-before it, or strictly after it. In plain terms: **a moment when everyone was synced to one agreed state.** These happen constantly in real editing — every time collaborators take turns, every gap between edit sessions, always in a single-author document.

```
   In the Figure 4 graph:
      after e2 ("hi")     → CRITICAL (single line, everyone agrees)
      e3..e7              → NOT critical (the two branches straddle this region)
      e8 onward ("Hey!")  → CRITICAL again (the fork has closed)
```

Two optimizations exploit this, and together they're the reason Eg-walker is "Smaller" and "Faster":

1. **At a critical version, throw the entire internal structure away.** Nothing after a critical version depends on the transformation details before it, so both B-trees and all `s_p`/`s_e` values can be discarded. Memory does **not** grow over a document's lifetime — it resets to near-zero every time the doc is in an agreed state.

2. **On a linear stretch (event and its parents both at critical versions), skip the machinery entirely** — the transformed operation equals the original, so emit it as-is. No B-tree, no CRDT, no work.

The real-world consequence: for a document written by one author, or several taking turns — which the paper argues is the *majority* of real documents — nearly the whole history is a chain of critical versions, so Eg-walker does **almost no CRDT work and uses OT-like memory.** When a genuine concurrent merge does happen, it rebuilds the structure only back to *the most recent critical version before the fork* (not the whole history), using a **placeholder** record to stand in for the agreed — and irrelevant — prefix.

```
   Loading a document:
      CRDT       → must rebuild all ID metadata → as slow as replaying everything
                   (hundreds of ms to seconds)
      Eg-walker  → the current text is cached as a plain file; just read it
                   (~0.05 ms) — the event graph stays on disk until a merge needs it
```

---

## The Numbers (Real-World Payoff)

The authors built a production Rust implementation ("Diamond Types") and benchmarked against **Automerge** and **Yjs** (leading CRDT libraries), their own optimized reference CRDT, and an OT implementation, using real keystroke-level editing traces (sequential papers/blogs, concurrent co-writing, and *asynchronous* traces reconstructed from Git histories with long-running branches).

| Dimension | Eg-walker | Best CRDTs (Automerge / Yjs) | OT |
|---|---|---|---|
| **Merge a long-diverged branch (trace A2)** | **24 ms** | similar to Eg-walker's reference CRDT | **61 minutes** (~160,000× slower) |
| **Merge sequential traces** | fastest; 7–10× faster than ref CRDT | far slower | ~same as Eg-walker |
| **Load document from disk** | **~0.03–0.12 ms** (read cached text) | as slow as a full merge (100s ms–seconds) | ~same as Eg-walker |
| **Steady-state memory** | **hundreds of KiB** (233 KiB–1 MiB) | Automerge 294–848 MiB; Yjs ~20–30 MiB | ~same as Eg-walker |
| **Peak memory (during big merge)** | ~a CRDT's steady state (temporary spike) | — | can spike huge (6.8 GiB on A2, impl. detail) |

The honest cost: Eg-walker must **store the event graph** (compact columnar encoding borrowed from Automerge's format + Yjs bit-packing), typically **20%–3× the plain-text size**. That's the price of keeping full history — but it's smaller than the state a CRDT persists anyway, and it buys **free version history** (replay any subgraph to reconstruct any past state). Worst case merge complexity is O(n² log n) with a bad traversal order, but the topological-sort heuristic avoids it and it's unlikely in real histories.

---

## Why It Matters

Tie it back to the recent notes, because that's where the significance lives:

- **It removes the reason the industry avoids CRDTs.** Google Docs, Office, and Overleaf all use *centralized server-based OT* specifically to dodge CRDT memory/load costs (Entry #024). Eg-walker is, per the authors, "the first CRDT to match OT's memory use and performance on sequential editing histories, while avoiding the quadratic merge complexity that makes OT impractical for long-running branches." It collapses the #024 dilemma.
- **It makes peer-to-peer local-first competitive with the cloud** (Entry #023). It needs **no central server** and merges arbitrary branches cheaply — exactly the improvement local-first needed. #023's essay (same author, Kleppmann) flagged "CRDTs accumulate large history → performance problems" as a *major open problem*; this is the direct answer, targeting offline/P2P settings with no cloud at all (aircraft, field ops, remote science, slow-internet).
- **It brings Git-style branch/merge workflows to prose**, and — because it stores fine-grained history — gives **version history and time-travel for free**, connecting to the undo/history problems sync engines wrestle with (Entry #025).
- **It generalizes.** The event-graph + transient-CRDT framework isn't text-specific; the authors expect it to extend to rich text, spreadsheets, and graphics (a Figma-like tool, Entry #024).

---

## Key Takeaways

- **The one job** of a collaborative editor is translating a remote operation's index (created against an old state) into the correct index for your current text. That's "the transformation."
- **Eg-walker stores history as an event graph** (a DAG of events = operation + ID + parents). An event's **parents define the exact state its operation must be interpreted in.** State = the pure function `replay(G)`; merging = union of event sets; convergence is guaranteed for any topological order.
- **The core trick:** replay while tracking two versions at once — **prepare** (where the current event was born) and **effect** (everything so far) — via per-character status fields `s_p` and `s_e`. **`retreat`/`advance`** slide the prepare version backward/forward to match each event's parents (flags only, never moving records); **`apply`** moves both forward. The transformed index = the operation's index found in the prepare version, re-read in the effect version.
- **The Figure 4 trace** shows it: to apply `e5` (authored against `"hi"`) onto the real doc `"Hi"`, Eg-walker retreats e3 and e4 so prepare = `"hi"`, finds the "i" at index 1, then reads its effect index (also 1) — correctly deleting the "i", not the "H".
- **Speed:** count-augmented B-trees make each apply/retreat/advance **O(log n)**, so branch merges are **O((k+m) log(k+m))** vs OT's O(n²) — turning a 1-hour merge into 24 ms.
- **Memory:** the internal CRDT is **transient** — discarded at every **critical version** (a synced, no-straddling-concurrency point, which real editing hits constantly), and skipped entirely on linear stretches. So steady-state memory is 1–2 orders of magnitude below CRDTs, and loading is just reading a cached text file.
- **Significance:** OT's cheapness + CRDT's robustness in one algorithm — removing the reason apps avoid CRDTs, and making peer-to-peer, local-first, Git-for-prose collaboration practical (with free version history), at the cost of storing a compact event graph (20%–3× the text size).
