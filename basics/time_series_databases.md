# Time Series Databases

> **TL;DR** — Time-series data (metrics, IoT, logs-as-metrics, financial ticks) is **append-mostly, timestamp-ordered, written in huge volume, queried by time range and aggregated**, and old data matters less. **TSDBs** (InfluxDB, TimescaleDB, Prometheus/VictoriaMetrics/M3, ClickHouse, Druid, Apache Pinot, QuestDB) optimize for this with **time-partitioned storage, columnar compression, downsampling/rollups, retention policies and tag/label indexes**.
> Main risk: **high cardinality** (too many unique tag combinations). See [Metrics aggregation](../problems/metrics_aggregration.md), [Distributed metrics](../problems/distributed_metrics.md), [WAL/LSM](wal_lsm.md), [File formats](file_formats.md), [Scaling writes](scaling_writes.md), [HyperLogLog](hyperloglog.md).

## Data model
```
measurement/metric: cpu_usage
tags (indexed, low-ish cardinality): host=web-17, region=us-east, env=prod
fields/value (not indexed):         usage=0.73
timestamp:                          2026-11-15T16:23:05Z
```
A **series** = unique metric + tag set. Each series is a sequence of `(timestamp, value)` points. Cardinality = number of distinct series.

## Why general-purpose DBs struggle
- Write rate (millions of points/s) with tiny rows → index and WAL overhead.
- Deletes are by **time range** (retention), not row; B-tree deletes/vacuum are expensive.
- Queries scan time ranges and aggregate (avg, p99, rate, group by tag, `time_bucket`), favoring **columnar** layouts and compression.

## Storage techniques
| Technique | What it does |
|---|---|
| **Time-based partitioning** (chunks/blocks, e.g. 2h or 1d) | Prune by time; drop an entire partition for retention (cheap) |
| **Append-only / LSM-style ingest** | Buffer in memory + WAL, flush sorted immutable blocks, compact ([WAL/LSM](wal_lsm.md)) |
| **Columnar + compression** | Timestamps via **delta-of-delta**, floats via **XOR (Gorilla)** encoding; often 10x+ compression |
| **Series index (inverted index tag → series IDs)** | Fast label filtering; its size drives memory use |
| **Rollups / downsampling / continuous aggregates** | Store 1s data for 7 days, 1m for 30 days, 1h for a year |
| **Retention & tiering** | TTL, move cold blocks to object storage ([Blob storage](blob_storage.md)) |
| **Out-of-order handling** | Late data windows, backfill paths |

## Architecture patterns
- **Pull (Prometheus)**: server scrapes targets; simple, service-discovery-driven, single-node local storage; scale via federation, **remote write** to long-term stores (Thanos, Cortex/Mimir, VictoriaMetrics).
- **Push (agents, StatsD, OpenTelemetry)**: clients push to a gateway/queue; buffer through [Kafka](kafka.md) to absorb spikes and decouple ([Backpressure](backpressure_load_shedding.md)).
- **Ingestion pipeline**: agent → Kafka → stream aggregation ([Flink](flink.md)) → TSDB; pre-aggregate high-volume streams before storing.
- **Sharding**: by series hash and/or time range; **replication** for availability; queries fan out and merge ([Partitioning](data_partitioning.md)).
- **Real-time analytics engines** (ClickHouse, Druid, Pinot) handle event-level data with time-based partitions and fast aggregations beyond classic metrics.
- **TimescaleDB**: PostgreSQL extension with hypertables (auto time/space partitioning), compression and continuous aggregates; good when you need SQL joins with relational data ([PostgreSQL](postgres_mysql.md)).

## Cardinality: the #1 operational problem
Putting unbounded values in tags (user_id, request_id, URL with IDs, container IDs that churn) multiplies series: `series = Π(distinct values per tag)`. Effects: index memory blow-up, slow queries, ingest failures. Mitigate: **never use unbounded IDs as tags**, put them in fields/logs/traces instead; drop/relabel labels at ingest; per-tenant series limits; use approximate distinct counts ([HyperLogLog](hyperloglog.md)); aggregate before storing.

## Query patterns
`WHERE time BETWEEN … AND tag=…`, `GROUP BY time_bucket(1m), tag`, rate/derivative of counters, percentile (use histograms/sketches, not averages of percentiles), top-k by series, gap-filling and interpolation, joins with metadata, alerting rules evaluated continuously.

## Design checklist
1. Write rate (points/s), series count, retention per resolution, query latency targets.
2. Choose pull vs push and a buffer (Kafka) for bursts.
3. Define tags vs fields; enforce cardinality limits.
4. Pick rollups and retention tiers; cold storage for history.
5. Shard/replicate for scale and HA; plan for out-of-order and late data.
6. Alerting path separate from heavy dashboard queries.

## Pitfalls
- High-cardinality tags; querying raw data over long ranges instead of rollups; averaging percentiles; no retention policy (disk fills); single-node Prometheus as long-term storage; clock skew between hosts; unbounded label churn in Kubernetes; coupling ingestion directly to the DB with no buffer.

## Interview follow-ups
- *"Why not Postgres or Cassandra?"* They can work at modest scale (Timescale = Postgres), but TSDB layouts give better compression, time pruning and downsampling.
- *"How do you store a year of metrics?"* Rollups per tier, columnar compression, cold object storage.
- *"What breaks at high cardinality?"* Series index memory and query planning; fix by removing unbounded tags.
- *"How do you compute p99 across hosts?"* Mergeable histograms/sketches, not an average of per-host p99s.

# References
* [Gorilla: A Fast, Scalable, In-Memory Time Series Database (VLDB 2015)](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
* [Prometheus storage](https://prometheus.io/docs/prometheus/latest/storage/)
* [TimescaleDB documentation](https://docs.timescale.com/)
* [InfluxDB documentation](https://docs.influxdata.com/)
* [Hello Interview: Time series databases](https://www.hellointerview.com/learn/system-design/deep-dives/time-series-databases)
