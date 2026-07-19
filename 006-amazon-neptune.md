# Amazon Neptune: Managed Graph Database

**Date:** 2026-07-19
**Sources:**
- [Neptune Storage, Reliability and Availability](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-storage.html)
- [Neptune Graph Data Model](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-data-model.html)
- [Neptune Transaction Semantics](https://docs.aws.amazon.com/neptune/latest/userguide/transactions-neptune.html)
- [Neptune DB Clusters and Instances](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html)
- [Neptune Indexing Strategy](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-storage-indexing.html)
- [Neptune Customers](https://aws.amazon.com/neptune/customers/)

**Related entries:**
- [005-facebook-tao.md](005-facebook-tao.md) — Opposite end of graph data store spectrum: constrained API at extreme scale vs. Neptune's flexible traversals
- [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) — Neptune is "the network model reborn" (CODASYL with ACID and quasi-declarative queries)
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — When to choose Neptune vs. relational vs. DynamoDB

---

## What Neptune Is

A **fully managed, ACID-compliant, general-purpose graph database** on AWS. Unlike TAO (purpose-built for one workload), Neptune supports **arbitrary graph queries** — traversals of unknown depth, pattern matching, relationship-centric analytics.

Supports two data models simultaneously:
- **Property Graph** (queried via Gremlin or openCypher)
- **RDF** (queried via SPARQL)

---

## Where Neptune Fits in the "What Goes Around" Framework

| System | Historical Model | Tradeoff |
|--------|-----------------|----------|
| TAO | Hierarchical (IMS-like) with typed edges | Constrained API → extreme performance |
| Neptune | Network/Graph (CODASYL-like) | General traversal → flexible but slower per-query |
| DynamoDB | Hierarchical (IMS) | Fixed access patterns → predictable latency |
| Aurora/RDS | Relational | Declarative SQL → maximum flexibility |

Neptune = CODASYL reborn, but with:
- ACID transactions (CODASYL didn't have these properly)
- Quasi-declarative query languages (Gremlin/openCypher vs. CODASYL's pure navigation)
- Managed infrastructure

The paper's critique applies: for **most** workloads, relational (Aurora) suffices. Neptune's sweet spot is when **graph traversals of variable depth are your primary operation**.

---

## Architecture

### Storage Layer (Shared with Aurora's Design)

```
┌─────────────────────────────────────────────────────────┐
│              Neptune Cluster Volume                       │
│                                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │  AZ-1   │  │  AZ-2   │  │  AZ-3   │                │
│  │ 2 copies│  │ 2 copies│  │ 2 copies│  = 6 copies    │
│  └─────────┘  └─────────┘  └─────────┘                │
│                                                          │
│  NVMe SSD, 10GB segments, auto-expands to 128 TiB       │
│  Quorum writes: 4/6 must acknowledge                    │
│  Quorum reads: 3/6 (used during recovery)               │
└─────────────────────────────────────────────────────────┘
```

- **6 copies** of data across **3 AZs** (2 per AZ)
- Storage auto-scales in 10GB segments
- **Quorum writes:** 4 out of 6 storage nodes must ACK → durable
- **Quorum reads:** 3 out of 6 (only during recovery/repair)
- Automatic segment repair if corruption detected

**Key difference from TAO:** Neptune's storage is a single shared volume — all instances (primary + replicas) read from the **same physical storage.** No replication lag between primary and replica data (only buffer cache propagation delay, typically < 100ms).

### Compute Layer (Primary + Read Replicas)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Primary    │     │  Replica 1   │     │  Replica 2   │
│  (read+write)│     │  (read only) │     │  (read only) │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                     │                     │
       └─────────────────────┴─────────────────────┘
                             │
                    ┌────────┴────────┐
                    │  Shared Cluster  │
                    │     Volume       │
                    └─────────────────┘
```

- **One primary instance:** handles all reads AND writes
- **Up to 15 read replicas:** read-only, share same storage volume
- **No data copying when adding a replica** — connects to existing cluster volume
- Failover: if primary dies, replica promoted in seconds
- Query threads per instance = 2 × vCPUs (e.g., r5.4xlarge = 32 concurrent queries)

### Contrast with TAO Architecture

| | TAO | Neptune |
|---|---|---|
| Write path | Client → Follower → Leader → MySQL | Client → Primary → Shared volume |
| Read path | Client → Follower cache (96% hit rate) | Client → Primary or Replica → buffer cache or storage |
| Scaling reads | Add follower tiers (thousands of servers) | Add replicas (max 15) |
| Storage replication | MySQL async replication (full DB copy per region) | 6 copies in shared volume (single region) |
| Multi-region | Full copy per region, async replication | Not built-in (Global Database for cross-region) |

---

## Data Model: How Neptune Stores Graph Data

### The Quad Model (SPOG)

Internally, Neptune stores **everything** as quads:

```
(Subject, Predicate, Object, Graph)
```

Universal representation — both property graphs and RDF encoded into quads:

**Property Graph vertex with properties:**
```
Vertex "person_1" with label "Person", age=40, lives_in="New York"

Stored as quads:
  (person_1, type,     Person,   default_graph)   ← vertex label
  (person_1, age,      40,       default_graph)   ← property
  (person_1, lives_in, New York, edge_2)          ← edge
```

**Property Graph edge:**
```
person_1 --knows--> person_3

Stored as:
  (person_1, knows, person_3, edge_1)   ← "edge_1" in G position = edge ID
```

### Contrast with TAO's model

| | TAO | Neptune |
|---|---|---|
| Unit of storage | Objects + Associations (separate types) | Quads (uniform) |
| Edge identity | (id1, atype, id2) — no separate edge ID | Edge has its own ID (in G position) |
| Properties on edges | Key-value data attached to association | Separate quads with edge ID as subject |
| Schema | Per-otype/atype schema (fixed columns) | Schemaless (any predicate on any subject) |

### Dictionary Encoding

Neptune maintains a dictionary table mapping strings to numeric IDs:

```
"person_1" → 5
"knows"    → 6
"person_3" → 3
"edge_1"   → 9
```

Quad `(person_1, knows, person_3, edge_1)` stored as `(5, 6, 3, 9)` — compact integers, fast to compare.

---

## Indexing Strategy

Neptune maintains **three indexes** by default (optionally a fourth):

| Index | Key Order | What it answers efficiently |
|-------|-----------|---------------------------|
| **SPOG** | Subject → Predicate → Object → Graph | "All properties and outgoing edges of vertex X" |
| **POGS** | Predicate → Object → Graph → Subject | "All edges with label 'knows'" or "All vertices where age=40" |
| **GPSO** | Graph → Predicate → Subject → Object | "All properties of edge E" (edge ID in G position) |
| **OSGP** (optional) | Object → Subject → Graph → Predicate | "All incoming edges to vertex X" (reverse traversal) |

### How queries use indexes

**"Find all friends of person_1"** (outgoing 'knows' edges):
```
Pattern: S=person_1, P=knows, O=?, G=?
→ SPOG index: prefix scan on (person_1, knows) → all matching quads
```

**"Find everyone who knows person_3"** (incoming edges):
```
Pattern: S=?, P=knows, O=person_3, G=?
→ POGS index: prefix scan on (knows, person_3) → all matching quads
```

**"Find ALL incoming edges to person_3"** (any label):
```
Without OSGP: Must union-scan across ALL predicates × POGS. Expensive if many predicates.
With OSGP:    Prefix scan on (person_3) in OSGP → efficient
```

### Contrast with TAO's indexing

| | TAO | Neptune |
|---|---|---|
| Index structure | MySQL B-tree on (id1, atype, time) | Three/four quad indexes (SPOG, POGS, GPSO, OSGP) |
| "Outgoing edges from X" | One lookup: (id1, atype) | SPOG prefix scan on S=X |
| "Incoming edges to X" | Requires inverse association (separate write) | POGS or OSGP index (no extra write needed) |
| "All edges of type T" | Not supported (full scan) | POGS prefix scan on P=T (efficient) |

**Key difference:** TAO requires **pre-computing** inverse edges at write time. Neptune indexes both directions automatically. TAO can only efficiently query "edges FROM X of type T." Neptune can query "edges TO X" and "all edges of type T" without extra bookkeeping.

---

## Transaction Semantics (ACID)

### Read-only queries: Snapshot Isolation (MVCC)

- See a consistent snapshot as of query start time
- Never see dirty reads, non-repeatable reads, or phantom reads
- **Never block writers** (MVCC — readers don't take locks)
- Read replicas always run under snapshot isolation

### Mutation queries: READ COMMITTED with range locks

- Take range locks on index prefixes they read
- Prevent other mutations from inserting/deleting in locked ranges
- Guarantees repeatable reads within a mutation transaction

**Example of locking:**
```
Transaction T1: "Read all properties of person_1"
  → Locks range S=person_1 in SPOG index
  → No other transaction can INSERT or DELETE properties for person_1 until T1 commits

Transaction T2: "Add age=30 to person_1" (concurrent)
  → Needs exclusive lock on S=person_1 range in SPOG
  → BLOCKED. Waits up to 60s for T1 to finish.
```

### Conflict Resolution

- **No deadlock:** Blocked transaction waits up to 60 seconds for lock release
- **Deadlock detected:** Neptune immediately rolls back one transaction (fewest changes)
- **False conflicts (gap locks):** ~3-4% of writes under high load fail due to gap lock collisions. Clients should **retry** (→ see [001-backoff-and-jitter.md](001-backoff-and-jitter.md) and [002-retry-policies.md](002-retry-policies.md))

### Gap Locks and False Conflicts

Neptune uses gap locks (like MySQL InnoDB). A range lock covers the gap between index records:

```
SPOG index records for person_1:
  (5, 1, 12, 2)   ← person_1, type, Person
  (5, 6, 3, 9)    ← person_1, knows, person_3
  (5, 8, 40, 2)   ← person_1, age, 40
  (5, 10, 11, 14) ← person_1, lives_in, New York
  (7, 1, 12, 2)   ← person_2, type, Person  ← first non-matching record

Locking S=person_1 range locks gaps between these records.
Falsely blocks: any insert for person_3 (ID=3, adjacent in index) or new vertices with IDs between 5 and 7.
```

### Contrast with TAO's consistency

| | TAO | Neptune |
|---|---|---|
| Isolation | Eventual consistency (read-after-write within one tier) | Snapshot isolation (reads), READ COMMITTED + range locks (writes) |
| Multi-statement transactions | No (each API call independent) | Yes (multiple mutations in one atomic transaction) |
| Conflict handling | Leader serializes all writes per shard (no conflict possible) | Lock-based with deadlock detection and timeout |
| Cross-entity atomicity | No (inverse edge can fail independently → "hanging association") | Yes (single transaction modifies multiple vertices/edges atomically) |

---

## Read Replicas and Consistency

**Primary instance:**
- Absolute latest data (reads and writes)
- Mutation queries run here

**Read replicas:**
- Share same storage volume (same physical data, not a copy)
- See writes with small lag (< 100ms typically) — buffer cache propagation
- All reads run under snapshot isolation → always consistent (just possibly slightly stale)

**Need strong read-after-write:** Send reads to writer endpoint (primary).

**Contrast with TAO:**
- TAO: replication lag = MySQL async replication (seconds across regions)
- Neptune: replication lag = buffer cache invalidation within same cluster (< 100ms)
- But: Neptune is single-region by default. TAO is multi-region by design.

---

## What Neptune Can Do That TAO Cannot

### 1. Variable-depth traversals

```cypher
// "Find all people within 3 hops of Alice"
MATCH (a:Person {name:'Alice'})-[:KNOWS*1..3]-(friend)
RETURN DISTINCT friend.name
```

TAO: impossible. Would require separate API calls per hop, assembled in application code.

### 2. Pattern matching

```cypher
// "Find triangles — groups of 3 people who all know each other"
MATCH (a)-[:KNOWS]->(b)-[:KNOWS]->(c)-[:KNOWS]->(a)
RETURN a.name, b.name, c.name
```

TAO: impossible without exhaustive application-level computation.

### 3. Reverse traversals without pre-computed inverses

```cypher
// "Who follows Alice?" (incoming edges)
MATCH (follower)-[:FOLLOWS]->(a:Person {name:'Alice'})
RETURN follower.name
```

TAO: requires inverse associations maintained at write time. Neptune indexes automatically.

### 4. Multi-entity atomic transactions

```cypher
// Transfer an edge from one vertex to another — atomically
MATCH (a)-[r:OWNS]->(item) WHERE a.name = 'Alice'
DELETE r
CREATE (b:Person {name:'Bob'})-[:OWNS]->(item)
```

TAO: no cross-shard transactions. Two independent API calls that could partially fail.

---

## What TAO Can Do That Neptune Cannot

### 1. Billion-reads-per-second scale

Neptune: max 15 replicas. TAO: thousands of follower servers.

### 2. Sub-millisecond read latency

TAO: 1ms p50 on cache hits (96.4% hit rate). Neptune: no multi-tier caching — typical 5-15ms.

### 3. Multi-region with local reads

TAO: full copy per region, all reads local. Neptune: single-region by default.

### 4. Predictable, fixed-cost queries

Every TAO call: bounded cost (one index lookup, one range scan). Neptune: unbounded — traversal touching millions of vertices can take seconds.

---

## Performance Comparison

| Dimension | TAO (2013) | Neptune (typical) |
|---|---|---|
| Read latency (p50) | 1-1.3ms (cache hit) | 5-15ms (simple lookup) |
| Read latency (cache miss) | 5-8ms | Same range (SSD) |
| Write latency | 12ms (local), 74ms (cross-region) | 10-50ms (depends on transaction complexity) |
| Throughput (reads) | 1 billion/sec (thousands of machines) | Thousands to low millions/sec (max 16 instances) |
| Throughput (writes) | Millions/sec | Thousands to tens of thousands/sec |
| Max read replicas | Thousands of followers | 15 |
| Consistency | Eventual (read-after-write within tier) | ACID snapshot isolation |
| Multi-region | Native | Optional (Global Database) |
| Query complexity | O(1) per API call (bounded) | O(unbounded) |

**Why TAO is orders of magnitude faster:**
1. In-memory caching tier (96.4% never touch disk)
2. Fixed query complexity (every call is bounded)
3. Constrained API (only 5 operations)
4. Horizontal read scaling (thousands of servers vs. 15 replicas)

**Why Neptune provides what TAO cannot:**
1. Arbitrary traversals of unknown depth
2. ACID multi-entity transactions
3. Automatic reverse indexing
4. Ad-hoc queries without pre-planned access patterns

---

## Real-World Use Cases

### Fraud Detection

- **Careem** (50M users, ride-hailing): Detecting fraud rings — accounts sharing devices, phones, payment methods. Unbounded traversal: "all accounts connected to X within N hops."
- **Infinitium** (financial services): Replaced rule-based fraud with graph-based pattern detection. Catches novel patterns rules can't express.

**Why graph?** Fraudsters form networks. Detection requires traversing relationships of unbounded depth.

### Identity Graphs

- **Cox Automotive** (Autotrader, Kelley Blue Book): Linking a person across devices, emails, cookies, sessions. Identity resolution = merging clusters of connected identifiers.

**Why graph?** Each signal (same WiFi, same card) adds an edge. Finding the full identity cluster is a traversal.

### Knowledge Graphs / GraphRAG

- **Alexa (Amazon):** Knowledge graph powering "who directed Inception?" Entity-relationship traversal.
- **BMW:** 10PB+ data hub. Neptune + Bedrock for GraphRAG — graph provides structured context to LLM.
- **Trend Micro:** Security knowledge graph for GenAI assistant. Connects threat data across multiple hops.

**Why graph?** Knowledge is relationships between concepts. Answering questions = traversing those relationships.

### Security & Compliance

- **Wiz:** 100s of billions of nodes. "Is there a path from internet to this database through any misconfigured security group?" Reachability query.
- **JupiterOne:** Cloud asset relationships for HIPAA/PCI. "All data stores reachable from public subnet containing PII."

**Why graph?** Security questions are reachability questions. "Can X reach Y through any path?" = graph traversal.

### Supply Chain & Regulatory

- **Merck:** Graph of materials, processes, suppliers, regulations. "If I change this supplier, what regulatory approvals are affected downstream?" Cascading dependency traversal.
- **Siemens Energy:** Turbine spare parts. Parts → turbines → maintenance → failures.

**Why graph?** Supply chains are networks with cascading dependencies. Impact analysis requires variable-depth path traversal.

### Recommendations

- **Smartsheet:** "People who use this feature also use these." Collaborative filtering as graph proximity.
- **Dream11** (220M users): Player/team suggestions based on social graph and behavior.

**Why graph?** "Similar to people like you" = "entities close to you in the graph that you haven't interacted with."

---

## When to Use Neptune vs. Other Options

| Your workload | Use | Why |
|---|---|---|
| Variable-depth traversals are PRIMARY (fraud, social distance, paths) | **Neptune** | Only graph DB handles unbounded traversal efficiently |
| Fixed access patterns, extreme scale, low latency (feeds, timelines) | **DynamoDB** (TAO-like) | Hierarchical model + known patterns → predictable performance |
| M:N relationships, complex queries, evolving access patterns | **Aurora PostgreSQL** | Relational flexibility + recursive CTEs handle most graph-ish queries |
| Graph queries are secondary (1-2 hops, known) alongside relational | **Aurora PostgreSQL + recursive CTEs** | Avoid adding a database for a side use case |
| Knowledge graphs, RDF, semantic web, linked data | **Neptune (SPARQL)** | Purpose-built for RDF standards |

---

## The Fundamental Tradeoff: Neptune vs. TAO

```
Constrained ────────────────────────────────────── Flexible
Fast ──────────────────────────────────────────── Powerful

TAO / DynamoDB                               Neptune / Neo4j
  │                                                │
  │ Fixed patterns                   Arbitrary queries
  │ Billions of reads/sec           Thousands of reads/sec
  │ Sub-ms latency                  5-50ms+ latency
  │ Eventually consistent           ACID
  │ No traversals                   Unbounded traversals
  │ Scale: horizontal (thousands)   Scale: vertical + 15 replicas
  │                                                │
  │ "I know exactly how I'll query"  "I don't know what I'll ask"
  └────────────────────────────────────────────────┘
```

TAO succeeds because it's the **hierarchical/IMS model** (constrained access patterns → extreme optimization). Neptune serves the **network/CODASYL niche** (flexible graph traversal) that relational can't efficiently handle.
