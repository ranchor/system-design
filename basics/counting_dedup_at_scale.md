# Counting & Deduplication at Scale

> **TL;DR** — Exact counting and dedup need memory proportional to the data. At scale use **approximate structures** with controlled error: **Bloom filter** ("have I seen X?": no false negatives, tunable false
> positives), **Count-Min Sketch** ("how often did X occur?" and heavy hitters: overestimates only), **HyperLogLog** ("how many distinct?": ~1% error in KBs). For exact needs use **sharded counters**,
> **batch/stream aggregation**, or **idempotency keys + unique constraints**. Choose by the question, error tolerance, and whether you need merge/delete. See [Bloom filter](bloom_filter.md), [Count-Min Sketch](count_min_sketch.md), [HyperLogLog](hyperloglog.md), [Hot keys](hot_keys.md).

## Pick the structure by question
| Question | Structure | Error | Memory | Notes |
|---|---|---|---|---|
| Seen before? (dedup, cache-penetration guard, "username taken?", crawler URLs) | [**Bloom filter**](bloom_filter.md) | False positives only | ~10 bits/item for 1% FP | No deletes (counting/cuckoo filters add them) |
| How many times? top-K / heavy hitters (trending, rate limit by key) | [**Count-Min Sketch**](count_min_sketch.md) + heap | Overestimates | Fixed `w×d` counters | Mergeable; error ∝ total/width |
| How many distinct? (unique visitors) | [**HyperLogLog**](hyperloglog.md) | ~1.04/√m | ~12 KB | Mergeable; no intersection |
| Exact count, high write rate | **Sharded counter** / stream aggregation | Exact (eventually) | O(keys × shards) | Sum on read ([Hot keys](hot_keys.md)) |
| Exact dedup of events | **Unique key / idempotency table** | None | O(ids in window) | Retention window ([Idempotency](idempotency.md)) |
| Quantiles (p99) | t-digest / DDSketch / HDR | Small relative | KBs | For latency metrics |
| Membership with deletes | Counting Bloom / cuckoo filter | FP | More bits | |

## Counting patterns
1. **Write-aggregated counters**: buffer increments in memory/Redis (`INCRBY`), flush periodically to DB; tolerate losing a few seconds on crash or use a log (Kafka) as the durable source ([Scaling writes](scaling_writes.md)).
2. **Sharded counters** for single hot items: N sub-counters, random increment, sum on read, cache result ([Distributed counter](../problems/distributed_counter.md)).
3. **Stream aggregation** (Flink/Spark): tumbling/sliding windows by key; late data handling; sink to OLAP/KV ([Flink](flink.md), [Ad click aggregation](../problems/ad_click_aggregration.md)).
4. **Lambda/Kappa**: approximate real-time numbers + exact batch reconciliation ([Spark](spark.md)).
5. **Top-K**: CMS or Space-Saving for candidates, exact counts for the shortlist, per-window merge ([Top-K](../problems/top_k.md)).
6. **Rate limiting counters**: per-key windows in Redis; CMS for very high-cardinality keys ([Rate limiter](../problems/rate_limiter.md)).

## Dedup patterns
- **Request/event dedup**: idempotency key + unique index; keep ids for the retry window ([Delivery semantics](delivery_semantics.md)).
- **Pre-filter with Bloom, confirm with DB**: if Bloom says "no" → definitely new (skip DB); if "maybe" → check exact store. Saves lookups for the common case ([Web crawler](../problems/web_crawler.md)).
- **Content dedup**: hash chunks/files (SHA-256, content-defined chunking); store once, reference count ([Large blobs](large_blobs.md)).
- **Near-duplicate**: SimHash/MinHash + LSH for similar documents/images.
- **Stream dedup**: windowed state keyed by id (Flink keyed state with TTL).

## Trade-offs to state
- Accuracy vs memory; mergeability across shards/time; supports deletes?; consistency of the number across views (dashboards disagreeing slightly); **never** use approximations for money or compliance.

## Pitfalls
- Using HLL/CMS where exactness is contractual; Bloom filter sized for too few items (FP rate explodes); forgetting CMS only overestimates; counters lost on crash without a log; hot single-row counters; dedup windows shorter than the retry window.

## Interview follow-ups
- *"Count unique users per day across 1B events."* HLL per day/page; merge for ranges.
- *"Trending hashtags?"* Sliding-window CMS + top-K heap, or stream aggregation by key.
- *"Don't hit the DB for IDs that don't exist."* Bloom filter in front (cache penetration).
- *"Exactly count likes under a viral spike."* Sharded counters + async aggregation, exact total reconciled from the log.

# References
* [Hello Interview: Counting at scale / probabilistic data structures](https://www.hellointerview.com/learn/system-design/deep-dives) · Designing Data-Intensive Applications, Chapter 11 · [Redis probabilistic structures](https://redis.io/docs/latest/develop/data-types/probabilistic/)
