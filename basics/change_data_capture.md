# Change Data Capture (CDC)

> **TL;DR** — **CDC** turns a database's committed changes into an **ordered stream of events** (insert/update/delete with before/after images) so other systems (search index, cache, warehouse, vector index, other services) stay in sync **without dual writes**. The best implementation is **log-based CDC**: read the DB's transaction log (Postgres WAL via logical decoding, MySQL binlog, MongoDB oplog/change streams, DynamoDB Streams) with a tool like **Debezium**, publish to **Kafka**, and have idempotent consumers apply changes.
> It solves the dual-write problem, complements the **outbox pattern**, and enables event-driven architectures, CQRS read models and incremental ETL. Costs: **at-least-once delivery, schema evolution, snapshot/backfill complexity, lag, and WAL retention**. See [Event sourcing, CQRS & CDC](event_sourcing_cqrs_cdc.md), [Reliable messaging](reliable_messaging.md), [Kafka](kafka.md), [WAL/LSM](wal_lsm.md), [Idempotency](idempotency.md).

## Why CDC
- **Dual-write problem**: writing to DB *and* Kafka/Elasticsearch/cache from app code can't be atomic; one succeeds and the other fails → permanent divergence. CDC makes the DB the single source of truth and derives everything else from its log.
- Use cases: **search indexing** ([Elasticsearch](elasticsearch.md)), **cache invalidation/refresh** ([Caching](caching.md)), **vector index sync** ([Vector databases](vector_databases.md)), data warehouse/lakehouse ingestion, audit trails, microservice integration, materialized views, cross-region replication, migrations (zero-downtime moves).

## Approaches
| Approach | How | Pros | Cons |
|---|---|---|---|
| **Log-based** (Debezium, native logical replication) | Tail the WAL/binlog | Low overhead, captures all changes incl. deletes, ordered, no app changes | Needs DB config/permissions, log retention, DB-specific, schema-change handling |
| **Trigger-based** | DB triggers write to a change table | Works anywhere | Write amplification on the hot path, maintenance burden |
| **Query-based / polling** (`updated_at > ?`) | Periodic select | Simple | Misses deletes and intermediate states, load on DB, latency, clock issues |
| **Application-level outbox** | Write event row in the same transaction; relay publishes | Explicit event contract, transactional | Extra table and relay; often combined with log-based CDC to read the outbox |
| **Managed**: DynamoDB Streams, MongoDB change streams, AWS DMS, Datastream | Provider-captured | Operationally easy | Provider limits/retention |

## Log-based pipeline
```
App ─▶ DB (WAL/binlog) ─▶ CDC connector (Debezium/Kafka Connect) ─▶ Kafka topic per table ─▶ consumers
                                                                      ├▶ Elasticsearch / vector index
                                                                      ├▶ cache / read model
                                                                      └▶ warehouse / lake (Flink, Spark)
```
1. **Initial snapshot**: consistent read of existing rows, then switch to streaming from the log position (LSN/binlog offset) recorded at snapshot time. Incremental/online snapshots (watermark-based) avoid long locks.
2. **Streaming**: connector reads the log, emits events `{op: c|u|d|r, before, after, source{ts, lsn, txId}}`, commits its **offset**.
3. **Publish**: key = primary key so all changes for a row land in the **same Kafka partition** → per-row order ([Kafka](kafka.md), [Time ordering](time_ordering.md)).
4. **Consume**: idempotent upserts keyed by PK + version/LSN; deletes via tombstones.

## Guarantees and correctness
- **Ordering**: preserved per key (partition), not globally; cross-table/transaction order needs transaction markers or a single partition.
- **Delivery**: **at-least-once** (connector restarts re-emit) → consumers must be idempotent; use `version`/`lsn` to ignore stale or duplicate events ([Delivery semantics](delivery_semantics.md)).
- **Consistency**: downstream is **eventually consistent**; lag is a monitored SLI. Read-your-writes needs routing reads to the primary or version checks ([Consistency](consistency.md)).
- **Schema evolution**: use a schema registry (Avro/Protobuf/JSON schema) and backward-compatible changes; plan for column adds/drops/type changes.
- **Large/TOAST values, partial row images**: some DBs omit unchanged large columns; configure full images (e.g. Postgres `REPLICA IDENTITY FULL`) when you need before-images.

## Operational concerns
- **Replication slot / WAL retention** (Postgres): a stalled consumer makes the primary retain WAL until disk fills → alert on slot lag, set limits.
- **Failover**: log positions differ across replicas; plan for slot re-creation and re-snapshot, or use DB features that sync slots.
- **Backfill/replay**: re-snapshot a table or replay from Kafka retention to rebuild an index; keep topics compacted for latest-state rebuilds.
- **Throughput**: parallelize via table/partition; large transactions create bursty events.
- **Security/PII**: events contain full row data; mask/exclude columns, encrypt, ACL topics.
- **Monitoring**: end-to-end lag, connector status, DLQ, schema errors.

## CDC vs related ideas
- **vs Event sourcing**: ES stores events as the source of truth; CDC derives events from *state changes* of a conventional DB ([Event sourcing](event_sourcing_cqrs_cdc.md)).
- **vs Outbox**: outbox gives *domain events* with a stable contract; raw CDC exposes table-shape changes (leaky coupling). Often use CDC to ship the outbox table.
- **vs Replication**: DB replication copies state to replicas of the same engine; CDC feeds heterogeneous consumers.
- **vs Dual writes / 2PC**: avoids both anomaly and coordination cost ([Distributed transactions](distributed_transactions.md)).

## Pitfalls
- Dual writes "just this once"; consumers that aren't idempotent; ignoring deletes; unbounded replication-slot lag; exposing raw table schemas as a public contract; no plan for initial snapshot or schema change; assuming global ordering; polling with `updated_at` under clock skew or long transactions; forgetting PII in change events.

## Interview follow-ups
- *"How do you keep Elasticsearch in sync with Postgres?"* Log-based CDC → Kafka keyed by PK → idempotent indexer using version; periodic reconciliation.
- *"What about ordering and duplicates?"* Per-key order via partitioning; at-least-once delivery handled by idempotent upserts with LSN/version.
- *"CDC vs outbox?"* Outbox = explicit domain events written atomically; CDC = capture mechanism that can publish them.
- *"What happens if the consumer is down for a day?"* Offsets resume from Kafka; but watch retention and WAL slot growth upstream.

# References
* [Debezium documentation](https://debezium.io/documentation/reference/stable/)
* [Debezium: Outbox event router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
* [PostgreSQL logical decoding](https://www.postgresql.org/docs/current/logicaldecoding.html)
* [Designing Data-Intensive Applications, ch. 11 (stream processing & change data capture)](https://dataintensive.net/)
* [Hello Interview: Change Data Capture](https://www.hellointerview.com/learn/system-design/deep-dives/change-data-capture)
