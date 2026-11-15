# Scaling Reads

> **TL;DR** — Most systems are read-heavy (10:1 to 1000:1). Scale reads in this order, cheapest first: **(1) fix the query/index**, **(2) cache** (client, CDN, app, DB), **(3) read replicas**,
> **(4) denormalize / precompute** (materialized views, CQRS read models), **(5) shard**. Each step adds a consistency cost: stale data. State the staleness you accept. See [Caching](caching.md),
> [Replication](data_replication.md), [CDN](cdn.md), [Database Indexes](database_indexing.md), [Sharding](data_partitioning.md).

## Ladder of options
| Step | What | Gain | Cost / risk |
|---|---|---|---|
| 1. Index & query tuning | Right indexes, covering indexes, no N+1, pagination, `EXPLAIN` | Often 10-100× | Write overhead of indexes ([Indexing](database_indexing.md)) |
| 2. Caching | Browser → CDN → API gateway → Redis/Memcached → DB buffer pool | Removes load entirely for hot data | Invalidation, stampede, staleness ([Caching](caching.md)) |
| 3. Read replicas | Async replicas serve reads; primary takes writes | Linear read scaling | **Replication lag**: stale/reordered reads, read-your-writes breakage ([Replication](data_replication.md)) |
| 4. Precompute | Materialized views, counters, denormalized tables, search index, CQRS read store | Turns joins into key lookups | Write amplification, eventual consistency ([CQRS](event_sourcing_cqrs_cdc.md)) |
| 5. Shard | Spread data so no node holds it all | Scales data size too | Cross-shard queries, resharding ([Sharding](data_partitioning.md)) |
| 6. Move static/large content | Object storage + CDN | Offloads bandwidth | Cache purge, signed URLs ([Blob storage](blob_storage.md)) |

## Diagram
```
Client ─▶ CDN ─▶ LB ─▶ App ─▶ Redis ─▶ Read replica(s) ─▶ (Primary only for writes / strong reads)
 cache     cache         cache   cache   async replication
```

## Design notes
- **Cache hierarchy**: put the cache closest to the user that your staleness budget allows; use TTL + explicit invalidation on writes; protect against **stampedes** (request coalescing, jittered TTLs, locking) and **hot keys** ([Hot keys](hot_keys.md)).
- **Replica routing**: route reads to replicas, but send **read-your-writes** traffic (just-written user, critical flows) to the primary, or use a session token with the last LSN/version, or sticky reads for N seconds.
- **Monotonic reads**: pin a session to one replica so time doesn't go backwards ([Consistency](consistency.md)).
- **Pagination**: keyset/cursor pagination rather than `OFFSET` for deep pages.
- **Precomputation**: counts, feeds, leaderboards, aggregates computed at write time or by a stream job; read path is a key lookup ([Fan-out](fanout_write_vs_read.md)).
- **Search/analytics reads** go to a search index or OLAP store, not the OLTP primary ([Elasticsearch](elasticsearch.md)).
- **Back-of-envelope first**: QPS, hit ratio and working-set size decide whether one Redis node suffices ([Estimation](back_of_envelope_estimation.md)).

## Pitfalls
- Caching without an invalidation story; replicas for a write-bound problem; replica lag causing "my post disappeared"; cache cold-start after deploy overwhelming the DB; unbounded cache growth/hot-key meltdown.

## Interview follow-ups
- *"Reads are 100× writes; what do you do?"* Indexes → cache with CDN/Redis → replicas → precompute; quantify hit rate.
- *"How do you keep replicas consistent for the author?"* Read-your-writes via primary reads or version tokens.
- *"What if one key is read millions of times/s?"* Local in-process cache, replicate the hot key, CDN ([Hot keys](hot_keys.md)).

# References
* [Hello Interview: Scaling reads](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads) · Designing Data-Intensive Applications, Chapters 5 & 6
