## Problem Statement
Design a **Content Delivery Network** (like CloudFront, Akamai, Cloudflare): a globally distributed network of edge caches that serves static and streamable content close to users, reducing latency,
offloading origins, and absorbing traffic spikes/DDoS. Concept note: [CDN](../basics/cdn.md).

## Clarification Questions to Interviewer
- Content types: static assets, images, video (VOD/live), software downloads, dynamic API acceleration?
- Scale: users, regions, peak bandwidth (Tbps), objects/files, hit-ratio target?
- Push (publisher uploads) vs **pull** (edge fetches from origin on miss)? Both?
- Freshness: purge latency, TTL control, versioned URLs?
- Security: signed URLs, TLS termination, WAF/DDoS? Multi-tenant?

## Requirements
### Functional Requirements
- Route a client to a **nearby, healthy edge** (DNS/anycast).
- Cache and serve content at the edge; on miss, fetch from **origin** (via a shield tier) and cache per `Cache-Control`/TTL.
- **Invalidate/purge** objects by URL, prefix or tag; **signed URLs/cookies** for private content; range requests; compression.
- Support **video streaming** (HLS/DASH segments), HTTPS, HTTP/2/3.
#### Below the line (out of scope)
- Edge compute/serverless, full WAF rule engine, billing/analytics pipeline details.

### Non-Functional Requirements
- **Low latency**: TTFB tens of ms; **high cache hit ratio** (>90-95% for static).
- **High availability**: survive PoP, link, and origin failures (serve stale).
- **Scalability**: Tbps aggregate, millions of req/s; **cost-efficient** bandwidth.
- **Consistency**: bounded staleness (TTL), fast purge (seconds).
#### Below the line (out of scope)
- Strong consistency of cached content across all edges.

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 50 Tbps peak ÷ 100 Gbps per edge server ≈ 500+ servers just for bandwidth at peak, replicated across ~100s of PoPs.
- 95% hit ratio → origin sees 5% = 2.5 Tbps (still big: add a **shield/mid-tier** to collapse misses).
- Working set per PoP: Zipf-like popularity: caching the top ~10-20% of objects often yields most hits; size SSD/RAM accordingly ([Caching](../basics/caching.md), [Back-of-envelope](../basics/back_of_envelope_estimation.md)).

## High-level API design
```
Control plane (customer):  POST /distributions {origins, behaviors, ttl, tls}   POST /invalidations {paths|tags}
                           PUT /keys (signing keys)                             GET /metrics /logs
Data plane (client):       GET https://cdn.example.com/path?sig=…&exp=…   (Range, If-None-Match)
```

## Data Model
- **Edge cache index**: `cacheKey (host+path+varied headers) → {object location (RAM/SSD), etag, expires, size, last_access}`.
- **Config store** (versioned, pushed to edges): distributions, origins, behaviors, TLS certs, signing keys.
- **Purge log**: `(purgeId, pattern/tag, timestamp)` distributed to edges.

## High Level System Design
```
Client ─DNS/anycast─▶ nearest PoP (edge cache) ──miss──▶ regional/shield cache ──miss──▶ Origin (S3/servers)
                          │ hit → serve                       └ request coalescing, stale-while-revalidate
Control plane: config & purge propagation · health/traffic steering · logs/metrics
```
- **Request routing**: **GeoDNS** (resolver-based, simple, coarse) and/or **anycast** (same IP announced from many PoPs; BGP picks the nearest; fast failover); traffic steering using health, load and latency measurements ([Load balancing](../basics/load_balancing.md)).
- **Edge server**: TLS termination, HTTP parsing, cache lookup (RAM → SSD), consistent hashing across servers inside a PoP so each object lives on one/few servers (better hit ratio) ([Consistent hashing](consistent_hashing_design.md)).
- **Tiered caching**: edge → regional/mid-tier ("origin shield") → origin; collapses duplicate misses and protects the origin.
- **Origin**: object store or web servers; CDN adds `Via`/auth headers; origin failover list.
- **Control plane**: pushes config; processes purges; collects logs/metrics ([Monitoring](../basics/monitoring.md)).

## Deep Dive
### Caching semantics
`Cache-Control: max-age/s-maxage`, `ETag/Last-Modified` + conditional revalidation (`304`), `Vary` (careful: explodes keys), `stale-while-revalidate`/`stale-if-error` (serve stale during origin trouble). **Cache key design**: normalize query strings, ignore tracking params, include only needed headers/cookies.

### Cache invalidation
- Prefer **versioned/immutable URLs** (`app.3f9c.js`, long TTL) over purges.
- **Purge propagation**: control plane fans purge messages to all PoPs (pub/sub + gossip); target seconds; soft purge = mark stale and revalidate; tag-based purge for groups.
- Fall back to short TTLs for mutable content.

### Cache stampede / thundering herd
**Request coalescing** (one origin fetch per object per PoP, others wait), shield tier, jittered TTLs, background refresh ([Hot keys](../basics/hot_keys.md)).

### Eviction and storage
LRU/LFU/TinyLFU with admission policies (don't cache one-hit wonders: Bloom-filter "second hit" admission) ([Bloom filter](../basics/bloom_filter.md)); RAM for hottest, SSD for the long tail; large video segments streamed with range support.

### Video streaming
Origin packages into HLS/DASH segments + manifests; CDN caches segments (immutable) and short-TTL manifests (live); prefetch next segments; multi-CDN for resilience ([YouTube](youtube.md)).

### Security
Signed URLs/cookies with expiry + IP/path constraints; TLS with SNI and automated certs; DDoS absorption via anycast capacity + rate limiting ([Backpressure](../basics/backpressure_load_shedding.md)); origin shielding (allow only CDN IPs), WAF; hotlink protection.

### Dynamic content acceleration
Persistent warm connections to origin, optimized routing, TCP/QUIC tuning, edge caching of API GETs with short TTL, request collapsing.

### Failure handling
PoP down → anycast/DNS withdraws it; edge → origin errors → serve stale; origin failover/secondary origin; health checks and circuit breakers ([Resilience](../basics/resilience_patterns.md)); graceful capacity overflow to neighboring PoPs.

### Metrics
Hit ratio (request/byte), origin offload, TTFB, error rates, purge latency, bandwidth cost.

### Pitfalls
- Caching personalized responses (cache poisoning/leaks); bad `Vary`; query-string explosion; no shield → origin overload on global purge; long TTL without versioning; ignoring the cost of cache misses at tail.

## References
* [Cloudflare: How CDN works](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/), [Akamai: Overlay networks (research)](https://dl.acm.org/doi/10.1145/2805789.2805800), [Netflix Open Connect](https://openconnect.netflix.com/)
* [Hello Interview: Design a CDN / Caching](https://www.hellointerview.com/learn/system-design/core-concepts/caching) · [Caching](../basics/caching.md)
