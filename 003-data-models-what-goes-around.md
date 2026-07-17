# What Goes Around Comes Around: Data Models Through History

**Date:** 2026-07-17
**Source:** [What Goes Around Comes Around... And Around — Stonebraker & Pavlo (2024)](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf)

**Related entries:**
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — Practical AWS decision framework derived from this paper's principles
- [005-facebook-tao.md](005-facebook-tao.md) — Real-world example: constrained graph API on MySQL (hierarchical access patterns, not a general graph DB)

---

## The Big Idea

Data model ideas keep recycling every ~15 years, each generation often unaware of prior art, and the same fundamental principles determine which ones win and which ones die — every single time.

---

## The Principles That Always Win

Every data model in 55 years of history can be evaluated against these five principles. The ones that provide them survive; the ones that sacrifice them die or remain niche.

1. **Physical data independence** — applications shouldn't break when you change storage layout
2. **Declarative queries** (set-at-a-time) beat navigational/procedural (record-at-a-time)
3. **Schema is inevitable** — "schema-last" always drifts back to "schema-first" at scale
4. **Simplicity wins** — a "good enough" simple model beats an expressive complex one
5. **Automatic query optimization** is essential — humans can't hand-optimize at scale

---

## Era-by-Era Summary

### Era 1: Hierarchical Model (IMS, late 1960s)

**What:** Data organized as trees. IBM's IMS. Parent-child records.

**Example:**
```
Customer
├── Order 1
│   ├── Item A
│   └── Item B
└── Order 2
    └── Item C
```

**Strength:** Blazing fast for predefined access paths. If you always access "customer → orders → items," it's optimal.

**Fatal flaw:** Real data isn't a tree. A part can belong to multiple orders (many-to-many). You either duplicate data or hack in "logical pointers" that make it a mess.

**Died because:** Too rigid. Can't represent general relationships without contortions.

---

### Era 2: Network Model (CODASYL, early 1970s)

**What:** Data as a general graph. Records connected by named "sets" (edges). You navigate record-by-record using "currency indicators" (cursors tracking position).

**Strength:** Can represent any relationship. More flexible than hierarchical.

**Fatal flaw:** Programming was brutal. Procedural, record-at-a-time navigation:
```
FIND FIRST Customer WHERE name = "Smith"
FIND FIRST Order WITHIN Customer-Orders
FIND FIRST Item WITHIN Order-Items
GET Item
FIND NEXT Item WITHIN Order-Items
```

Programs were tightly coupled to physical data layout. Change structure → rewrite all programs.

**Died because:** Procedural navigation too complex. No data independence. Programmers drowned in "currency indicators."

---

### Era 3: Relational Model (Codd, 1970 → dominance by late 1980s)

**What:** Data as tables. Queries in SQL (declarative — say WHAT you want, not HOW). Optimizer figures out access paths.

**Why it won the "Great Debate" against CODASYL:**
- Simple: anyone understands tables
- Declarative: `SELECT * FROM orders WHERE customer_id = 5` vs. pages of navigational code
- Data independence: change indexes, reorganize storage → apps don't break
- Optimizer: explores 1000 access paths and picks the best; humans can't

**The debate was vicious** (1970s). CODASYL advocates said relational would be too slow. They were wrong — optimizers got good enough, and programmer productivity mattered more than micro-optimization.

**This is THE reference outcome.** Every subsequent model is measured against relational.

---

### Era 4: Entity-Relationship (Chen, 1976)

**What:** Conceptual design notation (entities, relationships, attributes). Boxes and diamonds on whiteboards.

**Outcome:** Universal *design tool* but never an implementation model. Draw E-R diagrams → translate to relational tables. Still used today.

---

### Era 5: Semantic Data Models (late 1970s-1980s)

**What:** Richer meaning — IS-A hierarchies (Employee IS-A Person), aggregation, classification.

**Why it failed:** Too many competing proposals, no consensus, no standard query language. Added expressiveness wasn't worth the complexity. Relational was "good enough."

---

### Era 6: Object-Oriented Databases (late 1980s-1990s)

**What:** Persist programming-language objects directly. ObjectStore, O2, Versant, GemStone. Objects have OIDs, inheritance, encapsulation, methods.

**The pitch:** "No more impedance mismatch! Your Java/C++ objects live directly in the database!"

**Why it failed (devastating critique):** It was CODASYL with different syntax. Navigated through object pointers record-by-record. Lost declarative queries. Lost data independence.

```java
// OO database navigation — looks like CODASYL
Customer c = db.lookup("Smith");
for (Order o : c.getOrders()) {
    for (Item i : o.getItems()) {
        // process item
    }
}
```

**Market:** Tiny niche (CAD/CAM). Most OO database companies went bankrupt.

**The "comes around" observation:** OO databases repeated the network model's mistakes 20 years later, apparently without knowing it.

---

### Era 7: Object-Relational (1990s)

**What:** Keep relational + SQL but add user-defined types, functions, inheritance. PostgreSQL (originally Postgres), then Oracle, DB2, Informix.

**Why it worked:** Evolutionary, not revolutionary. Keep data independence, keep declarative queries, keep SQL — just extend what a "column type" can be.

**Lesson:** Pragmatic middle ground wins. Don't throw out the baby with the bathwater.

---

### Era 8: Semi-Structured / XML (late 1990s-2000s)

**What:** Self-describing data without rigid schema. XML, DTDs, XQuery, XPath. "Schema-last."

**The devastating observation:** XML is a hierarchical data model. **It's IMS from 1968.** Thirty years later, same thing with angle brackets.

**Same problems reappeared:**
- Many-to-many relationships are awkward
- Duplicate data or use ID references (logical pointers... like IMS)
- XQuery is complex
- Performance worse than relational for structured data

