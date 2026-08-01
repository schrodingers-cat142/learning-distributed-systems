# Conflict Resolution: CRDTs and Operational Transformation

**Date:** 2026-08-01
**Sources:**
- DDIA — dealing with conflicting writes: conflict avoidance, last-write-wins, manual resolution, automatic resolution (CRDTs, OT)
- [What's different about the new Google Docs: Making collaboration fast — John Day-Richter (Google, 2010)](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html) — the OT collaboration protocol
- [How Figma's multiplayer technology works — Evan Wallace (2019)](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/) — a CRDT-inspired approach
- [Realtime editing of ordered sequences (fractional indexing) — Figma](https://www.figma.com/blog/realtime-editing-of-ordered-sequences/)

**Related entries:**
- [022-multi-leader-replication.md](022-multi-leader-replication.md) — introduced the four conflict strategies at a high level; this note goes deep on the fourth (automatic resolution) and its two techniques
- [023-local-first-offline-first.md](023-local-first-offline-first.md) — named CRDTs as local-first's enabling technology; this note is the mechanics that entry deferred
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — CRDTs provide *strong eventual consistency*, a specific point on that note's consistency spectrum; version vectors reappear here
- [025-sync-engines.md](025-sync-engines.md) — the architecture that ships these merged changes around; OT/CRDT is the merge logic, sync engines are the plumbing

> **Note on scope:** This note goes deep on the two automatic-conflict-resolution techniques — **operational transformation (OT)** and **CRDTs** — and grounds each in a real system (Google Docs for OT, Figma for CRDT-inspired). The surrounding *philosophy* is Entry #023; the *sync-engine architecture* is Entry #025. Entry #022 already introduced the four conflict strategies at a high level, so this note references that ladder briefly and spends its depth on the mechanisms.

---

## Recap: The Ladder of Conflict Strategies

Entry #022 established the four ways a system can deal with concurrent writes to the same data, and the ordering from "best if you can" to "last resort." A one-paragraph refresher, because this note lives on the last rung:

**Conflict avoidance** (route all writes for a record to one place, so conflicts can't arise) is best when achievable. **Last-write-wins** (pick a timestamp winner, discard the rest) is simple but silently lossy and clock-dependent. **Manual resolution** stores the conflicting versions ("siblings") and asks a human to merge — this is the "surface it to the user" UX from Entry #023 (Draft's side-by-side diff). And **automatic resolution** merges concurrent edits *programmatically* into a single sensible result with no data loss and no human intervention.

That last rung is what makes real-time collaborative editing possible — you cannot ask a human to resolve a conflict on every keystroke while two people type in the same paragraph. Two techniques achieve it: **operational transformation (OT)** and **conflict-free replicated data types (CRDTs)**. They solve the same problem in opposite ways, and this note is about how each works and where each wins.

```
   THE PROBLEM automatic resolution solves:

   Two users edit the same document concurrently, each applying their edit
   locally and instantly (no round-trip — that's the whole point). The edits
   then meet. There is no single pre-agreed order. How do we merge them so
   that BOTH users end up seeing the SAME final document, WITHOUT losing
   anyone's work and WITHOUT a human refereeing each conflict?

        OT  ── transform each operation against the others so it still
               "means the right thing" after reordering
        CRDT ── design the data structure so that ANY order of merging
               concurrent changes converges to the same result
```

The property both aim at is **strong eventual consistency**: not only do replicas eventually converge (ordinary eventual consistency, Entry #021), but any two replicas that have seen the *same set* of updates are in the *same state*, regardless of the order they applied them — and they converge *without* conflicts needing manual repair.

---

## Operational Transformation (OT)

### The Core Idea

OT represents every edit as an **operation** described by a position — for text, things like `InsertText 'Hello' @1` (insert "Hello" at index 1) or `Delete @5`. The problem is that a position-based operation becomes *wrong* the moment someone else's edit shifts the positions out from under it. OT's answer is exactly what its name says: when an operation arrives that was generated against a different version of the document, you **transform** it against the operations it didn't know about, adjusting its parameters (its positions) so it still does the right thing.

```
   Document is empty. Two users type at once:

   Luiz:  InsertText 'Hello' @1      →  document becomes "Hello"
   John:  InsertText '!'     @1      →  (in his empty copy) becomes "!"

   Now John receives Luiz's "Hello" @1. If he applied it verbatim he'd get
   a mess. Instead OT TRANSFORMS John's own pending op against Luiz's:

       John's '!' @1  ── transform against Luiz's 'Hello' @1 ──►  '!' @6

   because Luiz's 5-character insert at the front pushed John's "!" over by 5.
   Result on both sides: "Hello!"  ✓  (and later "Hello world!" — see below)
```

The transform function is the heart of OT: given two concurrent operations, it produces adjusted versions such that applying them in either order yields the same result. That sounds simple for two inserts; it is notoriously hard in general. Every pair of operation *types* (insert-vs-insert, insert-vs-delete, delete-vs-delete, and richer ops like formatting) needs a correct transform, and the interactions produce what Figma called a "combinatorial explosion of possible states." OT is powerful but **famously difficult to implement correctly** — which is the recurring theme when we compare it to CRDTs.

### Google Docs: OT Plus a Collaboration Protocol

The 2010 Google Docs write-up is the clearest real-world account of OT in production, and its key lesson is that **OT alone isn't enough — you also need a protocol** governing how clients and the server exchange operations. Google frames it as two problems: OT handles "how do we merge two edits to the same region," and the *collaboration protocol* handles "how do we coordinate many edits happening at once." Google Docs is **client/server with the server as central authority** — not peer-to-peer.

Each **client** tracks four things:
1. the revision number of the most recent change it received from the server,
2. local changes made but **not yet sent**,
3. local changes **sent but not yet acknowledged** by the server,
4. its current view of the document.

The **server** tracks three things:
1. changes received but not yet processed,
2. the full **revision log** (complete history of processed changes),
3. the current document state as of the last processed change.

Two rules make this work smoothly:

**Rule 1 — Optimistic local apply.** A client applies its own edit *immediately*, locally, without waiting for the server. This is why typing feels instant regardless of network speed — the same "no spinners" principle from Entry #023.

**Rule 2 — One in-flight change at a time.** A client sends only *one* change to the server and then holds all subsequent local edits in the "pending" list until that change is acknowledged. This bounds the coordination problem — the server never has to reason about a client's entire uncommitted history, only its single sent change.

The elegant part is what the **server does when a client's change collides with history**. Suppose John sends a change believing it will be Revision 2 — but the server has already committed someone else's Revision 2. The server doesn't reject John's change; it **transforms** it against every change committed since John last synced, and commits the result as Revision 3:

```
   John sends his op thinking it's Revision 2.
   Server already committed Luiz's op as Revision 2.
        │
        ▼
   Server transforms John's op against Luiz's committed op(s)
   (shifting John's position over by 6, exactly as John's client
    would have done on receiving Luiz's op) → commits as Revision 3.
        │
        ▼
   Server sends the acknowledgement to John, the transformed op to Luiz.
   Everyone converges on "Hello world!"  ✓
```

Note the symmetry: the *server* transforms an incoming op against the committed log, in the very same way a *client* transforms an incoming op against its pending changes. Transformation happens on both sides of the wire.

Google lists five advantages of this protocol, and they're worth keeping because they're the design goals of *any* good collaboration protocol:
- **Fast** — every editor optimistically applies changes locally; network speed never gates typing.
- **Accurate** — there's always enough information for every client to merge deterministically to the same result.
- **Efficient** — only the minimal delta (what changed) crosses the network.
- **Constant complexity** — the server keeps *no per-client state*, so adding more editors doesn't increase per-change processing cost. This is OT-with-a-central-server's big scaling win.
- **Distributed** — only the server needs the full history; only clients need their own uncommitted changes. The work is spread across the parties.

The payoff, in Google's words: no more collaboration conflicts, and editors see each other's changes "character-by-character." That character-level *merging* — where two people typing in the same word produce the union of both insertions — is precisely what OT gives you and what Figma's approach deliberately does *not* (below).

---

## CRDTs (Conflict-Free Replicated Data Types)

### The Core Idea

CRDTs attack the same problem from the opposite direction. Instead of *transforming* operations to fit a changing document (OT's approach), a CRDT is a **data structure designed so that concurrent changes commute** — merging them in *any order* mathematically converges to the same result. There's no transform function and, in the pure form, no central authority needed: replicas exchange their changes peer-to-peer, merge them in whatever order they arrive, and are guaranteed to converge. This is **strong eventual consistency** by construction.

The trick is in the data structure's design. A few illustrative building blocks:
- A **grow-only counter** where each replica increments its own slot, and merging takes the sum — obviously order-independent.
- A **last-write-wins register** that tags each value with a unique, ordered marker so merging just keeps the "winner" deterministically.
- **Sequence/list CRDTs** (the hard, interesting ones, for text) that give every character a stable, unique identity with a fractional/dense position between its neighbors, so two people inserting at "the same place" get two distinct identities that both survive and order deterministically — yielding the same character-level merge OT achieves, but via identity rather than transformation.

The famous cost, flagged in Entry #023: pure CRDTs must carry **metadata** (those unique identities, version vectors, tombstones for deletes) so that any replica can merge with any other with no coordinator. That metadata accumulates — a long-lived document grows a large change history — which is CRDTs' main practical drawback. And per Entry #023's essay, the one case CRDTs can't silently resolve is two users concurrently changing *the exact same property of the same object*; that still surfaces as a conflict.

### Figma: A CRDT-Inspired Approach (and Why Not OT)

Figma is the instructive real-world CRDT example precisely because it's *pragmatic* about it — it explicitly rejected OT, and it uses CRDT *ideas* without paying for pure CRDTs.

**Why they rejected OT:** as a startup they valued shipping quickly, and OT is "unnecessarily complex for our problem space." OT's power is in merging *long text documents* character-by-character — but Figma isn't a text editor, it's a design tool manipulating shapes. Their guiding principle was "no more complex than necessary," and OT's combinatorial-explosion difficulty bought them power they didn't need.

**Why not pure CRDTs either:** true CRDTs are built for *decentralized* systems with no central authority, and they pay memory/performance overhead for that generality. Figma is **centralized** — clients talk to a server cluster over WebSockets, and "the server is the central authority." So Figma stripped out the decentralization overhead and kept only the CRDT ideas that fit. Wallace's framing: a Figma document is *inspired by* several CRDTs combined, not a single true CRDT.

**The document model:** a Figma file is a **tree of objects** (like the HTML DOM) — a root, pages beneath it, content hierarchies below those. Each object has an ID and a set of properties. Conceptually it's a two-level map: `Map<ObjectID, Map<Property, Value>>`, or a database of `(ObjectID, Property, Value)` tuples. Adding a feature usually just means adding a new property.

**Conflict handling — last-writer-wins *per property*:** the server tracks the latest value any client sent for each property of each object.

```
   Two clients change DIFFERENT properties of the same object   → no conflict
   Two clients change the same property on DIFFERENT objects     → no conflict
   Two clients change the SAME property of the SAME object       → conflict:
        the last value received by the server wins (LWW register).
        No timestamp needed — the SERVER defines the order of events.
```

The critical consequence: **changes are atomic at the property-value boundary.** This is the deliberate difference from Google Docs. If a text property "B" is edited to "AB" by one client and "BC" by another concurrently, the result is "AB" *or* "BC" — **never "ABC."** Figma does not merge concurrent edits *within* a single property value. That's fine because it's a design tool, not a text editor — but it's exactly the character-level merge that OT (and sequence CRDTs) provide and Figma consciously forgoes.

**Object creation/deletion** are explicit operations (you can't create an object by writing to an unassigned ID). Clients generate **globally unique IDs by embedding their own client ID**, so two offline clients never collide — and IDs *can't* be server-assigned because creation must work offline. Deletion behaves like a LWW-set (an in/out boolean); deleted properties aren't kept on the server but live in the deleting client's undo buffer, so the document doesn't grow forever.

**The hard part — reparenting, cycles, and orphans:** moving an object to a new parent in the tree is where it gets subtle, and Figma's handling is a nice illustration of CRDT-style thinking:
- The parent link is stored as a **property on the child** pointing to its parent — so object *identity* is preserved (rejecting a "delete-and-recreate" approach that would drop concurrent edits when identity changes), and an object can't end up with two parents.
- But parent links are directed-graph edges with no built-in guarantee of forming a valid tree. Two clients can concurrently make A a child of B *and* B a child of A → a **cycle**. The **server rejects** parent updates that would create a cycle. A client can *temporarily* hold a local change plus a conflicting server change that together form a cycle; Figma's fix is to **temporarily remove such objects from the tree** (parenting them to each other, off-tree) until the server rejects the client's change — so an object may briefly *disappear*. Deemed acceptable as "a simple solution to a very rare temporary problem."

**Ordering children — fractional indexing:** to order siblings, each object gets a **position that is a fraction strictly between 0 and 1**. Children are sorted by position, and you insert *between* two siblings by setting the new position to the **average** of theirs. This means a reorder touches only *one* object's position, not all of them.

```
   siblings at positions:   0.25        0.5         0.75
                                   ▲
                     insert here → position = (0.25 + 0.5) / 2 = 0.375
```

The Figma details worth knowing:
- Positions are stored as **strings in base-95** (full ASCII) rather than base-10 for compactness, using **arbitrary precision** (not 64-bit floats) so you never "run out of precision after lots of edits."
- The parent link and the position must be stored as **a single atomic property** — a position is meaningless relative to a *former* parent, so they have to update together.
- **Pitfalls:** index strings can *grow* over time (rare in practice); **interleaving** can occur when clients concurrently insert in the same spot (acceptable for design objects that don't overlap, unacceptable for text — the reason this trick works for Figma but not a text editor); and two clients inserting between the *same* two siblings can produce **identical positions**, which you can't average — Figma's fix is for the **server to assign a unique position** to the second insert.

Fractional indexing is worth internalizing as a general technique: it's the standard way to maintain an ordered list under concurrent inserts without renumbering, and it shows up far beyond Figma.

---

## OT vs CRDT: Where Each Wins

The two techniques converge on the same goal — strong eventual consistency, automatic merge — but from opposite architectural starting points, and that shapes where each is used.

```
                    OT                                CRDT
   ─────────────────────────────────    ─────────────────────────────────
   idea:     transform operations         design data structures whose
             so they commute after        concurrent ops commute by
             reordering                    construction
   ─────────────────────────────────    ─────────────────────────────────
   authority: typically needs a          works peer-to-peer, no central
             central server to order      authority required (though can
             ops (Google Docs)            use one, like Figma, for leanness)
   ─────────────────────────────────    ─────────────────────────────────
   metadata: light on the document;      heavy — unique IDs, version info,
             complexity lives in the      tombstones accumulate in the doc
             transform functions          (history grows over time)
   ─────────────────────────────────    ─────────────────────────────────
   difficulty: transform functions are   the data-structure design is hard,
             famously hard to get right   but once built, merging is simple
             (combinatorial explosion)    and robust
   ─────────────────────────────────    ─────────────────────────────────
   sweet spot: real-time collaborative    distributed databases &
             TEXT editing                 general offline/decentralized apps
```

DDIA's summary of the split is the practical takeaway: **OT is most often used for real-time collaborative editing of text** (Google Docs being the canonical example — where character-level merge is exactly what you want), **whereas CRDTs are found in distributed databases** such as **Redis Enterprise, Riak, and Azure Cosmos DB** (where nodes must merge concurrent writes with no central coordinator, and there's no notion of "cursor position in text" to transform).

Figma sits interestingly between the poles: it took CRDT *ideas* but a *centralized* server (like OT's Google Docs), because that let it drop CRDT metadata overhead while still avoiding OT's implementation difficulty. Its LWW-per-property model is *weaker* than Google Docs' character-merge — it explicitly gives up "ABC" — but that weakness is a *deliberate, correct* choice for a design tool, and it's a reminder that you should pick the *weakest* merge semantics that still satisfies the product (the same "weakest sufficient guarantee" discipline as Entry #021).

The honest bottom line: OT gives you the strongest merge (true character-level text convergence) at the cost of a central server and hard-to-write transforms; CRDTs give you decentralization and robustness at the cost of metadata that accumulates; and real systems (Figma) freely mix ideas from both to hit the exact point they need. There is no universally best answer — which, by now, is the recurring shape of nearly every choice in this repo.

---

## Key Takeaways

- **Automatic conflict resolution** — the top rung of Entry #022's ladder — is what makes real-time collaboration possible, since you can't ask a human to referee every keystroke. Two techniques achieve it: **operational transformation (OT)** and **CRDTs**, both aiming at **strong eventual consistency** (replicas that saw the same updates are in the same state, regardless of apply order).
- **OT** models edits as position-based **operations** and **transforms** each incoming operation against the ones it didn't know about, adjusting positions so it still does the right thing. Powerful (true character-level text merge) but the transform functions are famously hard to get right.
- **Google Docs = OT + a collaboration protocol.** Central server as authority; each client tracks (sent / pending / last-synced-revision / current doc), each client applies edits **optimistically** and keeps **one change in flight** at a time; the server **transforms a late change against the committed revision log** and re-commits it. Advantages: fast, accurate, efficient, **constant complexity** (no per-client server state), distributed. Result: character-by-character merged editing.
- **CRDTs** design the data structure so concurrent changes **commute** — merge in any order, converge by construction, no central authority required. Cost: **metadata** (unique IDs, version info, tombstones) that accumulates over a document's life.
- **Figma = CRDT-inspired but centralized.** Rejected OT (too complex for a non-text tool) and pure CRDTs (decentralization overhead it didn't need). Doc is a **tree of `(object, property, value)` tuples**; conflicts resolved by **last-writer-wins per property** (server defines order) — so concurrent edits to one property never merge ("AB" or "BC", **never "ABC"**), a deliberate non-text choice. Handles reparenting via a parent-link property (server rejects cycles; offending objects briefly vanish) and orders children via **fractional indexing** (position = average of neighbors; base-95 arbitrary-precision strings; server breaks identical-position ties).
- **Where each wins (DDIA):** OT → real-time collaborative **text** (Google Docs); CRDTs → **distributed databases** (Redis Enterprise, Riak, Azure Cosmos DB). Figma mixes both. Pick the *weakest* merge semantics that satisfies the product.
