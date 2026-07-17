# Data Store Selection Framework (AWS)

**Date:** 2026-07-17
**Source:** Derived from [What Goes Around Comes Around (2024)](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf) principles applied to AWS services.

**Related entries:**
- [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) — Historical context and principles behind this framework
- [005-facebook-tao.md](005-facebook-tao.md) — Case study: Facebook chose MySQL (relational) + constrained API over a general graph DB, validating the "know your access patterns" principle

---

## The Core Decision: What Are You Giving Up?

Every non-relational choice trades away something from the relational model. The question isn't "which database is best?" but "which relational principle can I afford to sacrifice for my specific workload?"

| Relational Principle | What it gives you | What sacrificing it costs you |
|---|---|---|
| Declarative queries (SQL) | Ad-hoc questions, optimizer handles access | Must know access patterns upfront |
| Schema enforcement | Data integrity, validation at DB layer | App code becomes the schema enforcer |
| Joins | Normalize data, no duplication | Denormalize → update anomalies, data drift |
| ACID transactions | Correctness across multiple writes | App-level coordination, eventual consistency |
| Data independence | Change storage without rewriting apps | Access pattern changes → table redesign |

---

## AWS Services Mapped to Data Model Eras

| AWS Service | Data Model Era | Equivalent Historical System |
|---|---|---|
| DynamoDB | Hierarchical (1968) | IMS — tree-structured, predefined access paths |
| DocumentDB | Hierarchical (1968) | IMS with richer query (but still hierarchical) |
| Neptune | Network/Graph (1970s) | CODASYL — nodes and edges, navigational |
| Aurora / RDS PostgreSQL | Relational (1980s) | The 50-year winner |
| ElastiCache (Redis) | Key-Value | Not really a data model — raw lookup |
| OpenSearch | Inverted index + documents | Specialized retrieval (search, not storage) |
| Redshift | Relational (columnar) | Relational optimized for analytics |
| Athena | Relational (federated) | SQL over files (schema-on-read, but still SQL) |
| Timestream | Relational (time-series) | Specialized relational variant |

---

## The Decision Tree

### Question 1: Do you know your access patterns at design time?

This is the single most important question.

**"No, or they'll evolve"** → **Relational (RDS / Aurora PostgreSQL)**
- Optimizer handles ad-hoc queries
- Joins for questions you haven't thought of yet
- Schema changes are ALTER TABLE, not table redesigns
- Examples: internal tools, admin dashboards, reporting, any system where PMs will ask "can we also query by X?"

**"Yes, and they're locked in"** → DynamoDB is a candidate. Continue to Question 2.

---

### Question 2: What's your relationship structure?

**Primarily hierarchical (1:N, nested entities):**
→ **DynamoDB** or **DocumentDB**

Example: An order contains line items. Always accessed together.
```
PK: ORDER#123
SK: ITEM#1  → {product: "Widget", qty: 2}
SK: ITEM#2  → {product: "Gadget", qty: 1}
SK: META    → {customer: "Smith", date: "2026-07-17"}
```

**Many-to-many relationships are central:**
→ **Relational (Aurora / RDS PostgreSQL)**

Example: Products ↔ Categories, Customers ↔ Products. In DynamoDB, you'd need GSIs and data duplication for every access pattern. In PostgreSQL, it's just JOIN.

**Deep, variable-depth graph traversals:**
→ **Neptune**

Example: "Find all users within 6 degrees of connection who also liked X." Depth is unknown. Genuinely hard in SQL (recursive CTEs get ugly past 3-4 levels). But ask: is this your *primary* workload or just one query? If one query, PostgreSQL recursive CTEs may suffice.

---

### Question 3: Scale and latency requirements?

**Predictable single-digit-ms latency at any scale (100K+ req/s):**
→ **DynamoDB**

Paper's lesson: DynamoDB gives IMS-like performance for IMS-like access patterns. Trading query flexibility for performance certainty.

**Moderate scale (up to ~128K connections, heavy reads):**
→ **Aurora PostgreSQL** (up to 15 read replicas, auto-scaling storage)

**Moderate scale, cost-sensitive, full control:**
→ **RDS PostgreSQL**

**Massive read throughput on narrow key space (caching, sessions):**
→ **ElastiCache (Redis) / MemoryDB**

Not a data model choice — a caching layer. The paper: "not a data model, just a performance optimization."

---

### Question 4: Consistency requirements?

**Strong consistency, multi-row transactions:**
→ **Aurora / RDS** (full ACID)
→ **DynamoDB Transactions** (limited — up to 100 items, same partition preferred)

**Eventual consistency is fine:**
→ **DynamoDB** (default reads eventually consistent, half cost)
→ **DynamoDB Global Tables** (cross-region, eventually consistent replication)

**Warning from history:** People underestimate how often they need transactions. "We don't need transactions" often becomes "we have data corruption bugs" 6 months later.

---

### Question 5: OLTP or OLAP?

**OLTP (many small reads/writes, user-facing):**
→ **DynamoDB** (if access patterns fixed) or **Aurora** (if flexible queries needed)

**OLAP (complex aggregations, scans across millions of rows):**
→ **Redshift** (columnar, optimized for aggregation)
→ **Athena** (serverless, query S3 data lakes with SQL)

---

## The Practical Checklist

