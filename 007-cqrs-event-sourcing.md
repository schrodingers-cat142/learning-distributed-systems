# CQRS and Event Sourcing

**Date:** 2026-07-19
**Sources:**
- [CQRS Documents — Greg Young (2010)](https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf)
- [CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Event Sourcing — Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)

**Related entries:**
- [005-facebook-tao.md](005-facebook-tao.md) — TAO's architecture is structurally similar to CQRS: writes serialized through leaders, reads served from eventually-consistent follower caches

---

## The Problem: Why Traditional Architecture Breaks Down

### The Stereotypical Architecture

Most systems follow a DTO up/down cycle:

```
┌─────────────┐
│  Database    │  ← stores current state
├─────────────┤
│Domain Objects│  ← maps to/from DB via ORM
├─────────────┤
│  App Service │  ← business logic
├─────────────┤
│Remote Facade │  ← API endpoint
└──────┬──────┘
       │
    Client sends up DTO
    Client receives back DTO
```

1. Client requests a DTO (Customer #1234)
2. Server loads domain objects, maps to DTO, returns it
3. User edits, clicks Save
4. Client sends modified DTO back
5. Server maps DTO to domain objects, validates, persists

### Problem 1: User Intent Is Lost

When the client sends back a modified DTO, the server only knows "these fields changed." Not WHY.

```
Before: {name: "Alice Smith", address: "123 Main St"}
After:  {name: "Alice Smith", address: "456 Oak Ave"}
```

Was this a typo correction or a relocation? Different business events with different downstream consequences. The DTO up/down pattern destroys this distinction.

### Problem 2: Single Model Serves Two Masters

The same domain model must:
- Process writes (validate rules, maintain consistency, handle concurrency)
- Serve reads (join data, support pagination, sorting, search)

| Concern | Write side needs | Read side needs |
|---------|----------------|----------------|
| Data shape | Normalized (3NF), consistent | Denormalized (1NF), fast to query |
| Consistency | Strong (transactional) | Eventual is often fine |
| Scale | Moderate (writes are rare) | Extreme (reads are 10-1000× writes) |
| Model | Behavioral (enforce business rules) | Structural (project data for display) |

> **It is not possible to create an optimal solution for searching, reporting, and processing transactions utilizing a single model.** — Greg Young

### Problem 3: The Scaling Bottleneck

The database stores current state and serves both reads and writes. Can't scale reads independently of writes. Can't optimize read schema without impacting write performance.

---

## Step 1: Task-Based UI (Capturing Intent)

Instead of sending modified DTOs, the client sends **Commands** — messages expressing what the user wants to do.

### CRUD approach (intent lost):
```
Client sends: {inventoryItemId: 42, status: "deactivated", comment: "expired stock"}
Server: "some fields changed, I'll diff and save"
```

### Task-based approach (intent preserved):
```
Client sends: DeactivateInventoryItemCommand {inventoryItemId: 42, comment: "expired stock"}
Server: "user wants to deactivate this item because of expired stock"
```

**Commands are imperative tense** — tell the system to DO something:
- `DeactivateInventoryItem`
- `RelocateCustomer`
- `ApprovePurchaseOrder`

Not "ChangeAddress" or "UpdateUser" — those are CRUD verbs disguised. Real commands capture domain meaning. Is it "ChangeAddress" or "CorrectAddress" vs "RelocateCustomer"? The naming process itself surfaces domain insight.

**Commands can be rejected.** The domain can say "no, you can't deactivate this item because it has pending orders." Natural — you're making a request.

---

## Step 2: CQRS — Splitting Read and Write Models

### Origins

CQRS comes from Bertrand Meyer's **Command Query Separation (CQS):** a method should either change state or return data, never both.

CQRS elevates this to architecture: split into two separate models.

```
CustomerWriteService                    CustomerReadService
─────────────────────                   ────────────────────
void MakeCustomerPreferred(id)          Customer GetCustomer(id)
void ChangeCustomerLocale(id, locale)   CustomerSet GetCustomersWithName(name)
void CreateCustomer(customer)           CustomerSet GetPreferredCustomers()
void EditCustomerDetails(details)
```

### The Command Side (Write Model)

- Receives Commands
- Loads domain objects
- Executes business logic
- Persists state changes
- Returns only success/failure (no data)
- Behavioral — enforces business rules
- Data stored normalized (3NF)

### The Query Side (Read Model)

- Receives queries
- Reads directly from data store via a **Thin Read Layer**
- Returns DTOs
- **No domain logic** — just project data
- Bypasses domain model entirely
- Data stored denormalized (1NF) — pre-joined, pre-aggregated

The Thin Read Layer reads from DB and maps directly to DTOs — no ORM, no domain objects. Developers working on the read side don't need to understand the domain model. Just the data model.

### Separate Data Stores (Full CQRS)

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   WRITE SIDE                          READ SIDE              │
│                                                              │
│   ┌──────────┐                       ┌──────────┐           │
│   │Write DB  │──── Eventually ──────→│ Read DB  │           │
│   │(3NF,     │     (events/          │(1NF,     │           │
│   │normalized)│      sync)           │denormalized)         │
│   └────┬─────┘                       └────┬─────┘           │
│        │                                   │                 │
│   Domain Objects                    Thin Read Layer          │
│   App Services                      (direct to DTO)         │
│        │                                   │                 │
│   Commands sent                     DTOs requested           │
│   Ack/Nak returned                  DTOs returned            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Events flow from write side to read side, keeping the read model eventually consistent.

### What Separate Models Enable

| Property | Write side | Read side |
|----------|-----------|-----------|
| Consistency | Strong (ACID) | Eventual (async from write side) |
| Storage | Normalized relational | Denormalized, document store, in-memory |
| Scaling | Rarely the bottleneck | Scale independently with replicas |
| Optimization | Optimize for correctness | Optimize for query speed |
| Complexity | Rich domain model | Simple projection |

---

## Step 3: Event Sourcing — Storing What Happened, Not What Is

### The Traditional Way: Store Current State

```
Account table:
  id: 1, balance: 6992.00, name: "Alice"
```

You only know the final state. Not HOW it got there.

### The Event Sourcing Way: Store Changes

```
Event log for Account #1:
  1. AccountOpened {owner: "Alice"}
  2. DepositMade {amount: 10000, from: "Check 1372"}
  3. WithdrawalMade {amount: 4000, check: "Check 1"}
  4. PurchaseMade {amount: 3.00, merchant: "Coffee Shop"}
  5. PurchaseMade {amount: 5.00, merchant: "Internet ISP"}
  6. DepositMade {amount: 1000, from: "Check 1373"}
```

Current balance = replay all events: 0 + 10000 - 4000 - 3 - 5 + 1000 = 6992.

### What Is a Domain Event?

**An event is something that has happened in the past.** Always past tense:
- `CustomerRelocated` (not "RelocateCustomer" — that's a command)
- `InventoryItemDeactivated`
- `OrderShipped`

| | Command | Event |
|---|---------|-------|
| Tense | Imperative ("do this") | Past ("this happened") |
| Can be rejected? | Yes | No (already happened) |
| Example | `PlaceOrder` | `OrderPlaced` |
| Semantics | A request/intent | A fact/record |

### Rebuilding State from Events

Instead of storing an Order as structure, store the events:

```
Events for Order #789:
  1. CartCreated {}
  2. ItemAdded {sku: "SOCKS-137", qty: 2}
  3. ItemAdded {sku: "SHIRT-354", qty: 4}
  4. ItemRemoved {sku: "SOCKS-137", qty: 2}
  5. ShippingInfoAdded {address: "123 Main St"}
```

To get current state: replay events 1-5 on a blank Order object.

Three capabilities built on the event log:
- **Complete Rebuild:** Discard state, replay all events from scratch
- **Temporal Query:** Replay events up to a point in time → state at that moment
- **Event Replay:** Correct a past error by reversing and replaying

### There Is No Delete

You can't say "event #2 never happened." Instead, add a **reversal event.** The history shows items were added AND then removed — full audit trail preserved.

Architectural benefit: append-only storage distributes more easily than mutable storage — far fewer locks needed.

### Rolling Snapshots (Performance)

Problem: Aggregate with 1 million events is slow to replay.

Solution: Periodically save a snapshot — serialized state at a specific version.

```
Event stream:  [1] [2] [3] [snap@3] [4] [5] [6]

Load current state:
  1. Find most recent snapshot (version 3)
  2. Deserialize → state as of event 3
  3. Replay only events 4, 5, 6 on top
```

Snapshots are taken asynchronously by a background process. They're a heuristic — conceptually the event stream is still the source of truth.

### The Event Store

Two tables:

**Events table:**
| Column | Type |
|--------|------|
| AggregateId | Guid |
| Data | Blob (serialized event) |
| Version | Int |

**Aggregates table:**
| Column | Type |
|--------|------|
| AggregateId | Guid |
| Type | Varchar |
| Version | Int (current version, denormalized) |

**Only two operations:**
1. `GetEventsFor(AggregateId)` → all events, ordered by version
2. `SaveChanges(AggregateId, OriginatingVersion, Events[])` → append with optimistic concurrency

Write pseudo-code:
```
Begin Transaction
  version = SELECT version FROM aggregates WHERE AggregateId = @id
  if expectedVersion != version
    RAISE concurrency conflict
  foreach event in events
    INSERT INTO events VALUES (@id, @event, ++version)
  UPDATE aggregates SET Version = @newVersion
End Transaction
```

**Aggregate IDs are the only partition point.** Horizontal partitioning is trivial — shard by AggregateId.

### Event Store as a Queue

Events need to be published to the read side. Naive approach: two-phase commit (write to DB + publish to queue atomically).

Better: Add a **SequenceNumber** (auto-incrementing) to events table. A separate publisher process chases the log:

```
Publisher:
  Loop:
    SELECT * FROM events WHERE SequenceNumber > @lastProcessed ORDER BY SequenceNumber
    Publish each event to queue
    Update @lastProcessed
```

The event store IS the queue. Write path does one disk write. Publishing is async.

---

## CQRS + Event Sourcing Together

### Why Event Sourcing Needs CQRS

Event Sourcing alone: can't query current state. "Give me all customers named Greg" is impossible — you'd have to replay ALL events for ALL customers.

CQRS solves this: write side stores events (append-only). Read side maintains denormalized views built from events (fast queries).

### Why CQRS Benefits from Event Sourcing

Without Event Sourcing: synchronizing two data models requires change data capture or manual event publishing.

With Event Sourcing: the write side **naturally produces events.** These events ARE the integration mechanism. No impedance mismatch on the write side — domain produces events, events are stored directly.

### The Combined Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│   COMMAND SIDE                           QUERY SIDE                    │
│                                                                        │
│   Client sends Command                   Client requests DTO           │
│         │                                       │                      │
│         ▼                                       ▼                      │
│   ┌───────────┐                          ┌──────────────┐             │
│   │App Service│                          │Thin Read     │             │
│   └─────┬─────┘                          │Layer         │             │
│         │ load events                    └──────┬───────┘             │
│         ▼                                       │ simple query         │
│   ┌───────────┐                                 ▼                      │
│   │ Domain    │                          ┌──────────────┐             │
│   │ Aggregate │                          │  Read DB     │             │
│   └─────┬─────┘                          │(denormalized)│             │
│         │ new events                     └──────▲───────┘             │
│         ▼                                       │                      │
│   ┌───────────┐        Event Handlers           │                      │
│   │Event Store│ ─────── (async) ────────────────┘                      │
│   │(append    │                                                        │
│   │ only)     │                                                        │
│   └───────────┘                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Write path:**
1. Command arrives
2. Load aggregate by replaying events from event store
3. Execute business logic → aggregate produces new events
4. Append new events to event store (optimistic concurrency)
5. Return success/failure

**Read path:**
1. Event handlers subscribe to event store
2. New events → handlers update denormalized read tables
3. Client queries read tables directly (no domain logic)

### No Impedance Mismatch on the Write Side

Traditional: Domain Object ↔ ORM mapping ↔ Relational Table. Complex, buggy.

Event Sourcing: Domain produces events → events stored directly. **The domain model IS the persistence model.** No mapping needed.

There IS an impedance mismatch on the read side (events → relational read model), but it's simpler: events represent actions → map naturally to INSERT/UPDATE on read tables.

---

## Business Value of the Event Log

### Answering Questions You Didn't Know You'd Ask

**State-based team:** Business wants "correlation between cart removals and later purchases." Don't have the data. Must add tracking going forward. Report only has data from that point.

**Event-sourced team:** Already stores `ItemAdded` and `ItemRemoved` for every cart since day one. Run new handler over historical log. Report has years of data immediately.

> "As the events represent every action the system has undertaken, any possible model describing the system can be built from the events." — Greg Young

### The "What" vs "How"

Most queries ask **what:** "what's the balance?" "how many in stock?" Only need current state.

Increasingly valuable queries ask **how:** "how did the account reach this state?" "what's the pattern before churn?" These need the event log.

---

## When to Use CQRS

### Appropriate

1. **Read/write asymmetry is extreme:** Reads 100-1000× writes, need independent scaling
2. **Complex domain with DDD:** Domain benefits from being freed from read concerns
3. **Read and write models are genuinely different in shape**
4. **Multiple read models needed:** Different consumers need different views
5. **Eventual consistency is acceptable on reads**

### NOT Appropriate

1. **Simple CRUD domains:** Read and write models are the same shape
2. **Small systems where single model isn't a bottleneck**
3. **Strong consistency required on reads immediately after writes**
4. **Team unfamiliar with the pattern:** "CQRS is a significant mental leap" — Fowler
5. **Applied system-wide:** Only for specific bounded contexts, not entire systems

> "For most systems CQRS adds risky complexity." — Martin Fowler

> "CQRS should only be used on specific portions of a system (a BoundedContext in DDD lingo) and not the system as a whole." — Martin Fowler

---

## When to Use Event Sourcing

### Appropriate

1. **Audit trail legally/business-required:** Finance, healthcare, compliance
2. **Temporal queries are valuable:** "What was the state on Dec 5th?"
3. **Business doesn't know what questions it'll ask in 5 years:** Event log preserves all information
4. **Debugging complex logic:** Replay events to reproduce bugs
5. **Integration via events:** Many systems react to same changes — event log is natural backbone
6. **Naturally transactional domains:** Accounting, trading, logistics

### NOT Appropriate

1. **Simple CRUD with no temporal needs:** Settings page, config table
2. **Heavy external system interaction:** Replay complexity with side effects (emails, payments)
3. **Team can't commit fully:** "Very hard to retrofit onto a system not built with Event Sourcing" — Fowler
4. **Frequent breaking schema changes:** Event versioning/upcasting adds complexity

---

## External Systems and Replay

A key challenge: events that trigger external calls (send email, charge payment) must NOT re-trigger during replay.

**Solution:** Wrap external systems in Gateways that check whether processing is "for real" or a replay:

```
class CustomsNotificationGateway:
  def notify(ship, port):
    if processor.isActive:    # real processing
      actually_send_notification()
    else:                     # replay — suppress
      pass
```

For external **queries** (e.g., exchange rates), the gateway logs the response on first call and replays the stored result during rebuilds.

---

## Cost Analysis: CQRS+ES vs. Traditional

Greg Young argues the combined architecture is **not more expensive** in most cases:

- Client work is identical (still sends commands, receives DTOs)
- Read side is equally or less expensive (Thin Read Layer is simpler than ORM projections)
- Write side replaces ORM complexity with event handling (different work, not more work)
- Integration is "production ready" from day one (events are the integration model)
- No impedance mismatch on write side saves significant developer time

> "The CQRS and Event Sourcing model is actually less expensive in most cases!" — Greg Young

---

## Summary

```
Traditional Architecture:
  One model, one database, CRUD operations.
  Simple but limited. Intent lost. Scaling hard.

  + Task-Based UI:
    Capture intent as Commands, not DTO diffs.

    + CQRS:
      Separate write model (behavioral, consistent)
      from read model (structural, fast, eventually consistent).
      Scale independently. Optimize independently.

      + Event Sourcing:
        Write side stores events, not state.
        Read side built from events.
        Complete history. Temporal queries.
        No impedance mismatch on writes.
        Natural integration mechanism.
```

Each step builds on the previous. You can adopt incrementally — Task-Based UI without CQRS, CQRS without Event Sourcing. The full combination is where the most interesting architectural properties emerge.
