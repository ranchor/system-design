# Scaling Writes

> **TL;DR** — Writes can't be cached away; they must land durably somewhere. Scale them by **(1) reducing work per write** (batching, fewer indexes, append-only), **(2) buffering bursts**
> through a **queue/log**, **(3) partitioning** so many nodes accept writes (sharding, leaderless/multi-leader), and **(4) choosing write-optimized storage** (LSM, [Cassandra](cassandra.md), [DynamoDB](dynamodb.md)).
> The recurring traps are **hot partitions**, **cross-shard transactions**, and **backpressure** when producers outrun consumers. See [Sharding](data_partitioning.md), [Kafka](kafka.md), [WAL & LSM](wal_lsm.md), [Hot keys](hot_keys.md).

## Techniques
| Technique | How | Trade-off |
|---|---|---|
| **Batching** | Group many writes into one round trip/transaction/fsync (bulk insert, Kafka producer batching, group commit) | Adds latency (linger), larger blast radius on failure |
| **Write buffering via queue** | Accept into Kafka/SQS, consumers apply at the DB's pace ([Async](asynchronism.md), [Kafka](kafka.md)) | Eventual visibility; need idempotent consumers ([Delivery semantics](delivery_semantics.md)) |
| **Append-only / log-structured** | Insert events, derive state later; LSM engines ([WAL & LSM](wal_lsm.md)) | Read amplification, compaction cost |
| **Sharding** | Partition by key so N nodes share writes ([Sharding](data_partitioning.md)) | Hot shards, cross-shard joins/transactions, resharding |
| **Leaderless / multi-leader** | Any replica accepts writes ([Replication](data_replication.md), [Quorum](quorum.md)) | Conflicts, eventual consistency |
| **Reduce indexes & constraints** | Each secondary index multiplies write cost | Slower reads |
| **Aggregate before write** | Counters buffered in memory/Redis and flushed periodically; sharded counters | Risk of losing buffered data; approximate until flush |
| **Load shedding** | Drop/low-priority writes under overload ([Backpressure](backpressure_load_shedding.md)) | Data loss by design |
| **Coarser durability** | Async fsync, `acks=1`, relaxed replication | Loss window on crash |

## Patterns
- **Write-behind**: app writes to cache/queue; background worker persists. Fast acks, but durability depends on the buffer ([Caching](caching.md)).
- **Sharded counter**: `likes:{post}:{0..N-1}`; increment a random shard, sum on read; fixes a single hot counter ([Distributed counter](../problems/distributed_counter.md)).
- **Time-bucketed keys** for time-series so writes spread (and old buckets expire) ([Cassandra](cassandra.md)).
- **Hash-prefix / salting** a monotonic key (timestamps, auto-increment) to avoid a single hot tail partition.
- **Per-entity single writer** (actor/partition ownership) removes lock contention at the cost of per-key throughput limits ([Contention](contention.md)).
- **Batch ingestion pipeline**: client → Kafka → stream job (aggregate) → OLAP/store ([Ad click aggregation](../problems/ad_click_aggregration.md)).

## Choosing storage
- Heavy write, simple access patterns → Cassandra/DynamoDB/ScyllaDB (LSM, partition key).
- Relational integrity needed → Postgres/MySQL with partitioning/Vitess/Citus; keep transactions single-shard ([Postgres & MySQL](postgres_mysql.md)).
- Bursty with spiky peaks → queue + autoscaled consumers.

## Pitfalls
- Monotonic partition keys (timestamps) → one hot partition; cross-shard transactions on the hot path; queue consumers with no backpressure; unbounded queue growth masking an under-provisioned DB; non-idempotent retries creating duplicates.

## Interview follow-ups
- *"Write QPS exceeds what one primary can do."* Shard by a high-cardinality key, buffer with a queue, batch, simplify indexes.
- *"A celebrity's counter is hot."* Sharded counters / local aggregation then flush ([Hot keys](hot_keys.md)).
- *"What happens if the queue backs up?"* Alert, scale consumers, shed/prioritize, backpressure to producers.

# References
* [Hello Interview: Scaling writes](https://www.hellointerview.com/learn/system-design/patterns/scaling-writes) · Designing Data-Intensive Applications, Chapters 3, 6
