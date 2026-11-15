# Apache Spark

> **TL;DR** — Spark is a **distributed data-processing engine**: you express transformations on a dataset (RDD/DataFrame/SQL), Spark builds a **DAG**, optimizes it (Catalyst), splits it into
> **stages** at **shuffle** boundaries, and runs **tasks** over partitions on executors, keeping data in **memory** where possible. Fault tolerance comes from **lineage** (recompute lost partitions).
> It covers batch ETL, SQL, ML, graph and **Structured Streaming** (micro-batch). Rule of thumb: the cost is dominated by **shuffles and skew**. For true low-latency event processing prefer
> [Flink](flink.md) ([comparison](flink_vs_spark.md)). Predecessor: [MapReduce](mapreduce.md); context: [Batch vs Stream](bp_sp.md).

## Architecture
- **Driver** (builds the plan, schedules), **cluster manager** (YARN, Kubernetes, standalone), **executors** (run tasks, cache data). Data lives in S3/HDFS/Delta/Iceberg/Parquet ([File formats](file_formats.md)).
- **Abstractions**: **RDD** (low-level, immutable, partitioned), **DataFrame/Dataset** (schema + Catalyst optimizer + Tungsten binary execution), **Spark SQL**.
- **Lazy evaluation**: *transformations* (`map`, `filter`, `join`, `groupBy`) build a plan; an *action* (`count`, `write`, `collect`) triggers execution.
- **Narrow vs wide dependencies**: narrow (`map`, `filter`) stay within a partition, pipelined in one stage; **wide** (`groupByKey`, `join`, `repartition`) require a **shuffle** → new stage.
- **Fault tolerance**: lineage lets Spark recompute only lost partitions; checkpoint long lineages; shuffle files persist so downstream stages can retry.

## Performance model (what interviewers probe)
| Concern | What to say |
|---|---|
| **Shuffle** | All-to-all network + disk; minimize with filter/project early, `reduceByKey`/aggregations (map-side combine) over `groupByKey`, broadcast joins for small tables |
| **Partitions** | Too few = under-parallelism/OOM, too many = scheduling overhead; ~100-200 MB per partition is a common target; `repartition` vs `coalesce` |
| **Skew** | One hot key → one huge task. Salting keys, **Adaptive Query Execution (AQE)** skew-join handling, broadcast, pre-aggregation |
| **Joins** | Broadcast hash join (small side), sort-merge join (large/large), shuffle hash; partition on the join key to avoid reshuffles |
| **Caching** | `cache()/persist()` for reused datasets (iterative ML); beware memory pressure and spill |
| **File layout** | Columnar formats (Parquet), partition pruning, avoid many small files, compaction |
| **Memory** | Executor memory split (execution vs storage), GC pressure, off-heap; spill to disk when needed |

## Structured Streaming
- Treats a stream as an **unbounded table**; the engine runs incrementally in **micro-batches** (sub-second to seconds; an experimental continuous mode exists) with **event-time windows**, **watermarks** for late data, state stores, and checkpointed offsets for **exactly-once** sinks (idempotent/transactional).
- Good for ETL/streaming analytics with seconds-level latency and one engine for batch and stream; weaker than Flink for ms-latency, complex event-time state and low-level control ([Flink vs Spark](flink_vs_spark.md)).

## Typical architectures
- **Lakehouse ETL**: raw (S3, Parquet/Delta/Iceberg) → Spark jobs → curated tables → BI/warehouse.
- **Lambda/Kappa**: batch + stream layers; Kafka → Spark Streaming/Flink → serving store ([Kafka](kafka.md)).
- **ML pipelines**: feature engineering at scale, MLlib, distributed training prep.
- Examples in this repo: [Ad Click Aggregation](../problems/ad_click_aggregration.md), [Top-K](../problems/top_k.md), [Web crawler](../problems/web_crawler.md).

## When to use
**Use**: large-scale batch ETL/analytics, SQL on data lakes, iterative ML, backfills/reprocessing, micro-batch streaming.
**Avoid**: low-latency per-event processing, tiny data (a single node or DuckDB/Postgres is simpler and cheaper), OLTP serving, workloads dominated by one huge shuffle you could avoid.

## Pitfalls
- `collect()` on big data (driver OOM); `groupByKey` on skewed keys; UDFs that bypass Catalyst; small-files problem; wrong partition count; unbounded state in streaming; non-idempotent writes with task retries.
- Over-cluster: autoscaling/spot churn causing shuffle fetch failures.

## Interview follow-ups
- *"Why is Spark faster than MapReduce?"* In-memory pipelining across stages, DAG optimization, no disk materialization between every step, columnar/Tungsten execution.
- *"A job is slow at one task only."* Data skew: inspect partition sizes, salt/AQE/broadcast.
- *"How does Spark recover from executor loss?"* Lineage recomputation; shuffle service/persisted shuffle files; speculative tasks.
- *"Batch vs streaming for this problem?"* Latency requirement and correctness: batch for backfill/accuracy, stream for freshness; unify with Kappa when possible ([Batch vs Stream](bp_sp.md)).

# References
* [Spark documentation](https://spark.apache.org/docs/latest/), [RDD paper (Zaharia et al., NSDI '12)](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf)
* [Hello Interview: Spark / batch](https://www.hellointerview.com/learn/system-design/deep-dives/spark) · Designing Data-Intensive Applications, Chapter 10