For each new service/table:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ACCESS PATTERNS                                          │
│    □ Can I enumerate ALL access patterns today?             │
│    □ Will product requirements change them in 6 months?     │
│    □ Will anyone need ad-hoc queries (analytics, debugging)?│
│                                                             │
│    ALL known + stable  → DynamoDB candidate                 │
│    Any unknown/evolving → Relational (Aurora/RDS)           │
├─────────────────────────────────────────────────────────────┤
│ 2. RELATIONSHIPS                                            │
│    □ Mostly 1:1 or 1:N (hierarchical)?                     │
│    □ M:N relationships queried both ways?                   │
│    □ Variable-depth path traversals?                        │
│                                                             │
│    Hierarchical only   → DynamoDB / DocumentDB              │
│    M:N central         → Relational                         │
│    Deep graph traversal → Neptune (or Relational + CTEs)    │
├─────────────────────────────────────────────────────────────┤
│ 3. SCALE & LATENCY                                          │
│    □ Peak requests/second?                                  │
│    □ Latency SLA (p99)?                                     │
│    □ Data size (GB/TB/PB)?                                  │
│                                                             │
│    <10K rps, moderate size    → RDS PostgreSQL              │
│    10K-100K rps, read-heavy   → Aurora (read replicas)      │
│    >100K rps, predictable ms  → DynamoDB                    │
│    Analytical / PB-scale      → Redshift / Athena           │
├─────────────────────────────────────────────────────────────┤
│ 4. CONSISTENCY                                              │
│    □ Multi-entity transactions needed?                      │
│    □ Tolerate stale reads (seconds)?                        │
│    □ Cross-region active-active needed?                     │
│                                                             │
│    Multi-row ACID required    → Aurora/RDS                   │
│    Single-item or eventual OK → DynamoDB                    │
│    Cross-region active-active → DynamoDB Global Tables      │
├─────────────────────────────────────────────────────────────┤
│ 5. SCHEMA EVOLUTION                                         │
│    □ Schema changes frequently?                             │
│    □ Records are heterogeneous (different fields)?          │
│                                                             │
│    Frequent changes, heterogeneous → DynamoDB / DocumentDB  │
│    (But: you'll enforce schema in app code)                 │
│    Stable schema, homogeneous → Relational                  │
├─────────────────────────────────────────────────────────────┤
│ 6. QUERY COMPLEXITY                                         │
│    □ JOINs across entities?                                 │
│    □ GROUP BY, aggregations, window functions?              │
│    □ Full-text search?                                      │
│    □ Similarity/vector search?                              │
│                                                             │
│    Complex SQL (joins, agg)   → Aurora/RDS                  │
│    Full-text search           → OpenSearch                  │
│    Vector similarity          → OpenSearch / pgvector        │
│    Simple key lookups         → DynamoDB / ElastiCache      │
└─────────────────────────────────────────────────────────────┘
```

---

## The Paper's Warnings Applied to AWS

### DynamoDB = IMS (hierarchical, 1968)

DynamoDB's single-table design with composite sort keys is exactly the hierarchical model. Phenomenal when data fits a tree and access patterns are fixed. Painful when:
- New access pattern → add a GSI (max 20) or remodel table
- Many-to-many → duplicate data across items, maintain consistency yourself
- Ad-hoc queries → you can't. Full scan or nothing.

**Paper predicts:** Teams choosing DynamoDB for flexibility often add more GSIs, duplicate data, build app-level join logic — gradually reinventing a relational database, badly.

### DocumentDB = IMS with JSON

Same tradeoffs as DynamoDB with richer query language ($lookup = slow join). Documents are hierarchical — if data isn't a tree, you'll fight the model.

### Neptune = CODASYL (network/graph, 1970s)

Great for genuinely graph-shaped problems. But: most data isn't primarily a graph. If doing graph queries on fundamentally relational data, PostgreSQL + recursive CTEs is simpler and sufficient.

### Aurora/RDS PostgreSQL = Relational (the 50-year winner)

The safe default. Lose DynamoDB's infinite scaling and guaranteed single-digit-ms at extreme throughput. Gain ability to ask any question about your data without redesigning storage.

---

## The Rule of Thumb

> **Start with PostgreSQL (Aurora). Move to DynamoDB only when you can prove:**
> 1. Your access patterns are stable and enumerable
> 2. Your relationships are hierarchical (not M:N)
> 3. You need scale/latency that relational can't provide
> 4. You accept that new access patterns will be expensive to add
>
> If you can't prove all four, 55 years of history says you'll regret the non-relational choice within 18 months.

---

## Quick Reference: When to Use What

| Use Case | AWS Service | Why |
|---|---|---|
| General-purpose OLTP, evolving product | Aurora PostgreSQL | Flexible queries, ACID, schema evolution |
| High-scale, fixed access patterns, low latency | DynamoDB | Hierarchical model with predictable performance |
| Shopping cart, sessions, caching | ElastiCache Redis | Key-value lookup, ephemeral data |
| Social graph, fraud detection, deep traversals | Neptune | Graph traversal is the primary operation |
| Full-text search, log analytics | OpenSearch | Inverted index, specialized retrieval |
| Business analytics, aggregations over TB+ | Redshift | Columnar, optimized for scans and GROUP BY |
| Ad-hoc queries over data lake (S3) | Athena | Serverless SQL, pay per query |
| AI embeddings, semantic search | pgvector on Aurora / OpenSearch | Feature, not a standalone database |
| Cross-region active-active, eventual consistency | DynamoDB Global Tables | Multi-region replication built-in |
| Document management, heterogeneous records | DocumentDB | Richer query than DynamoDB, still hierarchical |
