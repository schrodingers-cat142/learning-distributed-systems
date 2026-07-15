# Exponential Backoff and Jitter

**Date:** 2026-07-15
**Sources:**
- [Exponential Backoff And Jitter — AWS Architecture Blog (2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Backoff doesn't always help — Marc Brooker (2022)](https://brooker.co.za/blog/2022/08/11/backoff.html)

**Related entries:**
- [002-retry-policies.md](002-retry-policies.md) — Retry admission control (whether to retry at all) complements this entry on retry timing
- [005-facebook-tao.md](005-facebook-tao.md) — TAO uses bounded serial clients (leader serialization) where backoff principles directly apply

---

## Part 1: The Original Insight (AWS Blog, 2015)

### The Problem: Thundering Herd / Contention

When multiple clients compete for a shared resource and fail, naive retries cause them to collide again and again at the same instant. This creates a self-reinforcing cycle of contention.

### What "Work Done" Means

"Work done" = the **total number of calls (attempts) made by all clients** before every client has successfully completed its request. It measures wasted effort — lower is better.

Example: 10 clients contending for a resource. If it takes 50 total attempts across all clients before all 10 succeed → work done = 50.

### Exponential Backoff

Instead of retrying immediately, wait an exponentially increasing amount of time:

```
attempt 1: wait 1s
attempt 2: wait 2s
attempt 3: wait 4s
attempt 4: wait 8s
...
general: wait min(cap, base * 2^attempt)
```

This spreads retries over time. But if all clients use the same formula, they still collide — just at later, synchronized times.

### Jitter

Add randomness to the wait time so clients naturally desynchronize.

Three variants tested in the blog:

**Full Jitter:**
```
sleep = random_between(0, min(cap, base * 2^attempt))
```

**Equal Jitter:**
```
temp = min(cap, base * 2^attempt)
sleep = temp/2 + random_between(0, temp/2)
```

**Decorrelated Jitter:**
```
sleep = min(cap, random_between(base, sleep_prev * 3))
```

### Results from Simulation

| Strategy | Work Done | Completion Time |
|----------|-----------|-----------------|
| No jitter (plain exponential) | High | High |
| Full jitter | Lowest | Lowest |
| Equal jitter | Low | Low |
| Decorrelated jitter | Low | Low |

Full jitter wins on both metrics. The key insight: jitter eliminates the "synchronized collision" problem that plain exponential backoff still suffers from.

---

## Part 2: The Correction — When Backoff Does NOT Help (Brooker, 2022)

Marc Brooker (co-author of the original post) published a correction: backoff only reduces total work **if it actually reduces the load entering the system**, not just delays it.

### The Critical Question: Are Your Clients Bounded and Serial?

#### Scenario 1: Short Spike, Fixed Clients → BACKOFF WORKS

**Setup:** 100 background workers restart simultaneously after a deployment. They all hit a database at once.

```
Time 0:  100 workers all send request → DB handles 10 at a time → 90 fail
Time 1:  Without backoff: 90 retry immediately → collide again
         With backoff+jitter: ~15 retry (others waiting) → spread out naturally
```

Works because:
- Fixed number of clients (100 workers)
- Once a worker succeeds, it stops retrying
- The overload is temporary — once the burst clears, system is fine

#### Scenario 2: Sustained Overload, Many Independent Clients → BACKOFF DOES NOT WORK

**Setup:** Server handles 100 req/s. A viral tweet sends 200 users/second continuously.

```
Without backoff:
  Second 1: 200 arrive → 100 succeed, 100 fail
  Second 2: 200 NEW + 100 retries = 300 → death spiral

With backoff:
  Second 1: 200 arrive → 100 succeed, 100 fail (retry in ~2s)
  Second 2: 200 NEW arrive → 100 succeed, 100 fail (retry in ~2s)
  Second 3: 200 NEW + 100 deferred retries from second 1 = 300 → still overloaded
```

Backoff just shifted retries later in time. Total work is the same because:
- New clients don't know about the overload (user #500 at second 5 has no idea)
- Clients are independent (one user backing off doesn't prevent another from arriving)
- The overload isn't temporary (200 req/s keeps coming)
- Worse: deferred retries pile up as a hidden queue, delaying recovery

#### Scenario 3: Serial/Bounded Workers → BACKOFF WORKS

**Setup:** 10 worker processes polling a queue in a loop.

```python
while True:
    message = queue.get()       # "first try"
    result = process(message)
    api.submit(result)          # if this fails, retry with backoff
```

Without backoff: worker loops as fast as possible → high request rate.

With backoff: while waiting 4 seconds for a retry, the worker is idle. It can't start its next message. **Total requests per second genuinely drops.**

This is why TCP congestion control works — one connection backs off, it literally sends fewer packets.

### The "First Tries vs. Retries" Framework

**Too many first tries?**
- Serial/bounded clients → backoff helps (slowing retries slows first tries)
- Independent/unbounded clients → backoff doesn't help. Need load shedding, autoscaling, rate limiting.

**Too many retries (retry storm)?**
- Backoff defers them but doesn't eliminate them
- Need an adaptive retry policy (→ see [002-retry-policies.md](002-retry-policies.md))

### Summary Table

| Situation | Does backoff help? | What you actually need |
|-----------|-------------------|----------------------|
| Short spike, fixed clients | Yes | Backoff + jitter |
| Sustained overload, many independent clients | No (just delays) | Rate limiting, load shedding, autoscaling |
| Serial/bounded workers | Yes | Backoff + jitter |
| Retry storms after transient failure | Partially | Backoff + jitter + circuit breakers/token buckets |

---

## Key Takeaway

Backoff is a **timing strategy** (when to retry), not a **load management strategy** (whether to retry or how much load to accept). It works when the retrying client IS the source of excess load (serial workers, thundering herd). It does nothing when excess load comes from sources that don't know about or respond to the backoff.
