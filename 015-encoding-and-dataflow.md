# Encoding, Schema Evolution, and Dataflow Between Services

**Date:** 2026-07-27
**Sources:**
- DDIA Chapter 4 (Encoding and Evolution) — JSON/XML/Protocol Buffers/Avro, reader vs writer schemas, modes of dataflow
- [Your API versioning is wrong (which is why I decided to do it 3 different wrong ways) — Troy Hunt](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/)
- [Exploring Event-Driven Architecture: A Beginner's Guide for Cloud-Native Developers — Srinath Perera, WSO2](https://wso2.com/blogs/thesource/exploring-event-driven-architecture-a-beginners-guide-for-cloud-native-developers/)

**Related entries:**
- [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) — event sourcing stores serialized events forever, so schema evolution of those events is a first-class concern; EDA here is the messaging substrate CQRS rides on
- [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) — "schema is inevitable" recurs here: even "schemaless" JSON has an implicit schema the reader must assume
- [005-facebook-tao.md](005-facebook-tao.md) — TAO is a concrete RPC-style service; its API is the "dataflow through services" pattern discussed here

---

## Why Encoding Matters at All

Every program works with data in two very different worlds. In memory, data lives as objects, structs, lists, hash maps — structures optimized for the CPU to follow pointers and manipulate quickly. But the moment you want to write that data to a file, send it over a network, or hand it to another process, you can't ship the pointers: a memory address is meaningless to anyone else. You need to translate the in-memory representation into a self-contained sequence of bytes. That translation is **encoding** (also called serialization or marshalling), and the reverse is **decoding** (parsing, deserialization, unmarshalling).

```
   IN MEMORY (pointers, objects)                 ON THE WIRE / ON DISK (bytes)
   ┌───────────────────────────┐                 ┌──────────────────────────┐
   │ User{                      │   encode  ──►   │ 0x0A 05 41 6C 69 63 65 …  │
   │   name: "Alice"  ─┐        │                 │ (a flat, self-contained   │
   │   friends: [ ●, ● ]│ ptrs  │   ◄── decode    │  byte sequence — no       │
   │ }                 └────►…  │                 │  pointers)                │
   └───────────────────────────┘                 └──────────────────────────┘
```

This sounds mundane, but it turns out to be one of the highest-leverage decisions in a long-lived system, for one reason: **data outlives code.** The messages you put on a queue today may be read by a service you haven't written yet; the events you append to a log (Entry #7) may be replayed years from now by code that has evolved many times. So the real question isn't just "how do I turn this into bytes?" but "how do I turn this into bytes such that *old code and new code can both read each other's data*?" That property is called **schema evolution**, and it's the thread running through this entire note.

Two directions of compatibility matter, and it's worth fixing the terms because they're easy to confuse:

- **Backward compatibility** — *new* code can read data written by *old* code. (You look "backward" in time at old data.) This is usually the easy one: you remember the old format.
- **Forward compatibility** — *old* code can read data written by *new* code. (Old code must gracefully ignore fields it doesn't understand yet.) This is the hard one, because old code has to be written to tolerate additions it can't anticipate.

In a system where you deploy services independently, you routinely have both old and new versions running *at the same time* during a rollout — so you need both directions to hold, or a deploy will break things mid-flight.

---

## The Encoding Formats

### JSON and XML: Textual, Human-Readable, Ubiquitous — and Loose

JSON and XML are the lingua franca of web APIs, and their great virtue is that a human can read them and almost every language can parse them. But they carry real weaknesses when used as a serious data-interchange format.

They're **verbose** — every record repeats all its field names as strings, and everything is text, so numbers and booleans take far more bytes than their binary equivalents. They're **ambiguous about types**: JSON can't distinguish an integer from a float, has no native binary-string type (people base64-encode binary, inflating size by 33%), and large integers lose precision because JavaScript treats all numbers as floats (Twitter famously had to return tweet IDs as both a number *and* a string because IDs above 2^53 got mangled). And critically, they have **no built-in notion of a schema** — the structure is implied by whatever the reader happens to expect. This is the "schema is inevitable" lesson from Entry #3 wearing a disguise: the schema doesn't vanish just because it's not written down; it just moves into the reader's assumptions, undocumented.

They're fine — genuinely fine — for public-facing APIs where human readability and universal tooling outweigh efficiency. They're a poor choice for high-volume internal traffic or long-term storage.

### Protocol Buffers and Avro: Binary, Schema-Driven, Compact

The binary formats fix the efficiency and type problems by requiring a **schema** defined up front, then encoding data *without* repeating field names — the schema tells both sides what each byte means. The two dominant choices, both from the big-data world, take subtly different approaches that are worth contrasting.

**Protocol Buffers** (Google; also Thrift, its close cousin from Facebook) uses **tag numbers**. You write a schema like:

```
message Person {
  required string name  = 1;   // the "1" is the field's tag number
  optional int64  id    = 2;
  repeated string email = 3;
}
```

Each field gets a permanent numeric tag. On the wire, each value is preceded by its tag number and a type annotation — the *field name string never appears*. Schema evolution then follows simple rules built entirely around those tags:

- To add a field, give it a **new tag number** and make it optional (or give it a default). Old readers see an unknown tag and skip it (forward compatible); new readers see the field missing and fall back to the default (backward compatible).
- You may **never reuse or change a tag number**, and you can't make a field `required` if old data might lack it. Tags are the contract.

**Avro** (from the Hadoop world) takes a strikingly different route: it uses **no tag numbers and no field identifiers at all.** The encoding is just concatenated values with nothing marking boundaries — it is completely unintelligible without the exact schema that wrote it. This seems fragile until you see the trick, which is the most conceptually interesting idea in this whole area.

### The Idea Worth Slowing Down For: Reader Schema vs Writer Schema

Avro separates the schema into two roles. The **writer's schema** is whatever version of the schema the program used when it *encoded* the data. The **reader's schema** is whatever version the program expects when it *decodes*. The key insight: **these two do not have to be identical — they only have to be compatible.** When Avro decodes, it looks at the writer's schema and the reader's schema *side by side* and resolves the differences:

```
   WRITER'S SCHEMA                    READER'S SCHEMA
   (used to encode the bytes)         (what the decoder expects)
   ┌──────────────────┐               ┌──────────────────┐
   │ name   : string  │               │ name     : string│
   │ id     : long    │               │ id       : long  │
   │ email  : string  │               │ phone    : string│  ← reader wants a field
   └────────┬─────────┘               └─────────┬────────┘     writer didn't write
            │                                    │
            └──────────► Avro resolution ◄───────┘
              • field in both, matched by NAME  → copy across
              • field only in writer            → reader ignores it
              • field only in reader            → fill from reader's default
              (order doesn't matter; names do)
```

Fields are matched by **name**, so order is irrelevant. A field present only in the writer's schema is ignored by the reader; a field the reader expects but the writer didn't provide is filled from the reader's declared default. That single mechanism gives both forward and backward compatibility cleanly, and it's why Avro is especially loved for large files and data pipelines.

But this raises an obvious question: if the bytes are meaningless without the writer's schema, **how does the reader get the writer's schema?** The answer depends on context, and each answer is a real pattern:

- **A large file with many records** (e.g., a Hadoop file with millions of rows): write the writer's schema *once* at the top of the file. The per-record cost of the schema is then negligible.
- **A database of individually-written records** written by different schema versions over time: tag each record with a small **version number** and keep a registry mapping versions → schemas. (This is exactly the shape of a **schema registry**, as used with Kafka.)
- **Messages over a network connection:** the two sides negotiate the schema version once when the connection opens.

### The Payoff Avro Uniquely Enables: Dynamically Generated Schemas

Because Avro identifies fields by name and needs no manually-assigned tag numbers, you can **generate a schema automatically from something else** — for instance, from the columns of a relational database table. If the table's columns change, you just regenerate the Avro schema; the reader/writer resolution handles the mismatch. With Protocol Buffers you'd have to manually assign and maintain tag numbers, so column changes require careful hand-editing to avoid reusing a tag. This makes Avro the natural fit for tools that dump databases or move data between systems whose schemas are themselves in flux. It's a small-sounding property with large practical consequences for data-infrastructure tooling.

### Quick Comparison

| | JSON / XML | Protocol Buffers / Thrift | Avro |
|---|---|---|---|
| Encoding | Text | Binary | Binary |
| Human-readable | Yes | No | No |
| Schema required | No (implicit) | Yes (`.proto`) | Yes (writer + reader) |
| Field identity | Field-name strings | **Tag numbers** | **Field names** (via schema resolution) |
| Evolution rule | Reader tolerance (ad hoc) | Never reuse a tag; new fields optional | Compatible reader/writer schemas + defaults |
| Dynamically generated schemas | N/A | Awkward (manual tags) | **Natural** |
| Best for | Public APIs, config, human eyes | Internal RPC, tight schemas | Big-data files, pipelines, evolving schemas |

---

## How Data Actually Flows Between Processes

Encoding is only half the story — it produces bytes, but *how* those bytes travel from one process to another shapes the whole system. DDIA identifies three broad modes of dataflow, and the rest of this note walks through each, because they embody very different coupling and failure characteristics.

```
   Mode 1: THROUGH A DATABASE          Mode 2: THROUGH SERVICE CALLS      Mode 3: THROUGH ASYNC MESSAGES
   ─────────────────────────          ────────────────────────────      ──────────────────────────────
   Process A writes ─► [ DB ] ─►       Process A ──request──► Process B    A ─► [ broker / queue ] ─► B
   Process B reads                         (REST / RPC)                       (A doesn't wait for B;
   (A and B may be the same app             ◄─response──                       B may not even exist yet)
    at different points in time)        synchronous, tightly coupled       asynchronous, loosely coupled
```

Mode 1 (a shared database) is really the schema-evolution problem again — the "message" is the row, and the process reading it later may run newer or older code. The interesting architectural choices live in modes 2 and 3.

---

## Mode 2: Synchronous Calls — REST and RPC

When one service needs another to *do something now and answer back*, it makes a synchronous call over the network. Two philosophies dominate.

**REST** is not a protocol but a *design style* built on top of HTTP. It leans into what HTTP already gives you: resources identified by URLs, standard verbs (`GET`, `POST`, `PUT`, `DELETE`), status codes, caching, and content negotiation. A REST API feels like navigating a web of nouns. Its strength is that it works everywhere, is easy to inspect (you can `curl` it), and rides the entire HTTP ecosystem for free.

**RPC (Remote Procedure Call)** takes the opposite stance: make calling a remote service *look like calling a local function*. gRPC (built on Protocol Buffers), Thrift, and the older systems all do this. The appeal is developer ergonomics — you call `getUser(id)` and it feels like normal code. The danger is that this appeal is a *leaky abstraction*, and understanding why is important:

- A local function call is fast and predictable; a network call can be slow, and its latency varies wildly.
- A local call either returns or throws; a network call can **time out with no answer at all**, leaving you genuinely unsure whether the other side did the work. Retrying then risks doing it twice unless the operation is *idempotent* (which ties directly back to the retry and backoff reasoning in Entries #1 and #2).
- A local call passes pointers cheaply; a remote call must encode every argument, so large objects are expensive.

RPC frameworks with binary encoding (gRPC) are excellent for high-performance *internal* service-to-service traffic where both sides are yours and you control deploys. REST tends to win for *public* APIs where reach, debuggability, and evolvability matter more than raw speed.

### The Real Operational Problem: API Versioning (Troy Hunt)

Whichever style you pick, the moment your API has external consumers you hit the same wall Troy Hunt describes from building *Have I Been Pwned*: **APIs must evolve, but you cannot break existing consumers.** He originally returned a bare list of breach *names*; he needed to return rich objects (title, breach date, count, description). That's a breaking change, so it needs a version.

He lays out the three common ways to version an API — and his mischievous framing is that all three are "wrong," so he implemented all three and lets the caller choose:

1. **Version in the URL path** — `https://haveibeenpwned.com/api/v2/breachedaccount/foo`. Dead simple, and *shareable* — you can paste it in an email and it just works. The philosophical objection: a URL is supposed to identify a *resource*, and `v2` isn't part of the resource's identity, it's a representation of it.
2. **A custom request header** — same URL plus `api-version: 2`. This works, but it reinvents something HTTP already offers (see #3), so it's redundant.
3. **Content negotiation via the `Accept` header** — `Accept: application/vnd.haveibeenpwned.v2+json`. This is the *semantically correct* option and Hunt's philosophical favourite: the resource stays at one stable URL, and the client states which representation it wants. The practical downside is testability — you can't just hand someone a clickable link; they need a tool that sets headers.

His actual lesson lands harder than the mechanics: *"It's about having a stable contract, stupid."* The versioning scheme barely matters — all three "return the same result" and the choice won't decide your project's success. What matters is that you **never break the contract existing consumers depend on.** A telling detail: when a caller specifies no version, HIBP returns **version 1, not the latest** — because defaulting to "latest" would silently break old clients the day you ship v2. Defaulting to the *oldest* keeps the promise.

```
   No version specified?
        │
        ├─ default to LATEST   ✗  breaks every old client the moment you ship a new version
        └─ default to v1 (oldest) ✓  old clients keep working; new clients opt in explicitly
```

That "default to oldest, opt in to new" rule is the practical heart of schema/API evolution, and it echoes the forward/backward-compatibility framing from the top of this note.

---

## Mode 3: Asynchronous Messages — Event-Driven Architecture

The third mode drops the "answer back right now" requirement entirely. Instead of A calling B and waiting, A emits an **event** — a record that something happened — and moves on. Some other service picks it up later. The WSO2 article's analogy is the clearest way in: **request/response is a phone call** (both parties must be present, synchronous), while **event-driven is email** (asynchronous, delivered even if the recipient is offline right now, processed when they get to it).

### The Components

```
   ┌────────────┐     event      ┌───────────────────────┐     event      ┌────────────┐
   │  Producer   │ ──────────►   │   Broker / Router      │  ──────────►  │  Consumer   │
   │ (event      │                │ (Kafka, RabbitMQ,      │                │ (event      │
   │  source)    │                │  Azure Event Hub)      │                │  handler)   │
   └────────────┘                 │                        │               └────────────┘
                                   │  decouples producer &  │               ┌────────────┐
                                   │  consumer in TIME and  │  ──────────►  │  Consumer 2 │
                                   │  SPACE                 │                └────────────┘
                                   └───────────────────────┘
```

An **event** is a significant change in state. A **producer** (event source) emits it, a **broker** sits in the middle, and one or more **consumers** (event handlers) react. The broker is the load-bearing piece: it **decouples producers from consumers in both time and space** — they needn't be running at the same moment, and neither needs to know the other's network address. This decoupling is the whole point, and it's what buys the architecture its headline benefits.

### Two Delivery Models

The broker can hand out events in two fundamentally different ways, and the distinction governs how you scale:

- **Publish/Subscribe (pub-sub):** consumers subscribe to a *topic*, and the broker delivers each event to *all* active subscribers. Use this when several independent services each need to react to the same event (an `OrderPlaced` event might fan out to billing, inventory, and email services simultaneously).
- **Queue (point-to-point):** consumers pull from a *queue*, and the broker delivers each event to *at most one* consumer. Use this to distribute work — ten worker instances on one queue means each job is handled once, and you scale throughput just by adding workers.

### Quality-of-Service Guarantees (the fine print that bites you)

Brokers differ in what they promise, and these promises are exactly where distributed-systems reality intrudes:

- **Persistence** — events written to disk so they survive a broker crash.
- **At-least-once** — no event is lost, but you may see **duplicates** (so consumers must be idempotent — the same theme from the retry notes yet again).
- **Exactly-once** — no loss *and* no duplicates; the strongest and most expensive guarantee.
- **In-order delivery** — events arrive in the order sent (often only guaranteed *within* a partition, not globally).

### Why Choose EDA — and Why Not

The benefits flow directly from decoupling. You get **loose coupling** (services evolve independently; you add a new consumer without touching producers), **scalability** (spread handlers across many nodes, often spun up on demand), **resilience to bursts** (a traffic spike piles up harmlessly in the durable queue and gets drained lazily, instead of timing out a synchronous call), and **fault tolerance** (a consumer can be down and catch up later).

The costs are just as real, and the article is refreshingly honest about them. The system is **harder to reason about** — you can't read it top-to-bottom as a sequence of statements, because control flow is scattered across handlers reacting to events. **Debugging is painful** — there's no clean stack trace; you reconstruct what happened by searching across logs and queues. You inherit all the **data hazards** of asynchrony: **missing, late, duplicate, and out-of-order events**, plus the need to document and evolve event schemas over time (which loops right back to this note's opening — those events want a format like Avro with a schema registry). A useful advanced concept the article introduces is **event correlation**: when a handler must wait for a *matching* event before acting (scatter-gather, windowed joins, happened-before patterns), which pushes you toward stream-processing / CEP engines (Kafka Streams, Siddhi) or durable workflow engines (BPMN, Camunda) that can hold long-running state.

The single sharpest heuristic from the article: **EDA optimizes for throughput; request/response optimizes for latency.** So EDA is a poor fit for a user-facing action that needs a sub-second answer, and an excellent fit for backend processing — ETL, fraud/anomaly detection, IoT and ML pipelines, monitoring long-running physical processes, and syncing data between microservices. The author's blunt advice: adopt EDA only when the advantages are *overwhelming*, because you're taking on genuine complexity to get them.

This connects straight back to Entry #7: **CQRS and event sourcing are built on top of exactly this messaging substrate.** The event log there is a durable, ordered stream; the read-side projections are event consumers. EDA is the plumbing; CQRS/ES is one sophisticated thing you build with it.

---

## Two Supporting Pieces: Service Meshes and Actor Frameworks

### Service Meshes (load balancing and the operational layer)

Once you have dozens of services calling each other (mode 2) or emitting events (mode 3), a new problem appears: every service needs the same *operational* capabilities — load balancing across instances, retries, timeouts, encryption (mTLS), health checking, and observability. Baking all of that into every service, in every language, is wasteful and inconsistent.

A **service mesh** (Istio, Linkerd) solves this by pulling those concerns out of the application and into a **sidecar proxy** deployed next to each service instance. The application just makes a plain call to `localhost`; the sidecar intercepts it and handles the load balancing, retries, and encryption transparently.

```
   Service A                             Service B
   ┌──────────┐   ┌────────┐   network   ┌────────┐   ┌──────────┐
   │  app code │──►│ sidecar │ ══════════►│ sidecar │──►│  app code │
   └──────────┘   │  proxy  │            │  proxy  │   └──────────┘
                  └────────┘             └────────┘
                  (load-balancing, retries, timeouts, mTLS,
                   metrics — all outside the application code)
```

The point is separation of concerns: **the mesh handles *how* traffic moves; the application handles *what* the traffic means.** Load balancing in particular moves from a central load balancer into these client-side sidecars, which can make smarter, per-request routing decisions.

### Distributed Actor Frameworks (a concurrency model for messaging)

The **actor model** (Akka, Microsoft Orleans, Erlang/Elixir's runtime) is a different way of structuring concurrent, message-passing systems. An **actor** is a small unit of state plus behavior that communicates *only* by sending and receiving asynchronous messages — it never shares memory with another actor. Because each actor processes one message at a time, you avoid locks and shared-memory race conditions entirely.

The elegant part for distributed systems: since actors already talk only through messages, **the model extends naturally across machines** — sending a message to an actor on another node looks the same as sending to a local one (with the same leaky-abstraction caveats as RPC: messages can be lost or delayed). A distributed actor framework is thus a programming model that bakes the "everything is an asynchronous message" philosophy of EDA directly into how you write the code, rather than treating messaging as external infrastructure.

---

## How It All Fits Together

```
                        WHAT SHAPE ARE THE BYTES?
                                  │
              ┌───────────────────┼───────────────────┐
         Human-facing?       Internal, fast?      Evolving / piped?
              │                   │                    │
           JSON/XML        Protocol Buffers           Avro
                            (tag numbers)      (reader/writer schemas,
                                                dynamic schemas)
                                  │
                        HOW DO THE BYTES TRAVEL?
                                  │
        ┌─────────────────────────┼──────────────────────────┐
   Via a database          Via a live call              Via async messages
   (schema evolution     REST (public, evolvable)       EDA / brokers
    of stored rows)      RPC/gRPC (internal, fast)      (pub-sub or queue)
                                  │                            │
                         versioning matters           throughput > latency;
                         (URL / header / Accept;       decoupled but harder
                          default to OLDEST)            to debug; CQRS builds
                                  │                     on this (#7)
                         service mesh handles
                         load balancing, retries,
                         mTLS via sidecars
```

The unifying thread from top to bottom is **evolution**: because data outlives code, every layer — the encoding format, the stored row, the API contract, the event schema — has to let old and new versions coexist. Get that right and services can be deployed independently and safely; get it wrong and every change becomes a coordinated, break-prone migration.

---

## Key Takeaways

- **Data outlives code,** so the real goal of any encoding is *schema evolution*: letting old and new code read each other's data (backward *and* forward compatibility).
- **JSON/XML** are human-readable and universal but verbose, type-ambiguous, and schema-implicit — great for public APIs, weak for volume or storage.
- **Protocol Buffers** encode fields by permanent **tag numbers**; **Avro** uses **field names** and resolves a **writer schema against a reader schema**, which uniquely enables **dynamically generated schemas** for data tooling.
- **Three modes of dataflow:** through a database (delayed schema evolution), through synchronous **REST/RPC** calls (tightly coupled; RPC's local-call feel is a leaky abstraction over network failure), and through asynchronous **events** (loosely coupled).
- **API versioning** (Hunt): URL, custom header, or `Accept` header — the scheme barely matters; keeping a **stable contract** does, so **default un-versioned callers to the oldest version**, not the latest.
- **Event-Driven Architecture** trades latency for throughput and coupling for complexity; a broker decouples producers and consumers in time and space, delivering via **pub-sub** (all subscribers) or **queues** (one consumer), with QoS knobs (at-least-once → duplicates, exactly-once, ordering) that force **idempotent** consumers.
- **Service meshes** externalize load balancing, retries, and mTLS into sidecar proxies; **actor frameworks** bake asynchronous message-passing into the programming model itself.
