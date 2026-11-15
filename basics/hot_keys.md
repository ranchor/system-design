# Hot Keys and Hot Partitions

> **TL;DR** — Partitioning spreads load only if keys are accessed evenly. A **hot key** (celebrity, viral post, flash-sale item, `GLOBAL` counter) or **hot partition** (monotonic key, popular tenant, geohash
> of a stadium) pins load on one node while others idle, capping throughput at *one node's capacity* no matter how many you add. Fix by **detecting**, then **splitting the key** (salting / sharded
> counters), **replicating or caching the hot item**, **absorbing with a buffer**, or **choosing a better partition key**. See [Sharding](data_partitioning.md), [Consistent hashing](consistent_hashing.md), [Caching](caching.md).

## Where they come from
| Source | Example |
|---|---|
| Skewed popularity | Celebrity profile, viral video, top-selling SKU |
| Monotonic keys | Timestamp / auto-increment as partition key → all writes hit the latest partition |
| Low-cardinality key | `country`, `status`, boolean |
| Tenant imbalance | One enterprise tenant = 80% of traffic |
| Spatial density | Geohash of a city center / stadium ([Geohash](geohash_quadtree.md)) |
| Global singleton | Single counter/row (`total_likes`), config row, lock |
| Thundering herd | Cache key expiry → everyone hits the DB for the same key |

## Detect
Per-partition metrics (CPU, throttling, p99), top-K keys via **Count-Min Sketch/heavy-hitter** sampling ([CMS](count_min_sketch.md)), DynamoDB Contributor Insights, Kafka partition lag skew, slow-log analysis.

## Mitigations
| Technique | Applies to | Idea | Cost |
|---|---|---|---|
| **Key salting / splitting** | Writes, hot rows | `key#rand(0..N)`; writes spread over N sub-keys, reads fan out and merge | Read complexity, N× reads |
| **Sharded counters** | Counters | Increment random shard, sum on read; or aggregate in memory and flush | Approximate/eventual |
| **Replicate hot data** | Reads | Cache copies in app memory/CDN/read replicas; client-side/local caching with short TTL | Staleness, invalidation |
| **Request coalescing / singleflight** | Cache stampede | One request fetches, others wait | Slight latency |
| **Jittered TTL, early refresh** | Expiry herds | Randomize expiry; refresh before expiry | More complexity |
| **Better partition key** | Skew by design | Composite keys (`tenant#shard`), hash instead of range, add entropy | Query patterns change |
| **Dedicated capacity** | Whale tenants | Isolate heavy tenants to their own shard/cluster | Operational overhead |
| **Buffer & batch** | Write bursts | Queue absorbs, consumers batch writes ([Scaling writes](scaling_writes.md)) | Eventual visibility |
| **Hybrid fan-out** | Celebrity feeds | Push for normal users, pull for celebrities ([Fan-out](fanout_write_vs_read.md)) | Two code paths |
| **Adaptive/auto-split** | Managed stores | DynamoDB adaptive capacity & split-for-heat; range re-splitting in Bigtable/Spanner/Cockroach | Not for a single hot *key* beyond one partition's limit |
| **Load shedding / rate limit** | Abuse | Throttle hot callers ([Rate limiter](../problems/rate_limiter.md)) | Dropped requests |

## Examples
- **Like counter on a viral post**: write to `post:{id}:shard:{rand(16)}` (or Redis `INCR` + periodic flush); read = sum of shards, cached for 1-5 s.
- **Time-series ingest**: key = `(device_id, hour_bucket)` not `timestamp`.
- **Flash sale inventory**: split stock into N buckets, route a buyer to a bucket, rebalance when a bucket empties ([Contention](contention.md), [Ticketmaster](../problems/ticketmaster.md)).
- **Celebrity profile reads**: serve from CDN/local cache with tiny TTL, replicate the key across cache nodes.

## Pitfalls
- Adding nodes to fix a single-key hotspot (doesn't help); salting without a reader that merges; caching with synchronized TTLs; assuming hash partitioning prevents skew (it doesn't for one key).

## Interview follow-ups
- *"Consistent hashing is balanced, so why is one node hot?"* Hashing balances keys, not **traffic per key**; one hot key still maps to one node. Use replication/salting.
- *"How do you split a hot write key but keep reads simple?"* Sharded writes with cached aggregated read, or approximate counters.
- *"How do you detect it?"* Per-partition metrics + top-K sampling.

# References
* [Hello Interview: Scaling writes (hot keys)](https://www.hellointerview.com/learn/system-design/patterns/scaling-writes) · [AWS: Choosing the right DynamoDB partition key](https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/)
