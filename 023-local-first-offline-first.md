# Local-First and Offline-First Software

**Date:** 2026-08-01
**Sources:**
- DDIA (2nd ed.) — "Sync Engines and Local-First Software": real-time collaboration, offline-first, local-first apps
- [Local-First Software: You Own Your Data, in spite of the Cloud — Kleppmann, Wiggins, van Hardenberg & McGranaghan (Ink & Switch, 2019)](https://www.inkandswitch.com/essay/local-first/)
- [Offline First — Alex Feyerke (A List Apart, 2013)](https://alistapart.com/article/offline-first/)

**Related entries:**
- [022-multi-leader-replication.md](022-multi-leader-replication.md) — offline clients and collaborative editing were two of multi-leader's justified use cases; local-first is that idea taken to its philosophical conclusion (every device is a leader)
- [024-conflict-resolution-crdts-ot.md](024-conflict-resolution-crdts-ot.md) — CRDTs, named here as the enabling technology, get their full treatment there
- [025-sync-engines.md](025-sync-engines.md) — the architectural pattern that makes local-first practical; this note is the *why*, that note is the *how*
- [021-consistency-models-session-guarantees.md](021-consistency-models-session-guarantees.md) — the Bayou session-guarantees work was built for exactly this mobile, intermittently-connected setting

> **Note on scope:** This is the first of three notes on the local-first cluster. It covers the *philosophy and UX* — why local-first/offline-first exists and what it demands. The *conflict-resolution mechanisms* (CRDTs, OT) are Entry #024, and the *sync-engine architecture* that implements it all is Entry #025. Read this one for the vision and the user-experience implications.

---

## The Problem: We Traded Ownership for Collaboration

To understand local-first software you have to see the trade the industry made over the last two decades, mostly without noticing. The Ink & Switch essay frames it as a tension between two eras of software.

The **"old world"** was files on your own disk. You opened a document in a desktop app, it lived on your hard drive, and it was *yours* — it worked with no network, it kept working whether or not the software vendor stayed in business, and nobody could revoke your access to it. But collaboration was miserable: you emailed `Budget draft 2 final final 3.xlsx` around, and merging everyone's edits was a manual nightmare of "conflicted copy" files.

The **cloud era** (SaaS, web apps — Google Docs, Trello, Figma) fixed collaboration beautifully. Everyone edits the same document, live, from any device. But it fixed it by moving the authoritative copy of your data onto *someone else's computer*. The essay's rallying sticker: *"There is no cloud — it's just someone else's computer."* And that inversion quietly costs you four things:

```
   OLD WORLD (files on disk)          CLOUD ERA (SaaS/web apps)
   ─────────────────────────          ─────────────────────────
   ✓ you own the data                 ✗ server owns the data
   ✓ works offline                    ✗ needs the network
   ✓ works forever                    ✗ dies when the service dies
   ✓ nobody can lock you out          ✗ provider can revoke access
   ✗ collaboration is painful         ✓ collaboration is seamless
```

The four cloud losses, concretely:
- **Ownership / control** — every action routes through the server, so you can only do what the server permits. If the server is down, or throttles you, or the feature you want isn't exposed, you're stuck.
- **Longevity** — when the company shuts the service down, the software stops working and your data may become inaccessible. The essay warns of a coming "digital Dark Age" of data locked in dead services.
- **Offline** — most cloud apps simply fail without connectivity.
- **Lock-out** — the provider can cut you off entirely (the essay cites 2017 cases of Google Docs users flagged as "abusive" and locked out of their own documents).

The animating question of local-first is therefore: **can we keep the seamless collaboration of the cloud while getting back the ownership of local files?** Not one or the other — both.

---

## The Core Inversion: Your Device Holds the Primary Copy

The central architectural idea of local-first is a single inversion of *where the truth lives.*

```
   CLOUD APP                              LOCAL-FIRST APP
   ─────────                              ───────────────
   server = the primary copy (truth)      YOUR DEVICE = the primary copy (truth)
   client = a temporary cache/view         server = a secondary copy, for sync
                                                     & backup & multi-device

   "the server is authoritative,          "your local data is authoritative,
    your screen is a window into it"        the server just helps it travel"
```

In a cloud app the server's copy is authoritative and your screen is just a window onto it — which is why the app is useless when the window loses its connection. In a local-first app, the copy **on your device** is the primary one. You read and write it directly and instantly; servers still exist, but demoted to a *supporting* role — they hold secondary copies to shuttle data between your devices, back it up, and help collaborators find each other. The server is no longer the source of truth; it's a helpful courier.

This single inversion is what makes everything else possible: if your local copy is authoritative, then of course you can work offline (you're not waiting on a window to a remote truth), of course there are no spinners (the data is right here), and of course you own it (it's on your disk). The hard part — the part that took until ~2011 to have a real answer for — is making multiple authoritative local copies *converge* when they've been edited independently. That's the conflict-resolution problem, and it's why CRDTs (below, and Entry #024) are the enabling technology.

---

## The Seven Ideals of Local-First Software

The essay's most cited contribution is a checklist: seven properties a truly local-first app should have. They're worth knowing individually because they double as a scorecard for evaluating any collaborative app (the essay grades Git, Google Docs, Dropbox, Firebase, etc. against them).

**1. No spinners — your work is at your fingertips.** Because the primary copy is local, the app responds instantly. There's no waiting for a server round-trip to see your own edit; sync happens in the background, out of the interaction path. Latency to your own data should be ~zero.

**2. Your work is not trapped on one device.** Local storage alone would recreate the old-world problem of data stuck on one machine. Local-first still syncs across all your devices — phone, tablet, laptop — so "local" doesn't mean "single-device."

**3. The network is optional.** You can read and write anytime, online or off, and changes sync whenever a connection appears. Crucially, "the network" needn't be the internet — sync could happen over local WiFi, Bluetooth, or even a USB stick. Connectivity is a convenience, not a requirement.

**4. Seamless collaboration with your colleagues.** Real-time collaboration at least as good as the best cloud apps — and specifically *without* the manual merge conflicts of the file world (Dropbox "conflicted copy," Git merge conflicts). This is the ideal that makes local-first hard, because it's in direct tension with #3: how do you merge independent offline edits automatically?

**5. The Long Now.** Your data should remain accessible for years, decades — "even after the company that produced the software is gone." This favors open, durable formats (plain text, PDF, JPEG) and the ability to keep running the original software (e.g., in an emulator). Data should outlive the vendor.

**6. Security and privacy by default.** Centralized server databases are honeypots — one breach exposes everyone. Because local-first keeps the real data on your devices, servers can hold only *encrypted* copies via **end-to-end encryption** (the model of Signal, WhatsApp, iMessage), so the courier can't read what it carries.

**7. You retain ultimate ownership and control.** "Ownership" here means *agency* — you control your data, not "intellectual property." Notably the essay says this doesn't require open source; it requires only that the software not *artificially* restrict what you do with your own files.

```
   1. No spinners (instant, local)          ┐
   2. Multi-device (syncs everywhere)        │  the "as good as cloud" ideals
   3. Network optional (works offline)       │
   4. Seamless collaboration                 ┘
   ─────────────────────────────────────────
   5. The Long Now (data outlives vendor)    ┐
   6. Privacy by default (E2E encryption)    │  the "better than cloud" ideals
   7. Ownership & control (user agency)      ┘
```

The first four are about *matching* the cloud; the last three are about *beating* it — the properties the cloud gave up that the old world had. A truly local-first app claims all seven.

---

## CRDTs: The Enabling Technology (Named Here, Detailed in #024)

The reason local-first became feasible around 2019 rather than 2009 is a piece of technology the essay leans on heavily: **Conflict-free Replicated Data Types (CRDTs)**, which came out of academic research around 2011.

The essay's framing is the useful one for now: CRDTs are general-purpose data structures — the collaborative equivalents of hash maps and lists — that are "multi-user from the ground up." A developer building a collaborative app can, in principle, swap an ordinary in-memory data structure for a CRDT and get automatic merging of concurrent edits for free. They let each device modify its local copy offline, then **automatically merge** everyone's independent changes when the devices reconnect — over *any* channel (server, peer-to-peer, Bluetooth). Changes can be as fine-grained as a single keystroke (enabling Google-Docs-style live collaboration) or as coarse as a batched pull-request. The one case CRDTs can't silently resolve is when two users concurrently change *the very same property of the same object* — and that gets surfaced for a human to sort out.

The essay likens CRDTs to an *enabling technology* the way packet switching enabled the internet — the foundational primitive that makes the whole vision buildable. Ink & Switch built **Automerge** (a JavaScript JSON CRDT) as their reference implementation. That's as far as this note goes; **the actual mechanics of how CRDTs converge — and their rival, operational transformation — are Entry #024.**

---

## Offline-First: The UX Discipline

Local-first is the ambitious, ownership-centric vision. **Offline-first** is a closely related, older, more UX-focused discipline — and the A List Apart essay (2013, predating both CRDTs-in-practice and service workers) is the classic statement of it. Where local-first asks "who owns the data?", offline-first asks a narrower, immediately practical question: **"how should an app behave when the network isn't there?"**

Its founding insight is an attack on a developer assumption: we build as if users share our conditions — latest devices, newest software, fastest always-on connections. They don't. As the essay puts it, *"Offline is simply a fact of life. If you're mobile, you'll be offline at some point"* — in a subway, a plane, an elevator, the countryside, or just the wrong corner of a room. The mistake is treating disconnection as an **error state**; offline-first treats it as a **normal, expected state** the app must handle gracefully.

The essay is careful about framing: it doesn't pitch offline-first *against* progressive enhancement. Instead it calls for "the UX equivalent of responsive web design" — a shared vocabulary of patterns for a disconnected, multi-device world, analogous to how responsive design gave us patterns for a multi-screen world.

### The Connectivity Lifecycle and Its Failure Points

The practical content is about handling the moments when connectivity changes. There are two points where the network can fail an app:

```
   CLIENT ──push──► SERVER      (can fail: your change can't be sent)
   CLIENT ◄──pull── SERVER      (can fail: you can't receive others' changes / fresh data)
```

A well-designed offline-first app handles both by:
- **Communicating or hiding connectivity state** — tell the user what's happening, or make it invisible, but never leave them guessing.
- **Allowing offline creation and editing** while reassuring the user their data is safe (e.g., "saved locally").
- **Rewording or disabling features that can't work offline** — a "Send" button becomes "Send later, when online" rather than failing.

### The Anti-Patterns It Names

The essay's most useful section is its catalogue of what *not* to do:
- **Losing local data.** An app that discards its state on disconnect is unforgivable. It should "retain their last state and make their data available, even if it can't be modified."
- **Treating offline like an error.** Don't show an empty view or an error dialog; show the last-known data. "Stop treating a lack of connectivity like an error."
- **Mishandling chronological data.** When queued changes finally sync, server-pushed content may land in unexpected places in a timeline; the UI must indicate *where in time* new content belongs.
- **Assuming text-only data.** Offline edits aren't just text — map markers, drawings, audio tracks, chart colors — and each needs its own offline story.

### Conflict-Surfacing as a UX Problem

Critically, the offline-first essay treats conflict handling as a **design** problem as much as a technical one — a bridge to Entry #024. Its contrasting examples are instructive:
- **Evernote** resolves note conflicts by *concatenating* both versions into one note, forcing the user to do "an inordinate amount of cognitive effort" to untangle them. Bad UX.
- **Draft** shows both versions side by side across three columns with per-difference "accept" / "ignore" buttons — "intuitive and visually appealing." Good UX.

And a subtle point: **not every conflict needs technical resolution.** The essay's restaurant example — an offline waiter's device can't *know* whether a wine is still in stock, but it *can* recognize the *risk* (low stock + offline) and prompt the waiter to give the customer a hedged answer. Sometimes the right move is to surface uncertainty to a human rather than resolve it automatically. This is the human-facing counterpart to the "manual conflict resolution" strategy that Entry #024 covers on the technical side.

---

## How Local-First Relates to What Came Before

It's worth placing local-first against the replication concepts already in this repo, because it's not a wholly new idea — it's a philosophical and technological escalation of them.

Recall from Entry #022 that two of multi-leader replication's justified use cases were **offline clients** and **collaborative editing**. Local-first is precisely those two cases taken to their logical extreme: **every device is a leader**, holding a full, authoritative, writable copy, syncing peer-to-peer or through demoted "cloud peer" servers. The write-conflict problem that made multi-leader dangerous (Entry #022) is exactly the problem local-first must solve — and its answer, CRDTs, is the "automatic conflict resolution for strong eventual consistency" that #022 named but deferred.

The connection to the Bayou session-guarantees work (Entry #021) is even more direct: Bayou was *built* for mobile users who are "frequently disconnected yet wish to collaborate" — the exact local-first setting, two decades early. And the essay's own evaluation confirms the lineage: it rates **Git + GitHub** as "the closest thing we have to a true local-first software package" (fast, offline, full control, good longevity) — limited only by its lack of real-time fine-grained collaboration and its line-based-text bias. Local-first is, in a sense, "Git's ownership model plus Google Docs' real-time collaboration," which is exactly the synthesis the opening question asked for.

The essay is honest about what's unsolved: CRDTs accumulate large edit histories that hurt performance over time, and peer-to-peer networking (especially NAT traversal) remains genuinely hard. Its pragmatic conclusion is that **cloud servers still have a role — as "cloud peers" for discovery, backup, and burst compute — just not as the source of truth.** That demotion of the server, not its elimination, is the realistic shape of local-first.

---

## Key Takeaways

- **The trade local-first reacts to:** the cloud era gave us seamless collaboration but took ownership, longevity, offline capability, and control — by making the *server* the authoritative copy of your data ("someone else's computer"). Local-first asks: can we have cloud collaboration *and* file-like ownership?
- **The core inversion:** in local-first, the copy **on your device is the primary/authoritative one**; servers are demoted to secondary "cloud peers" for sync, backup, and multi-device access. This is what buys instant response, offline work, and ownership.
- **The seven ideals:** (1) no spinners, (2) multi-device, (3) network optional, (4) seamless collaboration — matching the cloud; plus (5) the Long Now, (6) privacy by default (E2E encryption), (7) ownership/control — beating the cloud. They double as a scorecard for any collaborative app.
- **CRDTs are the enabling technology** — general-purpose, multi-user-from-the-ground-up data structures that auto-merge concurrent offline edits over any channel; the one unresolvable case (same property of same object) is surfaced to a human. (Mechanics: Entry #024.)
- **Offline-first is the UX discipline:** treat disconnection as a *normal state*, not an error. Handle both push and pull failure points; never lose local data or show empty error views; and treat conflict-surfacing as a *design* problem (Evernote's bad concatenation vs Draft's good side-by-side), sometimes surfacing *risk* to a human rather than auto-resolving.
- **Lineage:** local-first is multi-leader replication's "offline clients + collaboration" cases taken to the extreme (every device a leader), the setting Bayou (#021) targeted decades early, and roughly "Git's ownership + Google Docs' real-time collaboration." Servers keep a role — as peers, not authorities.
