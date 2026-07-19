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
