# Distributed Systems Learning Notes

Running notes from reading DDIA, blog posts, and discussions. Each file is a self-contained deep-dive on a topic, written to be used as reference input to AI systems.

## Index

| # | Topic | Source | Date | File |
|---|-------|--------|------|------|
| 1 | Exponential Backoff and Jitter | [AWS Architecture Blog (2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) + [Brooker (2022)](https://brooker.co.za/blog/2022/08/11/backoff.html) | 2026-07-15 | [001-backoff-and-jitter.md](001-backoff-and-jitter.md) |
| 2 | Retry Policies: Token Buckets and Circuit Breakers | [Brooker (2022)](https://brooker.co.za/blog/2022/02/28/retries.html) | 2026-07-15 | [002-retry-policies.md](002-retry-policies.md) |
| 3 | Data Models: What Goes Around Comes Around | [Stonebraker & Pavlo (2024)](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf) | 2026-07-17 | [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) |
| 4 | Data Store Selection Framework (AWS) | Derived from entry #3 | 2026-07-17 | [004-data-store-selection-framework.md](004-data-store-selection-framework.md) |

## How These Connect

```
Designing a system
    │
    ├─► [Data Model Selection] ── What store do I use?       → Entry #3, #4
    │                              (relational vs. hierarchical vs. graph)
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
