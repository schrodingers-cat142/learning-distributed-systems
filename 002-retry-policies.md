# Retry Policies: Token Buckets and Circuit Breakers

**Date:** 2026-07-15
**Source:** [Fixing retries with token buckets and circuit breakers — Marc Brooker (2022)](https://brooker.co.za/blog/2022/02/28/retries.html)

**Related entries:**
- [001-backoff-and-jitter.md](001-backoff-and-jitter.md) — Backoff + jitter (timing of retries) complements this entry on retry admission
- [005-facebook-tao.md](005-facebook-tao.md) — TAO's leader tier rate-limits DB queries (similar to server-side token bucket protecting a resource)

---

## Where This Fits

[Entry #1](001-backoff-and-jitter.md) covers **when** to retry (backoff + jitter = timing).
This entry covers **whether** to retry at all (retry policy = admission control for retries).

```
Request fails
    │
    ▼
[Retry Policy] ── Should I retry at all?         → THIS ENTRY
    │              (token bucket / circuit breaker)
    │              If budget exhausted → give up
    │
    ▼
[Backoff + Jitter] ── When should I retry?        → Entry #1
    │
    ▼
[System-Level] ── Will this actually help?        → Entry #1
```

---

## The Fundamental Tension

Two goals fight each other:

1. **Clients want high success rates** → retry aggressively (3, 5, 10 times)
2. **Servers need protection when struggling** → fewer requests during failure

These goals align at low failure rates (retries are cheap, they help) but **conflict catastrophically** at high failure rates (retries pile on a drowning server).

---

## The Amplification Problem

With N retries and X% failure rate:
- **Effective client failure rate:** X^(N+1) — retries improve availability
- **Server load amplification:** up to (1 + N)× at 100% failure

Example with 3 retries:
- At 5% failure: client sees 0.05⁴ = 0.000006% failure. Excellent!
- At 100% failure: server sees **4× normal load**. Catastrophic — the server is dying and retries are kicking it.

**The cost of retries is highest exactly when the server can least afford it.**

---

## Four Strategies Compared

### Strategy 1: No Retries

```
Client: request → fail → give up
```

- Server load: exactly 1× (no amplification ever)
- Client availability: equals server availability (no resilience)
- Use when: you'd rather fail fast than add load (fire-and-forget, best-effort)

### Strategy 2: Fixed N Retries (Naive)

```
Client: request → fail → retry → fail → retry → fail → retry → give up
```

- Server load: up to (1+N)× during total failure
- Client availability: X^(N+1) effective failure rate (great at low X)
- Problem: **amplification is worst precisely during outages**

Real example: Database is slow from high load → every client retries 3× → database sees 4× traffic → goes from "slow" to "completely dead." Retries caused the outage to cascade.

### Strategy 3: Circuit Breaker

Each client tracks its own recent failure rate. Above a threshold, retries stop entirely.

```
Normal mode:  request → fail → retry → retry → retry (up to N)
Tripped mode: request → fail → give up immediately (no retries)
```

**How it transitions:**
- Track last K requests (or time window)
- If observed failure rate > threshold (e.g., 50%) → trip the breaker
- After some cool-down period, allow a single "probe" request
- If probe succeeds → close breaker, resume retrying

**Pros:**
- Once tripped: zero additional load (protects server completely)
- Simple mental model (on/off)

**Cons:**
- **Modal** — either fully retrying or not retrying at all. Can oscillate: trip → server recovers → un-trip → retry storm → trip again...
- **Trips too early** in practice: with few data points per client, the observed rate is noisy. A client seeing 20 req/min with 50% actual failure might observe 70% in a small window and trip prematurely. Simulation shows it trips at roughly **half the configured threshold**.

### Strategy 4: Token Bucket / Adaptive Retries (Brooker's Favorite)

A smooth dial instead of a binary switch.

**Mechanism:**
- Each client maintains a bucket (e.g., capacity = 10 tokens)
- Every **success** deposits a fraction of a token (e.g., +0.1 tokens)
- Every **retry attempt** costs 1 full token
- If bucket is empty → no retry allowed, give up

**Behavior at different failure rates:**

```
At 2% failure rate:
  98 successes × 0.1 = 9.8 tokens earned
  2 failures × 1 token = 2 tokens spent
  → Bucket stays full, all retries allowed
  → Behaves like N retries (great availability)

At 60% failure rate:
  40 successes × 0.1 = 4 tokens earned
  60 failures want retries, but only 4 tokens available
  → Only 4/60 failures get retried
  → Load amplification: ~1.07× (minimal)
  → Graceful degradation

At 95% failure rate:
  5 successes × 0.1 = 0.5 tokens earned
  → Almost no retries possible
  → Behaves like no retries (protects server)
```

**Why the ratio matters:** 0.1 tokens per success / 1 token per retry = you need 10 successes to earn 1 retry. The system naturally limits retries to ~1/10th of the success rate. This ratio is your tuning knob:
- Higher deposit (0.2) = more aggressive retrying
- Lower deposit (0.05) = more conservative

**Pros:**
- Smooth transition (no cliff/oscillation)
- Self-tuning — adapts to actual failure rate automatically
- Simple to implement (one counter per client)

**Cons:**
- Does allow *some* additional load even at high failure rates (tunable but never zero)
- Depends on per-client state (see "Client Count Problem" below)

---

## Comparison at a Glance

| Failure Rate | No Retry | N Retries | Circuit Breaker | Token Bucket |
|:---:|:---:|:---:|:---:|:---:|
| 1% | 1% client failure | ~0% client failure | ~0% client failure | ~0% client failure |
| 1% server load | 1.01× server load | 1.01× server load | 1.01× server load |
| 50% | 50% client failure | 6.25% client failure | ~50% (tripped) | ~30% client failure |
| 1× server load | ~2× server load | ~1× (tripped) | ~1.1× server load |
| 99% | 99% client failure | 96% client failure | 99% (tripped) | ~99% client failure |
| 1× server load | ~4× server load | 1× (tripped) | ~1.01× server load |

Token bucket achieves the best compromise: near-N-retry availability at low failure rates, near-no-retry server protection at high failure rates, smooth transition between them.

---

## The Client Count Problem (Critical for Modern Architectures)

Both circuit breakers and token buckets rely on **each client's local estimate** of the failure rate. This estimate's accuracy depends on sample size.

### The Problem

- **10 long-lived clients** each seeing 1000 req/min → great statistical accuracy
- **1000 short-lived clients** (containers, Lambda functions) each seeing 10 req/min → noisy estimates

### How They Degrade (Opposite Directions!)

**Circuit Breaker with many small clients:**
- Each client's small sample is noisy
- Random noise pushes observed rate above threshold → premature tripping
- **Degrades toward "no retries" behavior** (trips too easily)
- You lose retry benefits even at low actual failure rates

**Token Bucket with many small clients:**
- Buckets typically start full (avoid cold-start penalty)
- Short-lived client arrives → full bucket → retries freely → dies before depletion
- Next short-lived client arrives → full bucket → same thing
- **Degrades toward "N retries" behavior** (never depletes)
- You lose server protection during high failure

| Many small clients | Circuit Breaker | Token Bucket |
|---|---|---|
| Effect | Over-protects server (trips early) | Under-protects server (buckets stay full) |
| Converges toward | No retries | N retries |
| Problem | Clients lose availability unnecessarily | Server loses protection when it needs it |

### Potential Solution: Shared/Global Retry Budget

Instead of per-client budgets, share state across all clients:
- A central token bucket that all clients draw from
- Or a shared failure-rate signal (e.g., via a sidecar or service mesh)

**Tradeoff:** Significantly more complexity (clients must coordinate, adds a dependency on the shared state itself).

---

## Key Formulas

| Formula | Meaning |
|---------|---------|
| `x^(N+1)` | Effective failure rate with N retries (assuming independence) |
| `1 + N` | Maximum load amplification at 100% failure |
| `deposit/cost` | Token bucket retry ratio (0.1/1 = system earns 1 retry per 10 successes) |
| `threshold/2` | Approximate actual trip point for circuit breakers with many low-volume clients |

---

## Implementation Notes

**Token bucket pseudocode:**
```python
class RetryBudget:
    def __init__(self, capacity=10, deposit_per_success=0.1, cost_per_retry=1.0):
        self.tokens = capacity  # start full
        self.capacity = capacity
        self.deposit = deposit_per_success
        self.cost = cost_per_retry

    def on_success(self):
        self.tokens = min(self.capacity, self.tokens + self.deposit)

    def may_retry(self) -> bool:
        if self.tokens >= self.cost:
            self.tokens -= self.cost
            return True
        return False
```

**Circuit breaker pseudocode:**
```python
class CircuitBreaker:
    def __init__(self, threshold=0.5, window_size=100, cooldown_sec=30):
        self.results = deque(maxlen=window_size)
        self.tripped = False
        self.tripped_at = None

    def record(self, success: bool):
        self.results.append(success)
        failure_rate = 1 - (sum(self.results) / len(self.results))
        if failure_rate > self.threshold:
            self.tripped = True
            self.tripped_at = time.now()

    def may_retry(self) -> bool:
        if not self.tripped:
            return True
        if time.now() - self.tripped_at > self.cooldown_sec:
            return True  # allow probe
        return False
```

---

## Key Takeaway

Retries are not free. Their cost (server load amplification) is highest exactly when the server can least afford it. Good retry design means:
- **Aggressive when failures are rare** (cheap to retry, high benefit)
- **Conservative when failures are common** (expensive to retry, low benefit)

The token bucket naturally encodes this tradeoff via its earn/spend ratio. Combined with backoff + jitter for timing (Entry #1), you get a complete retry system that's both client-friendly and server-protective.
