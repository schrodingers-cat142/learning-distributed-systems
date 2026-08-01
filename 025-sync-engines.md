# Sync Engines

**Date:** 2026-08-01
**Sources:**
- DDIA (2nd ed.) — "Sync Engines and Local-First Software": the pros and cons of sync engines
- [A Map of Sync — Convex (2024)](https://stack.convex.dev/a-map-of-sync) — a nine-dimension taxonomy of sync systems
- [Are sync engines the future of web applications? — Isaac Hagoel (2024)](https://dev.to/isaachagoel/are-sync-engines-the-future-of-web-applications-1bbi) — the sync-engine pattern and Replicache
- [Scaling the Linear Sync Engine — Tuomas Artman (Linear, 2023)](https://linear.app/now/scaling-the-linear-sync-engine) *(a recorded talk; characterized here via the secondary sources above rather than transcribed — see caveat)*

**Related entries:**
- [023-local-first-offline-first.md](023-local-first-offline-first.md) — sync engines are the *architecture* that makes local-first's "no spinners" and "network optional" ideals practical; this note is the *how* to that note's *why*
- [024-conflict-resolution-crdts-ot.md](024-conflict-resolution-crdts-ot.md) — OT/CRDTs are the *merge logic*; a sync engine is the *plumbing* that ships changes between local store and server
- [022-multi-leader-replication.md](022-multi-leader-replication.md) — "server-authoritative vs decentralized" here mirrors the single-leader-vs-multi-leader distinction; a server-authoritative sync engine keeps one write authority
- [015-encoding-and-dataflow.md](015-encoding-and-dataflow.md) — a sync engine is a specific answer to "how does data flow between client and server," replacing request/response with a synced local cache

> **Note on scope:** Final note of the local-first cluster. Entry #023 is the philosophy, #024 is the conflict-resolution mechanics, and this is the **architecture pattern** — sync engines — that ties them into buildable apps. On **Linear** specifically: its detailed internals were presented in a recorded conference talk I could not transcribe, so I characterize Linear via Convex's and Hagoel's write-ups rather than claim its implementation details firsthand; that limitation is flagged where it matters.

---

## What a Sync Engine Is

A sync engine is a specific architectural answer to a question that has quietly plagued web apps for two decades: *how should the client get and change data?* The default answer, ever since the web began, is **request/response** — the client fetches data when it needs it and POSTs changes back, treating the server as the live source of truth it must constantly talk to (Entry #015's synchronous dataflow). A sync engine replaces that with something different.

Hagoel's definition is the clearest: a sync engine sits "at the core of the system" as "a persistent buffer between the frontend and the backend." Concretely:

- The client **always reads from and writes to a local store** — treating it as if the whole dataset "runs locally in memory." The UI never waits on the network to render or to accept an edit.
- That local store handles **optimistic updates**, **persists** to browser storage (IndexedDB), and **syncs bidirectionally** with the backend in the background, managing all the edge cases (retries, dedup, reconnection).
- The backend implements the other half — pushing/pulling changes, notifying clients, and persisting to the real database.

```
   REQUEST/RESPONSE (the old default)        SYNC ENGINE
   ──────────────────────────────────       ────────────────────────────────
   UI ──fetch/POST──► server ──► DB          UI ──read/write──► LOCAL STORE
      ◄──wait for response──                            (instant, optimistic)
   every interaction waits on the                          │  background sync
   network; the server is the                              ▼      (bidirectional)
   live source of truth                              server ──► DB
                                             the LOCAL STORE is what the UI
                                             talks to; sync happens off the
                                             interaction path
```

The inversion is the same one from Entry #023 (local copy as primary), applied at the *engineering* level: instead of the UI being a window onto a remote truth, the UI talks to a local store, and a sync engine keeps that store and the server in agreement. This is what operationalizes local-first's "no spinners" and "network optional" ideals — those ideals are *aspirations*; a sync engine is *how you actually build them*.

---

## Why Sync Engines: The Problems They Solve

Hagoel motivates sync engines with a list of hard problems that mainstream frameworks mostly ignore — problems every serious app eventually hits, usually solving each one ad hoc and badly:

1. Making the app **feel snappy** over slow or patchy networks (SPAs were "an early and ultimately insufficient attempt").
2. Implementing **undo/redo and version history** for user content.
3. Behaving correctly across **multiple tabs or devices** open at once.
4. Handling **long-lived sessions** running an outdated frontend.
5. Near-real-time **collaboration/multiplayer** with conflict resolution.

His observation is that teams defer these, hit a wall at scale, and end up with "partial, patchy solutions" or full rewrites — "the pain is real." A sync engine addresses the whole cluster at once, because they're all facets of the same underlying thing: keeping a local view and a server in agreement over an unreliable network. Notably, several category-defining apps — **Linear, Figma, Superhuman, Trello** — "disrupted incredibly competitive markets by being technologically superior," and a custom sync engine was a large part of that superiority. The snappiness *is* the product differentiator.

### The Benefits (DDIA + Hagoel)

- **Instant, snappy UX** — reads and writes hit the local store; network latency never gates interaction.
- **Undo/redo and history fall out naturally** — because changes are modeled as discrete **mutations** (reducer-like operations), you get undo/redo and version history far more easily than with scattered POST calls.
- **Lower network traffic** — a refresh loads mostly from local storage and pulls only the *diff* since last sync, rather than re-fetching everything (often less traffic than typical REST/GraphQL).
- **Offline and multi-tab for free-ish** — the local store is the single source the UI reads, so offline work and cross-tab consistency come from the same mechanism (e.g., broadcasting changes across tabs).

### The Drawbacks and Non-Fits (the honest half)

DDIA and Hagoel are both careful to say sync engines are *not* a universal answer:

- **Large initial load** — the client has to get the data before it's useful; big datasets mean a slow first sync (partly mitigated by *partial sync* — only syncing the slice a user needs).
- **Big-diff / conflict pain after long offline periods** — a client returning after being offline for a long time produces a huge diff and potentially hard conflict resolution.
- **Local storage can be wiped** — browsers can evict IndexedDB, risking loss of un-pushed work.
- **Wrong for server-authoritative actions.** This is the key non-fit, and Hagoel's rule of thumb is sharp: *any action that cannot function offline should not rely on a sync engine.* Placing a stock trade, submitting an order, taking a timed online test — these need the server to be the real-time authority, and an optimistic local "success" would be dangerous. Datasets too large to fit on a user's machine are the other non-fit.

The good news he stresses: it's not all-or-nothing. An app can use a sync engine for the parts that benefit (the collaborative document, the issue list) and plain request/response for the parts that must be server-authoritative (checkout, payment). Mix as needed.

---

## The Design Space: Convex's Nine Dimensions

The most useful framework for reasoning about sync engines is Convex's "A Map of Sync," which resists the urge to crown a winner and instead gives **nine dimensions across three categories** to *compare* systems. Knowing the axes is what lets you place any sync system — and reason about your own requirements. (It extends Aaron Boodman's earlier three-dimension sketch.)

```
   CATEGORY 1 — DATA MODEL (the "nouns")
     1. Size          how much data one client accesses (Figma cursor bytes → Dropbox TBs)
     2. Update rate   how often clients send changes (Linear ~1Hz → Figma cursors 60Hz)
     3. Structure     how much the engine understands the data
                      (Dropbox: opaque bytes → Automerge: JSON w/ invariants → Linear: rich object graph)

   CATEGORY 2 — SYSTEMS REQUIREMENTS (real-world constraints)
     4. Input latency  how fast one user's change reaches another
                       (Valorant ~50ms, 10ms detectable → Linear hundreds of ms fine → Dropbox seconds)
     5. Offline support none (Valorant kicks you) → partial (Linear) → full (Dropbox, Obsidian)
     6. Concurrent clients fixed cap (Valorant 22, Figma ~500) → unlimited (Linear, Dropbox via durable op logs)

   CATEGORY 3 — PROGRAMMING MODEL (developer experience)
     7. Centralization  decentralized/P2P (Automerge) → server-authoritative (Replicache) → proprietary (Figma, Linear)
     8. Flexibility     how programmable the conflict/sync logic is
                        (bespoke: Dropbox → CRDT middle → highly programmable: Replicache mutators)
     9. Consistency     weak (CRDTs default to LWW, no cross-key invariants) → strong (Replicache transactions)
```

Two things this framework makes clear. First, **"sync engine" is not one thing** — a system tuned for 60Hz Figma cursors (tiny data, extreme latency sensitivity, fixed client cap) is architecturally nothing like one tuned for Dropbox (terabytes, opaque files, seconds of latency tolerance, unlimited clients). Second, the dimensions *trade against each other* — the same "pick your point, pay elsewhere" shape as the RUM conjecture (#14) and the consistency spectrum (#21). You cannot have tiny latency, huge data, full offline, unlimited clients, and strong consistency all at once.

The two dimensions that matter most for connecting to the rest of this repo are **centralization** and **consistency**, so they get their own section.

---

## The Central Axis: Server-Authoritative vs Decentralized

Convex's *centralization* dimension is the one that ties sync engines back to the replication notes, because it's the same distinction as single-leader vs multi-leader (#016 vs #022), now at the client-sync layer. There are three points on it:

```
   DECENTRALIZED / P2P          SERVER-AUTHORITATIVE          PROPRIETARY / BESPOKE
   ────────────────────         ────────────────────          ─────────────────────
   Automerge, BitTorrent,       Replicache / Zero              Figma, Linear, Asana
   the local-first manifesto    (#023)
   no central authority;        server has final say;          highly tuned to one
   replicas merge peer-to-       local store is a cache,        product's data model;
   peer (needs CRDTs, #24)      mutations flow through          unlimited clients, but
                                the server as code               not reusable elsewhere
```

**Decentralized** systems (Automerge — the CRDT library from Entry #023's Ink & Switch) have *no* central authority; replicas merge peer-to-peer. This is the purest local-first model, and it *requires* CRDTs (#024) because there's no server to define an order. Its weakness, per Convex, is that CRDTs default to last-writer-wins and **can't enforce invariants that span multiple keys** — there's no coordinator to check "these two values must stay consistent with each other."

**Server-authoritative** systems (Replicache, and its successor Zero) are the pragmatic middle ground, and the one Hagoel focuses on. The local store acts as a **cache**, but the **server has the final say**. The elegant part of Replicache's model: the client uses **mutators** (reducer-like functions, à la Redux) that run *optimistically* on the local store *and* authoritatively on the server. The client applies a mutation instantly, then the server re-runs it as the source of truth; if the server's result differs, the client **rolls back and rebases** its optimistic changes on top of the server's authoritative state. Because mutations are server-validated transactions, this model gets **strong consistency and cross-key invariants** — the thing pure CRDTs can't do. Changes are *pulled*, not pushed: "the client always pulls them," often prompted by a lightweight "poke" over SSE/WebSocket telling it *when* to pull. Hagoel likes that Replicache is a *library*, not a black-box service — the frontend "effectively functions as a normal global store."

**Proprietary/bespoke** engines (Figma, Linear, Dropbox) are built and tuned for one product's exact data model. They achieve things general engines can't (unlimited clients, product-specific optimizations) but aren't reusable outside their org.

The mapping to earlier notes is exact: **decentralized ≈ multi-leader/leaderless** (no single write authority → needs conflict-free merge, #022/#024), **server-authoritative ≈ single-leader** (one authority orders writes → strong consistency, #016). A sync engine is, in effect, replication between a server and a fleet of client-side replicas — and the same authority tradeoffs apply.

---

## Systems on the Map

Convex scores six representative systems across all nine dimensions. A few, placed on the axes that matter, make the space concrete:

- **Dropbox** — the extreme of *size* (~terabytes) and the extreme of *low update rate* (~0.01 Hz); *opaque* structure (files are just bytes); full offline; unlimited clients via a durable operation log. A file-sync engine, bespoke for exactly that.
- **Figma** — tiny per-client data (cursors are bytes), *extreme* update rate (60 Hz cursors) and low latency; ~500-client cap; proprietary; CRDT-inspired merge (#024). Real-time visual collaboration.
- **Valorant** (the useful outlier) — a game "sync engine": ~50 ms latency where even 10 ms is detectable, *no* offline (it kicks disconnected players), a hard 22-client cap. Shows the framework spans far beyond web apps.
- **Automerge** — decentralized, medium structure, CRDT-based; the reference *local-first* engine. Auto-merges but can lose data on long offline edits and can't enforce cross-key invariants.
- **Replicache** — server-authoritative; high on offline, flexibility, and **strong consistency** (transactional mutators). The programmable middle ground.
- **Linear** — a proprietary engine over a **rich, structured object graph** (issues, teams, projects), *bursty ~1 Hz* updates, *partial* offline (it shows a UI disclaimer for offline edits), and *unlimited* clients backed by a durable change log (Convex notes a Postgres change table). It's the archetype of a purpose-built app sync engine that made the product feel instant.

**On Linear specifically:** its architecture was detailed in a recorded conference talk that I could not transcribe from the source, so the characterization above comes from Convex's and Hagoel's write-ups, not from Linear's own text. The shape is well-attested — a client-side object model backed by a local cache, real-time propagation, and a server-side durable change log enabling unlimited clients and partial offline — but I'm flagging that I'm relying on secondary description rather than the primary talk for the internals.

---

## Are Sync Engines "The Future"?

Hagoel's title poses the question and he declines a definitive yes/no, but his enthusiasm is clear, and his framing is a fair conclusion for this whole cluster. The vision he paints is "planet-scale, real-time, multi-player collaboration" apps that "work reliably regardless of network conditions" — which is precisely the local-first dream of Entry #023, now expressed as an engineering pattern rather than a manifesto.

The sober read is the one DDIA lands on and that runs through this note: sync engines genuinely solve a real, recurring cluster of problems (snappiness, offline, multi-device, collaboration, undo) that request/response handles badly — and the best products in competitive markets (Linear, Figma) won partly *because* they built one. But they carry real costs (initial load, storage eviction, big-diff conflicts) and real non-fits (anything that must be server-authoritative in real time — payments, trades, tests). They are a *powerful tool for a class of apps*, not a universal replacement for request/response — and the right architecture is often a **mix** of both within one app.

That verdict rhymes with nearly every conclusion in this repo: there's no universally best design, only the right point in a tradeoff space for a given workload. The nine dimensions are the map; where you land on them is dictated by what you're building.

---

## Key Takeaways

- **A sync engine is a persistent local buffer between frontend and backend.** The client always reads/writes a **local store** (optimistically, instantly, persisted to IndexedDB); the engine syncs it with the server in the background. It replaces request/response's "UI is a window onto remote truth" with "UI talks to a local store" — the engineering realization of local-first's "no spinners / network optional" ideals (#023).
- **It solves a recurring cluster of hard problems at once:** snappy UX on bad networks, undo/redo & history (via discrete **mutations**), multi-tab/multi-device correctness, stale long-lived sessions, and real-time collaboration. Category-defining apps (Linear, Figma, Superhuman) won partly by building one.
- **Costs and non-fits:** large initial load (mitigated by *partial sync*), big-diff conflicts after long offline, local-storage eviction, and — the key rule — **anything that can't function offline (payments, trades, timed tests) shouldn't use a sync engine.** Mix sync-engine and request/response parts within one app.
- **Convex's nine dimensions** compare sync systems across Data Model (size, update rate, structure), Systems Requirements (latency, offline, concurrent clients), and Programming Model (centralization, flexibility, consistency). "Sync engine" is not one thing; the dimensions trade against each other (the RUM/#14, consistency/#21 shape).
- **The central axis — centralization — mirrors single-leader vs multi-leader:** *decentralized/P2P* (Automerge; needs CRDTs, can't enforce cross-key invariants) ≈ multi-leader; *server-authoritative* (Replicache/Zero; local store is a cache, server has final say via transactional **mutators** → strong consistency, rollback-and-rebase) ≈ single-leader; *proprietary* (Figma, Linear, Dropbox) tuned to one product.
- **The verdict:** sync engines are a powerful tool for a *class* of apps (collaborative, offline-capable, snappy), not a universal replacement — the same "right point in a tradeoff space, not a silver bullet" lesson that recurs throughout this repo.
