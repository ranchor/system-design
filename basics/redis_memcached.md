# Redis (and Redis vs Memcached)

> **TL;DR** — **Redis** is an in-memory data-structure server: strings, hashes, lists, sets, **sorted sets**, streams, bitmaps, HyperLogLog, geospatial indexes, pub/sub, Lua scripts. Single-threaded command execution (I/O threads aside) gives **atomic** per-command semantics and sub-millisecond latency.
> Use it for **caching, sessions, rate limiting, leaderboards, queues/streams, distributed locks, counters, geo lookups**. It is **not** a durable system of record by default: persistence (RDB snapshots, AOF) and async replication can lose recent writes.
> **Memcached** is a simpler multi-threaded string cache with no persistence or rich types. See [Caching](caching.md), [Hot keys](hot_keys.md), [Distributed locking](distributed_locking.md), [Consistent hashing](consistent_hashing.md).

## Data structures → use cases
| Structure | Typical use |
|---|---|
| **String** (+ `INCR`, `SETNX`/`SET NX EX`) | Cache values, counters, simple locks, idempotency keys |
| **Hash** | Objects/sessions; partial field updates |
| **List** | Simple queues, recent-items feeds (`LPUSH` + `LTRIM`) |
| **Set** | Tags, unique visitors, set algebra (intersections) |
| **Sorted set (ZSET)** | Leaderboards, delay queues (score = timestamp), sliding-window rate limiting, top-K |
| **Stream** | Append-only log with **consumer groups** (acks, pending list); lighter than Kafka ([Kafka](kafka.md)) |
| **Pub/Sub** | Fire-and-forget fan-out (no persistence; missed if subscriber is down) |
| **Bitmap / HyperLogLog** | Daily-active flags; approximate distinct counts ([HyperLogLog](hyperloglog.md)) |
| **Geo** (GEOADD/GEOSEARCH) | Nearby lookups on a sorted-set geohash ([Geohash](geohash_quadtree.md)) |

## Architecture and scaling
- **Single node**: one main thread executes commands, so keep commands O(1)/O(log n); avoid `KEYS *`, huge `SMEMBERS`, large Lua scripts (they block everyone).
- **Replication**: primary → replicas, **asynchronous**; `WAIT` gives limited sync. Failover can lose acknowledged writes ([Replication](data_replication.md)).
- **Sentinel**: monitors and promotes a replica (HA without sharding). **Redis Cluster**: **16,384 hash slots** sharded across primaries, each with replicas; clients route by `CRC16(key) mod 16384`; multi-key ops need keys in the same slot (**hash tags** `{user42}:cart`) ([Partitioning](data_partitioning.md)).
- **Persistence**: **RDB** (periodic snapshot, compact, can lose minutes) vs **AOF** (append log with `fsync` policy `always/everysec/no`, bigger, replay on restart); often both. Persistence is for recovery, not a database guarantee.
- **Eviction** when `maxmemory` is hit: `allkeys-lru/lfu`, `volatile-*`, `noeviction`; pick by workload. TTLs expire lazily + by sampling.
- **Memory**: everything in RAM; watch fragmentation, large keys, and replica output buffers.

## Common patterns
- **Cache-aside** with TTL + jitter; protect from stampedes with request coalescing/locks or probabilistic early refresh ([Caching](caching.md)).
- **Rate limiting**: fixed window (`INCR` + `EXPIRE`), sliding window (ZSET), token bucket (Lua script for atomicity) ([Rate limiter](../problems/rate_limiter.md)).
- **Leaderboard**: `ZADD`, `ZREVRANGE`, `ZRANK` with O(log n); shard by key when huge.
- **Distributed lock**: `SET key token NX PX ttl` + release only if token matches (Lua). Single-instance locks are *not* safe under failover/pauses; use **fencing tokens** or a consensus system for correctness-critical locks ([Distributed locking](distributed_locking.md)).
- **Job queue / delay queue**: lists/streams or ZSET by due time; at-least-once, so make jobs idempotent ([Long-running tasks](long_running_tasks.md)).
- **Pub/Sub for real-time**: fan out to WebSocket gateways; no replay, so combine with streams if you need catch-up ([Real-time updates](realtime_updates.md)).
- **Session store, feature flags, idempotency keys, dedupe sets** ([Counting & dedup](counting_dedup_at_scale.md)).

## Redis vs Memcached
| | Redis | Memcached |
|---|---|---|
| Data types | Rich structures | Strings/blobs |
| Threading | Single-threaded commands (+I/O threads) | Multi-threaded |
| Persistence / replication | Yes (RDB/AOF, replicas, cluster) | No (cache only) |
| Atomic ops, Lua, transactions | Yes | Basic (incr, cas) |
| Scaling | Cluster with slots | Client-side consistent hashing |
| Best for | Cache + data-structure workloads | Pure, simple, high-throughput cache |

## Failure modes
- **Hot key**: one key saturates one shard → local caches, key splitting, read replicas ([Hot keys](hot_keys.md)).
- **Big key**: deleting/reading blocks the thread → `UNLINK`, chunk large values.
- **Thundering herd** on expiry or cold start; **cache penetration** (non-existent keys) → negative caching/bloom filter ([Bloom filter](bloom_filter.md)).
- **Failover data loss** and split brain on partitions; **memory exhaustion** → eviction or OOM; **replica resync storms**.

## Pitfalls
- Treating Redis as the only copy of important data; locks without TTL or fencing; unbounded growth (no TTL); using pub/sub for reliable messaging; cross-slot multi-key commands in Cluster; blocking commands in the hot path.

## Interview follow-ups
- *"Why is Redis fast?"* In-memory, efficient data structures, single-threaded execution without lock contention, pipelining, I/O multiplexing.
- *"How do you scale beyond one node?"* Cluster slots + replicas; read replicas for reads; client-side caching for hot keys.
- *"Is a Redis lock safe?"* Only as a best-effort lease; use fencing tokens or ZooKeeper/etcd when correctness matters.
- *"Redis or Kafka for a queue?"* Redis streams for light, low-latency work; Kafka for durable, replayable, high-throughput logs.

# References
* [Redis documentation](https://redis.io/docs/latest/)
* [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
* [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
* [Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
* [Hello Interview: Redis deep dive](https://www.hellointerview.com/learn/system-design/deep-dives/redis)
* [ByteByteGo: A crash course in Redis](https://blog.bytebytego.com/p/a-crash-course-in-redis) · [Redis vs Memcached](https://blog.bytebytego.com/p/redis-vs-memcached)
* [Video: Redis crash course](https://www.youtube.com/watch?v=a4yX7RUgTxI)
