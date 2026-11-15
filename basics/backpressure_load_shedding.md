# Rate Limiting, Backpressure & Load Shedding

> **TL;DR** — Overload kills systems through **queue growth → latency → timeouts → retries → more load** (a metastable failure). Defend with layered controls: **rate limiting** (cap what each caller can send),
> **backpressure** (tell upstream to slow down: bounded queues, blocking, credit-based flow control, HTTP 429/503 + `Retry-After`), and **load shedding** (deliberately drop low-priority work to protect
> the critical path), plus **timeouts, retries with jittered backoff, circuit breakers, bulkheads**. Rule: *bounded queues, fail fast, prioritize, and never retry without backoff.*
> See [Resilience patterns](resilience_patterns.md), [Rate limiter design](../problems/rate_limiter.md), [Async](asynchronism.md), [Availability](availability.md).

## The overload spiral
```
Burst ─▶ queues grow ─▶ latency ↑ ─▶ client timeouts ─▶ retries ─▶ load ↑↑ ─▶ collapse
            └── unbounded queue hides the problem until memory/latency explodes
```
Little's Law: `L = λ·W`; as service time W grows, in-flight requests pile up; past capacity, throughput falls (goodput drops) unless you shed.

## Layered defenses
| Layer | Mechanism | Where |
|---|---|---|
| **Admission control / rate limiting** | Token bucket, leaky bucket, fixed/sliding window per user/IP/API key/tenant | API gateway, edge, service ([Rate limiter](../problems/rate_limiter.md)) |
| **Concurrency limits** | Max in-flight requests; adaptive (AIMD/gradient, "Netflix concurrency limits") | Service, client |
| **Backpressure** | Bounded queues + block/reject; HTTP 429/503; TCP windows; Reactive Streams demand; Kafka consumer pause; credit-based flow (Flink) | Between stages |
| **Load shedding** | Drop by priority (health checks and paid traffic last), by age (LIFO / drop stale), by cost; return cheap degraded responses | Service edge, queue consumers |
| **Timeouts & deadlines** | Per-call timeout plus propagated deadline so work for abandoned requests stops | Every RPC |
| **Retries** | Exponential backoff + **jitter**, retry budgets (≤10% extra), only idempotent calls ([Idempotency](idempotency.md)) | Clients |
| **Circuit breaker** | Open after failures, half-open probe, fail fast ([Resilience](resilience_patterns.md)) | Callers |
| **Bulkheads** | Separate pools/queues per dependency or tenant | Service internals |
| **Graceful degradation** | Serve cached/stale/partial content, disable features | App |
| **Autoscaling** | Add capacity (slow, needs headroom) | Infra |

## Rate-limiting algorithms (quick)
| Algorithm | Behavior | Notes |
|---|---|---|
| Token bucket | Refill rate r, burst b | Allows bursts, most common |
| Leaky bucket | Constant outflow | Smooths traffic |
| Fixed window | Counter per window | Boundary burst (2× at edges) |
| Sliding window log/counter | Accurate rolling limit | More memory/compute |
Distributed limits: central Redis (Lua/atomic `INCR`+TTL), or local limiters + periodic sync; accept slight over-admission ([Rate limiter](../problems/rate_limiter.md)).

## Backpressure patterns
- **Bounded queue + reject**: when full, return 429/503 immediately (fail fast) instead of buffering forever.
- **Pull-based consumers** (Kafka, SQS): consumers control pace; lag is the signal ([Kafka](kafka.md), [SQS](sqs.md)).
- **Credit-based/reactive**: downstream grants N credits; upstream sends ≤ N.
- **Client behavior**: honor `Retry-After`, use backoff + jitter, circuit-break.
- **Write path**: when compaction/replication lags, throttle writers (RocksDB write stalls, Cassandra backpressure) ([WAL & LSM](wal_lsm.md)).

## Shedding policy
1. Classify traffic (critical, normal, best-effort; per tenant tiers).
2. Shed best-effort first; keep health checks, auth, payments.
3. Prefer **dropping old requests** (their callers may have timed out) over new ones.
4. Make shed decisions **cheap** (before expensive work) and **observable** (metrics, 429 vs 503).

## Pitfalls
- Unbounded queues; infinite/synchronized retries (retry storms); retrying non-idempotent writes; shedding after doing the expensive work; autoscaling as the only defense; no per-tenant isolation (noisy neighbor); rate limiter as a single point of failure (fail open vs closed decision).

## Interview follow-ups
- *"Traffic spikes 10×. What happens?"* Rate limit per client, shed low priority, bounded queues with backpressure, degrade features, autoscale behind it.
- *"Downstream is slow; how do you prevent cascade?"* Timeout + circuit breaker + bulkhead + fallback; don't retry blindly.
- *"Where do you put the rate limiter?"* Edge/gateway for coarse limits, service-level for fine-grained or cost-based limits.
- *"Fail open or closed?"* Open for availability of non-sensitive APIs; closed for abuse-sensitive/payment endpoints.

# References
* [AWS Builders' Library: Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/), [Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
* [Google SRE Book: Handling Overload](https://sre.google/sre-book/handling-overload/), [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