**Outcome:** XML databases remained niche. Important for **data exchange** (messages between systems) but failed as a general-purpose database model. Relational absorbed XML features (SQL/XML).

---

### Era 9: MapReduce / Hadoop (2004-2015)

**What:** Google's MapReduce paper (2004) → Hadoop. Write map() and reduce() functions. No schema, no SQL, no optimizer.

**The pitch:** "SQL doesn't scale to petabytes! Just write code!"

**The critique:** CODASYL programming without even a data model. Procedural, no declarative queries, no optimizer, no schema.

**What happened:** Community gradually re-added everything they'd removed:
- Schema → Hive added schema on read
- SQL → Hive, Spark SQL, Presto
- Optimizer → query planning engines
- Data independence → storage layers separated

**Lesson:** You can reject SQL, but you'll spend a decade re-inventing it.

---

### Era 10: NoSQL / Document Stores (2009+)

**What:** MongoDB, CouchDB. JSON documents (nested, hierarchical). Schema-flexible. Initially no joins, no transactions, no SQL.

**The observation:** JSON documents are hierarchical. **IMS. Again.** Third time.

```json
{
  "customer": "Smith",
  "orders": [
    { "id": 1, "items": ["Widget", "Gadget"] },
    { "id": 2, "items": ["Doohickey"] }
  ]
}
```

**Same problems:**
- Many-to-many? Embed duplicates or use references (lose joins)
- No joins → denormalize → update anomalies
- No transactions → eventually added
- No SQL → query languages increasingly look like SQL

**Trajectory:** MongoDB in 2024 looks increasingly like a relational database with JSON syntax. Re-added joins ($lookup), transactions, schemas (validation), aggregation pipelines.

---

### Era 11: Key-Value Stores (Dynamo, Redis, Riak)

**What:** Simplest model — get(key) → value, put(key, value).

**Tradeoff:** Maximum simplicity/scalability, zero query power. All intelligence in application code.

**Use case:** Caching, sessions, simple lookups. Not general-purpose.

**Observation:** Not really a "data model" debate — deliberately choosing no data model for raw performance at a specific task.

---

### Era 12: Graph Databases (Neo4j, 2010s)

**What:** Nodes and edges. Query by traversal (Cypher, Gremlin, SPARQL).

**The observation:** This is the network model (CODASYL) reborn. Nodes ≈ records. Edges ≈ sets. Traversal ≈ navigation.

```cypher
MATCH (c:Customer)-[:PLACED]->(o:Order)-[:CONTAINS]->(i:Item)
WHERE c.name = "Smith"
RETURN i
```

**Where graphs genuinely shine:** Deep traversals (social networks, fraud rings, knowledge graphs) where recursion depth is unknown.

**But:** For most OLTP/analytics, relational still wins. Graph DB market remains small.

---

### Era 13: NewSQL (2010s — Spanner, CockroachDB, TiDB)

**What:** Distributed relational databases with SQL + ACID + horizontal scalability.

**The argument:** You don't have to sacrifice SQL/ACID for scale. NoSQL tradeoffs were a false dilemma.

**Validates the thesis:** Industry spent a decade escaping relational (NoSQL), then built distributed systems that re-implement it (NewSQL).

---

### Era 14: Vector Databases (2023+, Pinecone, Weaviate, Milvus)

**What:** High-dimensional embeddings. Query by similarity (nearest neighbor). For AI/RAG.

**The assessment:** This is a **feature, not a data model.** Being added to existing relational databases (pgvector). Standalone market may not survive as a category.

**Pattern:** Same as XML, JSON, graph — starts standalone, gets absorbed into relational ecosystem.

---

## The Recurring Cycle

```
1968: Hierarchical (IMS)
  ↓ "too rigid"
1970s: Network/Graph (CODASYL)
  ↓ "too complex, navigational"
1980s: Relational (SQL) ← WINS
  ↓ "not expressive enough"
1990s: Object-Oriented DBs
  ↓ "that's just CODASYL again" → FAILS
  ↓
1990s: Object-Relational ← evolutionary extension WORKS
  ↓
2000s: XML
  ↓ "that's just IMS again" → FAILS as standalone
  ↓
2009: NoSQL / Document (JSON)
  ↓ "that's IMS AGAIN" → gradually re-adds SQL features
  ↓
2010s: Graph DBs
  ↓ "that's CODASYL again" → remains niche
  ↓
2010s: NewSQL ← relational principles + scale WORKS
  ↓
2020s: Vector DBs
  ↓ "that's a feature, not a model" → being absorbed into RDBMS
```

---

## Key Lessons

1. **Hierarchical structures keep returning:** IMS → XML → JSON/Document stores. Each generation rediscovers that hierarchical data is intuitive but limited.

2. **Navigation vs. Declaration:** CODASYL, OO databases, graph DBs — navigational keeps losing to declarative (SQL) for general-purpose use.

3. **Schema flexibility vs. rigidity:** Every era debates "schema-first" vs. "schema-last." Flexibility is appealing early but structure becomes necessary at scale.

4. **Physical data independence is crucial:** Systems that sacrifice it (CODASYL, OO databases) ultimately lose.

5. **Complexity is the enemy:** Simpler "good enough" models beat more expressive complex ones.

6. **Features get absorbed:** Every specialized standalone (XML, JSON, graph, vector) either fails or gets absorbed into relational systems as an extension.

---

## Evaluating Any New Database Proposal

Per the paper, ask:
- Does it provide physical data independence?
- Does it have declarative queries with automatic optimization?
- Does it handle schema evolution gracefully?
- Is it simpler than the relational alternative for my use case?
- Or is it just IMS/CODASYL with new syntax?
