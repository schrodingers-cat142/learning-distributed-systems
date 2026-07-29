# Distributed Systems Learning Notes

Running notes from reading DDIA, blog posts, and discussions. Each file is a self-contained deep-dive on a topic, written to be used as reference input to AI systems.

## Index

| # | Topic | Source | Date | File |
|---|-------|--------|------|------|
| 1 | Exponential Backoff and Jitter | [AWS Architecture Blog (2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) + [Brooker (2022)](https://brooker.co.za/blog/2022/08/11/backoff.html) | 2026-07-15 | [001-backoff-and-jitter.md](001-backoff-and-jitter.md) |
| 2 | Retry Policies: Token Buckets and Circuit Breakers | [Brooker (2022)](https://brooker.co.za/blog/2022/02/28/retries.html) | 2026-07-15 | [002-retry-policies.md](002-retry-policies.md) |
| 3 | Data Models: What Goes Around Comes Around | [Stonebraker & Pavlo (2024)](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf) | 2026-07-17 | [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) |
| 4 | Data Store Selection Framework (AWS) | Derived from entry #3 | 2026-07-17 | [004-data-store-selection-framework.md](004-data-store-selection-framework.md) |
| 5 | TAO: Facebook's Distributed Data Store | [Bronson et al. (USENIX ATC 2013)](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf) | 2026-07-19 | [005-facebook-tao.md](005-facebook-tao.md) |
| 6 | Amazon Neptune: Managed Graph Database | [AWS Neptune Docs](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview.html) | 2026-07-19 | [006-amazon-neptune.md](006-amazon-neptune.md) |
| 7 | CQRS and Event Sourcing | [Greg Young (2010)](https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf) + [Fowler: CQRS](https://martinfowler.com/bliki/CQRS.html) + [Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) | 2026-07-19 | [007-cqrs-event-sourcing.md](007-cqrs-event-sourcing.md) |
| 8 | LSM Trees and Storage Engines | [LSM Survey — Luo & Carey (2019)](https://arxiv.org/pdf/1812.07527) | 2026-07-23 | [008-lsm-trees.md](008-lsm-trees.md) |
| 9 | Bigtable: Distributed Storage System | [Chang, Dean et al. (Google, OSDI 2006)](https://storage.googleapis.com/gweb-research2023-media/pubtools/4443.pdf) | 2026-07-26 | [009-bigtable.md](009-bigtable.md) |
| 10 | Trie Memtables in Cassandra | [Lambov (DataStax, PVLDB 2022)](https://www.vldb.org/pvldb/vol15/p3359-lambov.pdf) | 2026-07-26 | [010-trie-memtables-cassandra.md](010-trie-memtables-cassandra.md) |
| 11 | The Ubiquitous B-Tree | [Comer (Purdue, ACM Computing Surveys 1979)](https://web.archive.org/web/20170809145513id_/http://sites.fas.harvard.edu/~cs165/papers/comer.pdf) | 2026-07-26 | [011-b-trees.md](011-b-trees.md) |
| 12 | B-Trees: Modern Deep Dive | [Torn Writes — transactional.blog (2025)](https://transactional.blog/blog/2025-torn-writes) + DDIA Ch. 3 | 2026-07-26 | [012-b-trees-modern.md](012-b-trees-modern.md) |
| 13 | Vector Search: Graph Indexes and Billion-Scale Inverted Indexes | [HNSW — Malkov & Yashunin (2016)](https://arxiv.org/pdf/1603.09320) + [Revisiting Inverted Indices — Baranchuk, Babenko & Malkov (ECCV 2018)](https://arxiv.org/pdf/1802.02422) | 2026-07-26 | [013-vector-search-ann.md](013-vector-search-ann.md) |
| 14 | The RUM Conjecture: Read, Update, Memory — Pick Two | [Athanassoulis et al. (EDBT 2016)](https://openproceedings.org/2016/conf/edbt/paper-12.pdf) | 2026-07-27 | [014-rum-conjecture.md](014-rum-conjecture.md) |
| 15 | Encoding, Schema Evolution, and Dataflow Between Services | DDIA Ch. 4 + [Troy Hunt: API versioning](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/) + [WSO2: Event-Driven Architecture](https://wso2.com/blogs/thesource/exploring-event-driven-architecture-a-beginners-guide-for-cloud-native-developers/) | 2026-07-27 | [015-encoding-and-dataflow.md](015-encoding-and-dataflow.md) |
| 16 | Single-Leader Replication: Fundamentals | DDIA Ch. 5 | 2026-07-29 | [016-single-leader-replication.md](016-single-leader-replication.md) |

## How These Connect

```
Designing a system
    │
    ├─► [Data Model Selection] ── What store do I use?       → Entry #3, #4
    │                              (relational vs. hierarchical vs. graph)
    │
    ├─► [Real-World Graph at Scale] ── How does FB serve      → Entry #5
    │                                   a social graph?
    │                                   (constrained API + caching + MySQL)
    │
    ├─► [General-Purpose Graph DB] ── When do you need       → Entry #6
    │                                  arbitrary traversals?
    │                                  (Neptune: ACID, flexible, slower)
    │
    ├─► [Architecture Patterns] ── How do I structure        → Entry #7
    │                              reads vs. writes?
    │                              (CQRS, Event Sourcing, eventual consistency)
    │
    ├─► [Storage Internals] ── How do databases store       → Entry #8, #9, #10
    │                           data on disk?
    │                           (LSM trees, SSTables, compaction, Bloom filters)
    │                           (Bigtable: full system architecture)
    │                           (Trie memtables: optimizing the memory layer)
    │                           (B-trees: fundamentals)              → Entry #11
    │                           (B-trees: WAL, MVCC, buffer pool,
    │                            CoW, Bε-trees, SSD tradeoffs)     → Entry #12
    │                              │
    │                              └─► [The Lens] ── Why can't one       → Entry #14
    │                                   structure win at everything?
    │                                   (RUM: optimize 2 of Read/
    │                                    Update/Memory, pay on the 3rd —
    │                                    ties #8, #11, #12, #13 together)
    │
    ├─► [Similarity Search] ── How do I search by meaning       → Entry #13
    │                           instead of keywords?
    │                           (ANN: HNSW graphs for in-memory,
    │                            IVF+PQ inverted indexes for
    │                            billion-scale; hybrid w/ lexical)
    │
    ├─► [Data on the Move] ── How do services encode and        → Entry #15
    │                          exchange data as they evolve?
    │                          (JSON/Protobuf/Avro, reader vs
    │                           writer schemas; REST/RPC + API
    │                           versioning; EDA, meshes, actors)
    │                           builds on event sourcing         → Entry #7
    │
    ├─► [Replication] ── How do I keep copies of data        → Entry #16
    │                     across nodes, and stay consistent?
    │                     (single-leader model; sync/async/
    │                      semi-sync; backups vs replication;
    │                      adding followers without downtime)
    │
    └─► [Fault Tolerance] ── How do I handle failures?
            │
            ▼
        [Retry Policy] ── Should I retry at all?             → Entry #2
            │              (token bucket / circuit breaker)
            │
            ▼
        [Backoff + Jitter] ── When should I retry?           → Entry #1
            │                  (exponential backoff + jitter)
            │
            ▼
        [System-Level] ── Will this actually help?           → Entry #1 (Brooker's correction)
                           (bounded clients? short spike?)
```
