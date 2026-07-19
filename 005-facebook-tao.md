# TAO: Facebook's Distributed Data Store for the Social Graph

**Date:** 2026-07-19
**Source:** [TAO: Facebook's Distributed Data Store for the Social Graph — Bronson et al. (USENIX ATC 2013)](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)

**Related entries:**
- [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) — TAO illustrates "What Goes Around" lessons: constrained hierarchical API on relational storage beats general graph DBs
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — TAO validates: fixed access patterns + hierarchical data → optimized non-relational API; MySQL underneath for durability
- [001-backoff-and-jitter.md](001-backoff-and-jitter.md) — TAO's leader serialization avoids thundering herds (same problem backoff solves at the client)
- [002-retry-policies.md](002-retry-policies.md) — TAO leaders rate-limit DB queries per shard (server-side analogue of token-bucket retry limiting)
- [006-amazon-neptune.md](006-amazon-neptune.md) — Opposite end of the graph spectrum: Neptune trades TAO's speed/scale for arbitrary traversals and ACID transactions

---

## What TAO Is

**TAO = "The Associations and Objects"** — Facebook's distributed data store purpose-built to serve the social graph. Not a general-purpose graph database. It's a **read-optimized, geographically distributed, eventually consistent caching system** sitting on top of MySQL.

**Scale (2013):** 1 billion reads/second, millions of writes/second, thousands of machines, multiple petabytes of data.

---

## The Problem TAO Solves

### Before TAO

Facebook stored the social graph in MySQL and used memcache as a lookaside cache:

```
Application → check memcache → miss? → query MySQL → store in memcache
```

Three fundamental problems:

**1. Inefficient edge lists:** Memcache is a key-value store. An edge list ("all friends of user X") is stored as one big blob. Any change requires fetching the entire list, modifying it, and writing it back. Concurrent updates are a nightmare.

**2. Distributed control logic:** In a lookaside cache, the *application* is responsible for cache invalidation. Logic lives in hundreds of application servers that don't coordinate → thundering herds, stale data, complex failure modes.

**3. Expensive read-after-write consistency:** Facebook uses async MySQL replication (primary → replica). After a write, the local replica may not have it yet. The previous solution ("remote markers") forwarded reads for recently-written keys to the primary region — expensive cross-region hops.

### TAO's Insight

Move the graph abstraction *into the cache layer itself*. Instead of dumb key-value caching, build a cache that **understands objects and edges** and can:
- Incrementally update edge lists (no read-modify-write)
- Serialize concurrent writes through a single leader per shard
- Provide read-after-write consistency via write-through caching

---

## The Data Model

### Objects (Nodes)

```
Object: (id) → (otype, (key → value)*)
```

- **id:** globally unique 64-bit integer (contains embedded shard_id)
- **otype:** type of object (USER, CHECKIN, COMMENT, LOCATION, etc.)
- **data:** key-value pairs, schema per otype

Examples:
```
id: 105, otype: USER, data: {name: "Alice"}
id: 534, otype: LOCATION, data: {name: "Golden Gate Bridge", loc: "37°49'..."}
id: 632, otype: CHECKIN, data: {}
id: 771, otype: COMMENT, data: {text: "Wish we were there!"}
```

### Associations (Edges)

```
Association: (id1, atype, id2) → (time, (key → value)*)
```

- **id1:** source object
- **atype:** type of edge (FRIEND, AUTHORED, TAGGED, LIKED_BY, COMMENT, etc.)
- **id2:** destination object
- **time:** 32-bit timestamp (used for ordering)
- **data:** optional key-value pairs

Constraint: **at most one association of a given type between any two objects.** (id1, atype, id2) is unique.

Examples:
```
(105, FRIEND, 244)       → Alice is friends with Bob
(105, AUTHORED, 632)     → Alice authored the checkin
(632, TAGGED, 244)       → Bob is tagged in the checkin
(471, LIKED_BY, 771)     → David liked the comment
```

### Bidirectional Edges

Many edges are symmetric (FRIEND) or asymmetric-inverse (AUTHORED / AUTHORED_BY). Bidirectional edges are modeled as **two separate associations**. TAO automatically maintains the inverse when you write the forward.

### Actions vs. Objects

- **Associations** naturally model actions that happen at most once (accepting an event invitation)
- **Objects** are better for repeatable actions (comments — each is a new object)

---

## The API

### Object API

Simple CRUD:
- `obj_get(id)` → returns object
- `obj_add(otype, data)` → creates, returns new id
- `obj_update(id, data)` → partial update
- `obj_delete(id)` → delete

Notable omission: no compare-and-set (useful in strong consistency; less useful in eventual consistency).

### Association Write API

- `assoc_add(id1, atype, id2, time, data)` → create/overwrite edge + its inverse if configured
- `assoc_delete(id1, atype, id2)` → remove edge + inverse
- `assoc_change_type(id1, atype, id2, newtype)` → reclassify edge

### Association Query API

Queries always start from **(id1, atype)** — "all edges of type X from object Y." These form **association lists**, ordered by time (newest first).

```
Association List: (id1, atype) → [a_new ... a_old]
```

- `assoc_get(id1, atype, id2set, high?, low?)` → check if specific edges exist
- `assoc_count(id1, atype)` → how many edges of this type
- `assoc_range(id1, atype, pos, limit)` → paginated list (by position)
- `assoc_time_range(id1, atype, high, low, limit)` → paginated list (by time window)

**Example queries:**
```
"50 most recent comments on Alice's checkin"
  → assoc_range(632, COMMENT, 0, 50)

"How many checkins at Golden Gate Bridge?"
  → assoc_count(534, CHECKIN)

"Is Bob tagged in Alice's checkin?"
  → assoc_get(632, TAGGED, {244})
```

**Limit:** TAO caps at 6,000 edges per (id1, atype). Longer lists require client pagination.

### Creation-Time Locality

Most queries are for the **newest** subset. "Show recent comments," not "show all 10,000 comments." This is a characteristic of social data — most data is old, but most queries are for new data.

### What TAO Does NOT Support

- No general graph traversals (no "find shortest path," no "all nodes within 3 hops")
- No joins across multiple association types in one query
- No arbitrary filters (no "all friends who live in Seattle")
- No compare-and-set / optimistic locking exposed to clients

Deliberately limits expressiveness to enable extreme performance and scalability.

---

## Architecture (Section 4)

TAO is separated into two caching layers and a storage layer.

### 4.1 Storage Layer (MySQL, Sharded)

#### Why sharding?

Petabytes of graph data. One MySQL server holds a few TB. Split data into **logical shards** — many more shards than physical servers.

Example: 4096 logical shards across 100 database servers. One server hosts ~40 shards. Rebalance by moving shards between servers.

#### Shard ID embedded in object ID

Object's 64-bit ID contains the shard number. Given any object ID, TAO immediately computes which shard (and therefore which server) holds it.

```
Object ID: 0x0003_0000_0000_002A
           ^^^^                    ← shard 3
```

Objects are bound to their shard for life.

#### Associations live on id1's shard

An association (id1, atype, id2) is stored on **the shard of id1** (source object).

```
Alice (id=105, shard 1) → FRIEND → Bob (id=244, shard 2)
This FRIEND edge lives on shard 1 (Alice's shard).
```

**Why?** The dominant query is "give me all X edges FROM object Y" — co-locating all outgoing edges of an object makes this a single-shard query.

#### Inverse edges are on different shards

Bidirectional edge (Alice → FRIEND → Bob) requires:
- Forward: (Alice → FRIEND → Bob) on shard 1 (Alice's shard)
- Inverse: (Bob → FRIEND → Alice) on shard 2 (Bob's shard)

Writing a bidirectional edge touches TWO shards. TAO does NOT do this atomically. If one write succeeds and the other fails → "hanging association" repaired asynchronously.

#### Why MySQL (not a custom store)?

1. Already there — operational maturity, tooling, backups, team expertise
2. TAO's API maps to simple SQL (SELECT, INSERT, UPDATE, DELETE by PK or range scan)
3. MySQL provides per-shard ACID
4. The caching layer handles distributed systems problems; DB just needs to be a reliable single-shard store

#### MySQL schema

Each shard's database has:
- One table for objects (all fields serialized into a single 'data' column — allows different otypes in one table)
- One table for associations (indexed on id1, atype, time for range queries)
- One table for association counts (avoids expensive SELECT COUNT queries)

---

### 4.2 Caching Layer

#### What's in the cache?

Each TAO cache server holds:
- **Objects:** full key-value data
- **Association lists:** ordered list of edges for (id1, atype), cached as contiguous time-sorted prefix
- **Association counts:** (id1, atype) → integer

#### How TAO's cache differs from memcache (dumb key-value)

**Semantic awareness — answering queries from partial data:**
```
Cache has: assoc_count(Alice, FRIEND) = 0
Query: assoc_get(Alice, FRIEND, {Bob}) — "Is Bob Alice's friend?"
Answer: NO (count is zero → no edges exist → no DB query needed)
```

**Incremental updates (no read-modify-write):**
```
Memcache approach (old):
  1. Fetch entire friend list from cache (5000 friends)
  2. Add new friend
  3. Write entire list back
  4. Race condition with concurrent updates

TAO approach:
  1. Prepend new edge to cached association list
  2. Increment cached count
  3. Done. No race condition.
```

**Serving range queries from a prefix:**
```
Full list in DB:     [edge10, edge9, edge8, edge7, edge6, edge5, edge4, edge3, edge2, edge1]
Cached prefix:       [edge10, edge9, edge8, edge7, edge6]

Query: "3 most recent friends" → served from cache (edges 10, 9, 8) ✓
Query: "8 most recent friends" → prefix too short → go to DB
```

#### Write-through caching

Writes update the cache **at write time** (synchronously for the writing client), not lazily. This enables read-after-write consistency.

#### Cache implementation details

- Based on Facebook's customized memcached: slab allocator, equal-size items, LRU eviction
- Dynamic slab rebalancer keeps LRU ages similar across slab classes
- RAM partitioned into **arenas** by object/association type — extend cache lifetime for important types, prevent poor cache citizens from evicting better-behaved data
- Small fixed-size items (counts) stored in direct-mapped 8-way associative caches (no pointers, saves memory overhead) — yields ~20% more items in cache
- A slab item holds one node OR one edge list

---

### 4.3 Client Communication Stack

#### The problem

Rendering one Facebook page = 100+ TAO queries to objects/edges on **different shards** = different cache servers. With 1000 app servers × 1000 cache servers = 1M connections.

#### The solution

**Multiplexed connections with out-of-order responses.** Pipeline many requests over one connection, handle responses as they arrive (like HTTP/2 or gRPC streaming).

Why out-of-order matters: cache hits return in 1ms, cache misses may take 50ms. You don't want a 1ms hit blocked behind a 50ms miss on the same connection.

TAO request latency is higher than memcache (because misses go to DB), so head-of-line blocking would be especially harmful.

---

### 4.4 Leaders and Followers

#### The problem with a single cache tier

If shard 7 maps to exactly one cache server:

1. **Hot spots:** Celebrity objects get millions of reads → one server crushed
2. **Connection explosion:** All app servers must connect to all cache servers → quadratic growth
3. **Write serialization is good but no read scaling:** The single server serializes writes (good) but can't horizontally scale reads

#### The solution: Two-tier cache

```
                   ┌──────────┐
        ┌──────────│  Leader   │──────────── DB
        │          └──────────┘
        │ invalidations/refills (async)
        │
   ┌────┴────┐  ┌──────────┐  ┌──────────┐
   │Follower1│  │Follower2 │  │Follower3 │   ← many follower tiers
   └────┬────┘  └────┬─────┘  └────┬─────┘
        │             │              │
    Clients       Clients        Clients
```

#### Follower Tier (many — read scaling)

- **Multiple follower tiers** (each tier = set of cache servers collectively covering all shards)
- Each tier independently serves any TAO read
- Clients talk **only to their nearest follower** — never to leader directly
- Followers serve cache hits from their own RAM
- Total read capacity = sum of all followers. Add tiers = add capacity.

#### Leader Tier (one per region — writes and coordination)

- Exactly **one leader tier** per region
- Each shard has exactly **one leader cache server** within that tier
- Leader is the **single point of contact** with the database for its shard

#### How a READ works (follower HIT)

```
1. Client → Follower-3: "assoc_range(Alice, FRIEND, 0, 10)"
2. Follower-3: local cache HIT → return data
   Total: ~1ms. Leader not involved. DB not involved.
```

#### How a READ works (follower MISS)

```
1. Client → Follower-3: MISS
2. Follower-3 → Leader
3. Leader: cache HIT? return to Follower-3
           cache MISS? → query DB → cache in leader → return to Follower-3
4. Follower-3: caches result locally → returns to client
   Total: ~5-15ms
```

**Why go through leader (not directly to DB)?**
- Leader's cache = second-level cache. If 5 followers miss simultaneously, only ONE DB query fires.
- Leader serializes and rate-limits DB queries → protects DB from thundering herds.
- Leader enforces a limit on concurrent pending queries per shard.

#### How a WRITE works

```
1. Client → Follower-3: "assoc_add(Alice, FRIEND, Eve)"
2. Follower-3 → Leader (forwards write)
3. Leader:
   a. Write to MySQL → success
   b. Update leader's cache (prepend edge, increment count)
   c. Return changeset SYNCHRONOUSLY to Follower-3
   d. Send async invalidation/refill to ALL other followers
4. Follower-3:
   a. Apply changeset → cache updated immediately
   b. Return success to client

   CLIENT CAN NOW READ ITS OWN WRITE ✓
   
5. (async, milliseconds later) Follower-1, Follower-2 receive invalidation → update
```

#### The changeset trick: read-after-write without waiting for propagation

Only the writing follower gets the synchronous update. Other followers get async notification. This gives:
- Read-after-write consistency for the writing client (synchronous)
- Eventual consistency for everyone else (async)

```
t=0ms:  Client writes via Follower-3
t=5ms:  Follower-3 has fresh cache. Client reads immediately → sees write ✓
t=5ms:  Follower-1, Follower-2 still have STALE cache
t=15ms: Async invalidation arrives → they update
```

#### Why ONE leader per shard (write serialization)?

Two clients writing simultaneously:
- Client A: "remove Eve from Alice's friends"  
- Client B: "add Eve to Alice's friends"

Without serialization, cache state may be inconsistent with DB. By funneling all writes for a shard through one leader:

```
Leader queue: [A's write] → [B's write] → execute in order → always consistent
```

#### Invalidation vs. Refill

- **Object updates:** leader sends **invalidation** to followers (follower discards cached object; next read refills from leader/DB)
- **Association writes:** leader sends **refill** messages to followers (tells them what was added/removed so they can update their cached list without discarding the whole thing)

Why refill for associations? Because invalidating an association list means the entire list must be re-fetched on next read. Since TAO caches only contiguous prefixes, invalidating discards many valid cached edges. Refill is much cheaper — just prepend/remove one edge.

#### Version numbers for consistency

Each cached item has a version number (in persistent store + cache). On update:
1. Leader increments version
2. Changeset includes new version
3. Follower checks: does changeset's "previous version" match my cached version?
   - YES → apply changeset
   - NO → my cache is stale from a missed update → invalidate (refill from leader on next read)

This handles the race condition where a follower misses one invalidation message.

---

### 4.5 Scaling Geographically

#### The problem

Users worldwide. If all DBs in Virginia, Tokyo users get 150ms latency on every cache miss.

#### The solution: Multiple regions with full data copies

Each geographic region has:
- A **complete copy** of the social graph (MySQL replicas)
- Its own **leader tier**
- Multiple **follower tiers**

```
┌─────────── US-EAST (Master Region) ────────────┐
│                                                  │
│  Clients → Followers → Leader → Master DB        │
│                                     │            │
└─────────────────────────────────────│────────────┘
                                      │ async MySQL replication
                                      ▼
┌─────────── EUROPE (Slave Region) ──────────────┐
│                                                  │
│  Clients → Followers → Leader → Slave DB         │
│                           │                      │
│                   writes forwarded to US-EAST    │
└──────────────────────────────────────────────────┘
```

**Each shard has one master region.** The master is chosen per-shard (not globally), and can be switched for failover.

#### Reads in slave regions: always local

```
User in Paris: "show me Alice's friends"
→ Local follower (Europe): cache HIT → return (1ms)
  OR
→ Follower MISS → Europe's leader → Europe's slave DB → return (5-15ms)

NEVER crosses the ocean for reads.
```

#### Writes in slave regions: cross-region hop

```
User in Paris: "add a comment"
1. → Follower-EU → Leader-EU
2. Leader-EU → Leader-US-EAST (crosses ocean, ~58ms)
3. Leader-US-EAST writes to master DB → success
4. Returns changeset to Leader-EU
5. Leader-EU updates cache, sends changeset to Follower-EU
6. Follower-EU updates cache → returns success to user

Total write latency: ~12ms (master local) + ~58ms (ocean) = ~74ms
```

#### How slave region's DB gets the write (replication)

After the master DB writes:
1. MySQL async replication carries the transaction to Europe's slave DB
2. **Invalidation/refill messages are EMBEDDED in the replication stream**
3. When slave DB applies the transaction, invalidations fire to local leader/followers
4. But the writing follower already has fresh data (from changeset) — this is for everyone else

#### Why embed invalidations in the replication stream?

If sent separately (earlier than replication):
```
t=0:    Master DB writes
t=0:    Invalidation sent to Europe → arrives t=58ms
t=58:   Europe follower invalidates cache, next read goes to...
t=58:   ...Europe's SLAVE DB, which hasn't replicated yet!
t=58:   Gets OLD data → caches stale data again!
t=200:  Replication arrives (too late)
```

By embedding in replication:
```
t=0:    Master DB writes
t=200:  Replication arrives at slave DB (data is fresh)
t=200:  Invalidation fires (embedded)
t=200:  Cache invalidated → next miss → gets FRESH data ✓
```

#### Why not multi-master (write from any region)?

Multi-master requires conflict resolution (last-writer-wins? vector clocks? CRDTs?). Enormously complex.

Facebook chose: **single master per shard.** Writes always flow through one master DB. Cost: extra cross-region write latency (~58ms). Benefit: zero conflict resolution, simple ordering, simple cache invalidation.

Since 99.8% of requests are reads (served locally), the extra write latency affects only 0.2% of requests. Excellent tradeoff for a read-heavy social network.

#### Read misses by followers: 25× more frequent than writes

This is why primary/replica architecture was chosen for DB replication (not consensus). Read-miss latency must be low (local), write latency is acceptable cross-region. Reads = 25× writes in terms of DB access.

---

### Full End-to-End Example: Alice in Europe adds Bob as a friend

Tracing through the entire system:

```
EUROPE (Slave Region)                           US-EAST (Master Region)
─────────────────────                           ──────────────────────

1. Alice's app → Follower-EU-2
   "assoc_add(Alice, FRIEND, Bob)"

2. Follower-EU-2 → Leader-EU
   (forwards write — followers never write to DB)

3. Leader-EU → Leader-US-EAST              ──→  4. Leader-US-EAST
   (cross-ocean forward, ~58ms)                    writes to Master DB:
                                                   INSERT INTO assoc
                                                   VALUES (Alice, FRIEND, Bob, now())

                                                 5. DB confirms → Leader-US-EAST updates its cache
                                                    sends changeset back to Leader-EU

6. Leader-EU receives changeset            ←──  (response, ~58ms back)
   Updates its own cache
   Sends changeset to Follower-EU-2
   Sends async invalidation to Follower-EU-1, Follower-EU-3

7. Follower-EU-2 updates cache
   Returns success to Alice

   ALICE CAN NOW READ HER OWN WRITE ✓
   (total latency: ~74ms)

8. (async, seconds later)
   MySQL replication carries the INSERT to Europe's slave DB
   Embedded invalidation fires → but cache already fresh
   Other followers already got invalidation in step 6

───────────────────────────────────────────────────────────────

MEANWHILE, writing the INVERSE edge:

4b. Leader-US-EAST also needs to write (Bob, FRIEND, Alice) on Bob's shard
    Bob might be on a different shard, possibly different server
    → Leader-US-EAST sends RPC to the leader hosting Bob's shard
    → That leader writes to DB + updates cache + sends invalidations

    If this inverse write FAILS → "hanging association"
    → async repair job will fix it later
```

---

### Architecture Summary

| Layer | Role | Scaling mechanism |
|-------|------|-------------------|
| Follower tiers | Serve 96%+ of reads from RAM | Add more follower tiers = more read capacity |
| Leader tier | Serialize writes, fill cache misses, protect DB | One per region, shields DB from thundering herds |
| MySQL (sharded) | Persistent storage, source of truth | More shards = more capacity |
| Multi-region | Low-latency reads globally | Full copy per region, reads always local |

**The insight: separation of concerns.**
- Followers optimize for **read throughput** (stateless, horizontally scalable)
- Leaders optimize for **write correctness** (serialization, single point of coordination per shard)
- MySQL optimizes for **durability** (disk, replication, backups)
- Each layer can be scaled, tuned, and operated independently

---

## Consistency and Fault Tolerance (Section 6)

The two most important requirements for TAO are **availability** and **performance** — not consistency. When failures occur, Facebook would rather show stale data than show nothing.

### 6.1 Consistency

#### Baseline: Eventual Consistency

After a write, TAO guarantees that eventually an invalidation or refill will be delivered to all tiers. Given enough quiet time (no more writes), all caches converge to the same correct state. Replication lag: usually < 1 second.

#### Stronger: Read-After-Write Consistency (within one tier)

If you write something and immediately read it back from the same follower tier, you see your own write.

**Mechanism:**

```
1. Client writes assoc_add(Alice, FRIEND, Eve) → Follower-3
2. Follower-3 forwards to Leader
3. Leader writes to master DB → success
4. Leader returns a CHANGESET synchronously on the response path:
   - Through slave leader (if in slave region)
   - To Follower-3
5. Follower-3 applies changeset to its local cache BEFORE returning to client
6. Client reads → hits Follower-3 → sees the write ✓
```

The key: changeset is returned **synchronously** on the write path. The follower updates its cache before responding to the client.

#### Cross-shard (inverse associations)

Bidirectional edge touches two shards:
- Forward: (Alice, FRIEND, Eve) on Alice's shard
- Inverse: (Eve, FRIEND, Alice) on Eve's shard

The master leader's changeset contains **both** updates. The slave leader and originating follower each forward the id2's changeset to the appropriate shard in their tier. Read-after-write works for both directions, as long as the write succeeded.

#### When read-after-write BREAKS

Only holds **within a single follower tier.** If you write via Follower-Tier-A but read from Follower-Tier-B (e.g., load-balanced to a different tier), you might not see your write — Tier-B hasn't received async invalidation yet.

In practice: user requests are sticky to one follower tier → consistent reads.

#### The changeset race condition (version numbers)

**Problem:** What if a changeset arrives at a follower whose cache is already stale from a missed previous update? Applying the changeset on top of stale data would corrupt the cache.

**Solution:** Every cached item has a version number (in persistent store + cache). On each update:

```
Leader increments version.
Changeset includes: new_version + expected_previous_version.

Follower receives changeset:
  If changeset.previous_version == my_cached_version:
      → APPLY changeset (my cache was up-to-date, safe to apply)
      
  If changeset.previous_version != my_cached_version:
      → INVALIDATE my cache (I missed an earlier update, can't apply)
      → Next read will refill from leader/DB with correct data
```

Worst case: follower invalidates and refills on next read. Never ends up in a corrupted state. Version numbers are internal — NOT exposed to TAO clients.

#### Rare race condition in slave regions ("time travel")

```
1. Follower has Object X at version 5 in cache
2. LRU evicts Object X (memory pressure)
3. Client reads Object X → cache miss → goes to slave DB
4. Slave DB still has version 4 (replication hasn't caught up)
5. Follower caches version 4
6. Client sees an OLDER value than it previously had!
```

Client's view goes backward. Only happens when:
- Cache eviction occurs (item leaves cache)
- AND slave DB replication is behind what cache had
- AND read refills from slave DB before replication catches up

Rare in practice: requires slave storage update to be slower than cache eviction cycle.

#### Critical reads (escape hatch for strong consistency)

For the rare case where stale data is unacceptable (authentication, credentials):

```
Client: "critical read for obj_get(user_credentials)"
→ Proxied directly to MASTER region's database
→ Bypasses ALL caches
→ Returns authoritative latest value
```

Expensive (cross-region hop + DB hit), so used only for a tiny subset of reads.

---

### 6.2 Failure Detection and Handling

TAO runs on thousands of machines across multiple geographic locations. Things break constantly.

**Philosophy:** Route around failures to preserve availability, at the cost of consistency.

#### Failure Detection Mechanism

- **Aggressive network timeouts** per destination
- Track per-destination timeout history
- Several consecutive timeouts → mark host as **down**
- Once marked down → proactively abort future requests (don't waste time trying)
- Periodically probe downed hosts to detect recovery

Simple failure detector — not perfect (bursty packet drops can look like failure), but fast. Speed > precision at this scale.

#### Database Failures

**Master DB goes down:**
- One of its slave DBs is **automatically promoted** to new master
- Writes during switchover are **failed back to client** (not retried)
- Brief window of write unavailability

**Slave DB goes down (in slave region):**
- Cache misses in that region can't be served from local slave DB
- Redirect cache misses to **master region's leaders** (cross-region, slower)
- Problem: invalidation messages normally embedded in replication stream to that slave. Slave is down → can't deliver.
- Solution: **additional binlog tailer** on master DB delivers refills/invalidations **inter-regionally** while slave is down
- When slave recovers → replication resumes → delivery reverts to normal

```
Normal:
  Master DB → replication stream → Slave DB → invalidation → local cache

Slave DB down:
  Master DB → additional binlog tailer → direct inter-regional delivery → local cache
  (more expensive, but maintains cache freshness)
```

#### Leader Cache Failures

A leader dies. This is serious — the leader serializes writes, fills cache misses, sends invalidations.

**For reads:** Followers route cache misses **directly to database** (bypass dead leader). Lose second-level cache benefit but still functional.

**For writes:** Followers reroute to a **random other member** of leader tier. This "replacement leader":
- Writes to database
- Modifies inverse association
- Sends invalidations to followers
- ALSO enqueues an **async invalidation to the original leader** (for when it recovers)

These async invalidations are:
- Recorded on the coordinating node
- Inserted into the replication stream
- Spooled until the original leader is reachable

**If leader is partially available** (receives messages but can't serve): followers may see stale data until spooled invalidations are delivered.

**If leader is permanently dead** (box replaced):
- **Bulk invalidation** of all shards that mapped to the old leader
- All followers invalidate those shards
- Expensive but ensures no stale data lingers

#### Follower Cache Failures

Least critical — followers are effectively stateless (cache rebuilds from leader).

Each TAO client has:
- **Primary follower tier**
- **Backup follower tier**

```
Normal:       Client → Primary tier (specific server for shard)
Primary down: Client → another server in same tier (if available)
              OR → Backup follower tier
```

**Consistency caveat on failover:**
```
1. Client writes via Primary-Tier-Server-7 → cache updated via changeset
2. Server-7 goes down
3. Client's next read goes to Backup-Tier-Server-12
4. Backup tier hasn't received async invalidation yet
5. Client sees STALE data (own write is missing)
```

Known tradeoff: availability over consistency during failover. Window is short (invalidations propagate in milliseconds) and scenario is rare.

#### Refill and Invalidation Delivery Failures

Leaders send refills/invalidations asynchronously. If a follower is unreachable:

```
Leader tries to send invalidation to Follower-X:
  → Network error / timeout
  → Leader QUEUES message to disk
  → Retries delivery later

Follower down for a long time:
  → Messages accumulate on disk
  → On recovery, queued messages delivered → cache becomes consistent

Leader PERMANENTLY fails before delivering queued messages:
  → Messages LOST
  → Follower keeps stale cache entries until:
    a. LRU eviction naturally removes them
    b. Bulk invalidation after leader box replacement
```

This is the weakest link in TAO's consistency. Acceptable because:
1. Rare (permanent leader failure + pending messages)
2. Stale window bounded by cache LRU lifetime
3. Bulk invalidation after box replacement cleans it up

---

### Consistency Hierarchy Summary

```
┌─────────────────────────────────────────────────────────┐
│ STRONGEST                                                │
│                                                          │
│ Critical reads → master DB directly                      │
│ (auth, credentials)                                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Read-after-write within a tier (normal operation)        │
│ Mechanism: synchronous changeset on write response path  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Eventual consistency across tiers and regions            │
│ Mechanism: async invalidation/refill propagation         │
│ Typical lag: < 1 second                                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Degraded consistency during failures                     │
│ - Leader failure: stale until invalidations catch up     │
│ - Follower failover: may miss recent writes briefly      │
│ - Slave DB down: inter-region delivery may lag           │
│                                                          │
│ WEAKEST                                                  │
└─────────────────────────────────────────────────────────┘
```

### Design Philosophy

**Availability > Consistency for a social network.**

- Showing a 1-second-stale friend list is invisible to users
- Showing a blank page because you're waiting for consistency is a bad user experience
- Only exception: security-sensitive data (credentials) → critical reads
- TAO never blocks reads waiting for consistency. Never refuses to serve because data might be stale. Always gives you *something* — and in 99.9%+ of cases, it's fresh.

---

## Performance Numbers (Production, 2013)

| Metric | Value |
|--------|-------|
| Read:write ratio | 99.8% reads : 0.2% writes |
| Overall cache hit rate | 96.4% |
| Availability (90-day window) | 99.9999% (4.9 × 10⁻⁶ failure rate) |
| Replication lag (85th percentile) | < 1 second |
| Replication lag (99th percentile) | < 3 seconds |
| Replication lag (99.8th percentile) | < 10 seconds |

### Read latency (client-observed, including network)

| Operation | Hit p50 | Hit p99 | Miss p50 | Miss p99 |
|-----------|---------|---------|----------|----------|
| assoc_count | 1.1ms | 28.9ms | 5.0ms | 186.8ms |
| assoc_get | 1.0ms | 25.9ms | 5.8ms | 143.1ms |
| assoc_range | 1.1ms | 24.8ms | 5.4ms | 93.6ms |
| assoc_time_range | 1.3ms | 32.8ms | 5.8ms | 47.2ms |
| obj_get | 1.0ms | 27.0ms | 8.2ms | 186.4ms |

### Write latency

| Location | Average |
|----------|---------|
| Same region as master | 12.1 ms |
| Remote region (~58ms RTT) | 74.4 ms |

### Workload characteristics

- Most edge queries return **empty results** (checking "does edge exist?" — usually no)
- 45% of `assoc_count` calls return zero
- 64% of non-empty range queries return exactly 1 edge
- Average association data: 97.8 bytes
- Average object data: 673 bytes
- 1% of assoc_count returns ≥ 512K (celebrity accounts)
- Follower hardware: 144GB RAM, 2× Xeon E5-2660, 10GbE

### Request type breakdown

| Read Operations | % | Write Operations | % |
|---|---|---|---|
| assoc_get | 15.7% | assoc_add | 52.5% |
| assoc_range | 40.9% | assoc_del | 8.3% |
| assoc_time_range | 2.8% | assoc_change_type | 0.9% |
| assoc_count | 11.7% | obj_add | 16.5% |
| obj_get | 28.9% | obj_update | 20.7% |
| | | obj_delete | 2.0% |

---

## Connecting to the Data Models Paper (Entry #3)

TAO is fascinating in light of "What Goes Around Comes Around":

1. **NOT a graph database** in the CODASYL/Neo4j sense. No general traversal. Can't say "find all paths between Alice and Eve." More like a **hierarchical model with typed edges** — always query from a specific object outward.

2. **Uses MySQL (relational) underneath.** The relational model wins again for storage. TAO adds a purpose-built caching abstraction on top.

3. **API is deliberately constrained** — a fixed set of access patterns (the IMS/DynamoDB lesson). By knowing all access patterns upfront, you can optimize radically.

4. **Eventually consistent** — explicit CAP tradeoff: availability + partition tolerance over consistency, for a workload where stale reads are acceptable.

---

## Key Design Philosophy

> TAO's goal is NOT to support a complete set of graph queries, but to provide sufficient expressiveness to handle most application needs while allowing a scalable and efficient implementation.

This is the same lesson as DynamoDB: constrain your query model → unlock extreme performance. The tradeoff is that complex queries (multi-hop traversals, pattern matching) must be built in application code by composing simple TAO calls.
