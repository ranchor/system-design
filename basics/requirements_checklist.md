# Functional vs Non-Functional Requirements Checklist

> **TL;DR** — **Functional requirements** say *what the system does* ("users can post a photo"); **non-functional requirements (NFRs)** say *how well* it does it
> (scale, latency, availability, consistency, durability, security, cost). In the interview, pick **~3 functional requirements** (state what's below the line)
> and **3-5 NFRs with numbers**. NFRs drive the architecture: they decide whether you need caching, sharding, queues or multi-region.
> See the [delivery framework](interview_framework.md) and [key characteristics](key-characteristics.md).

## Functional requirements
Phrase as "Users should be able to …". Prioritize; explicitly put the rest **below the line**.
- Core actions (create, read, update, delete, search, share, pay, book).
- Actors/roles (user, admin, driver, merchant) and permissions.
- Important edge behaviors: ordering, uniqueness, cancellation, refunds, notifications.
- **Clarifying questions**: Who are the users? Mobile/web? One region or global? Real-time or eventual? What's the read/write ratio? What is out of scope (analytics, admin, payments)?

## Non-functional checklist (ask or state each)
| Area | Question | Typical interview answer / consequence |
|---|---|---|
| **Scale** | DAU/MAU, peak QPS, data size/growth, read:write ratio | "100M DAU, 10:1 reads, ~1M QPS peak" → caching, sharding, CDN |
| **Latency** | p50/p99 per operation; user-facing vs background | "Feed < 200 ms p99" → cache, precompute, avoid fan-in at read time |
| **Availability** | Target nines, tolerate region loss? | "99.99%" → multi-AZ, no SPOF, failover ([Availability](availability.md)) |
| **Consistency** | Strong, eventual, read-your-writes? Per feature | "Booking strong, browsing eventual" ([CAP/PACELC](cap_theorem.md), [Consistency](consistency.md)) |
| **Durability** | Can we ever lose data? RPO/RTO | Payments: replicated + backups; likes: loss tolerable ([Replication](data_replication.md)) |
| **Throughput / burst** | Spiky traffic (flash sale, live event)? | Queue to level load, rate limit, autoscale |
| **Ordering / exactly-once** | Must events be ordered or deduplicated? | Per-key partitioning, idempotency ([Delivery semantics](delivery_semantics.md)) |
| **Security / privacy** | AuthN/Z, PII, encryption, compliance (GDPR, PCI) | ([Security](security.md), [AuthN/AuthZ](authn_authz.md)) |
| **Fault tolerance** | Node/AZ/region failure behavior; graceful degradation | Replicas, retries, circuit breakers ([Resilience](resilience_patterns.md)) |
| **Cost / operability** | Budget, team size, observability | Managed services, metrics, alerts ([Monitoring](monitoring.md)) |
| **Compliance / data residency** | Region-bound data? Retention? | Geo-sharding, TTL, deletion pipelines |
| **Extensibility** | Likely future features | Clean boundaries, event-driven |

## Turning NFRs into numbers (30-second math)
- DAU × actions/day ÷ 86,400 = average QPS; peak ≈ 2-10× average. Storage = items/day × size × retention × replication.
- Example: 10M DAU × 5 posts viewed… see [Back of Envelope](back_of_envelope_estimation.md).

## Good vs weak NFR statements
| Weak | Strong |
|---|---|
| "Should be fast and scalable" | "p99 read < 100 ms at 50k QPS; scales horizontally to 10× in a year" |
| "Highly available" | "99.99% for reads; writes may degrade during a region failover" |
| "Consistent" | "Seat booking is strongly consistent; browse is eventually consistent (≤ 5 s)" |
| "Secure" | "Auth via OAuth2, PII encrypted at rest, per-tenant isolation" |

## Examples
- **URL shortener**: FR: create short link, redirect, optional expiry. NFR: redirect p99 < 50 ms, 100:1 read:write, 99.99%, unique codes, eventual analytics.
- **Ticketing**: FR: browse events, book a seat. NFR: no double-booking (strong), surge handling (queue/waiting room), browse available.
- **Chat**: FR: 1:1/group messages, history, presence. NFR: low latency delivery, ordering per conversation, at-least-once with dedup, offline sync.

## Pitfalls
- Listing 10 functional requirements; skipping NFRs; vague adjectives instead of numbers; forgetting consistency per feature.
- Asking questions you could state as assumptions; the interviewer wants decisions, not an interrogation.

# References
* [Hello Interview: Requirements](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery), [Key Characteristics](key-characteristics.md)
