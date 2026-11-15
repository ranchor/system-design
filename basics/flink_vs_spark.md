# Flink vs Spark (stream processing engines)

> **TL;DR** — **Flink** is **streaming-first**: records flow continuously through operators with **event-time** semantics, rich **keyed state** and **exactly-once** checkpoints, giving
> millisecond-to-second latency; batch is a special case. **Spark** is **batch-first**: Structured Streaming runs **micro-batches** on the same engine, giving second-level latency with one
> unified API/SQL/ML stack. Pick **Flink** for low-latency, stateful, event-time-heavy pipelines (fraud, real-time aggregation, CEP); pick **Spark** when batch dominates, you want one engine
> for ETL + ML + streaming, or second-level latency is fine. Details: [Flink](flink.md), [Spark](spark.md), [Batch vs Stream](bp_sp.md).

| Dimension | **Apache Flink** | **Apache Spark (Structured Streaming)** |
|---|---|---|
| Processing model | True streaming: record-at-a-time, pipelined operators | Micro-batch (default); experimental continuous mode |
| Typical latency | Milliseconds to low seconds | ~Sub-second to seconds (batch interval bound) |
| Event time & late data | First-class: watermarks, allowed lateness, side outputs | Supported: watermarks, but fewer fine-grained controls |
| State | Large keyed state (RocksDB), queryable, state TTL, savepoints | State store (HDFS-/RocksDB-backed), simpler model |
| Fault tolerance | Asynchronous barrier snapshots (Chandy-Lamport style) → exactly-once state | Checkpointed offsets + WAL, replay micro-batch |
| Exactly-once sinks | Two-phase-commit sinks (Kafka, files) | Idempotent / transactional sinks |
| Windowing | Rich (tumbling, sliding, session, custom triggers) | Tumbling, sliding, session |
| Batch performance | Good, but ecosystem less deep | Excellent; mature SQL optimizer (Catalyst/AQE), huge ecosystem |
| ML / graph / SQL ecosystem | Smaller (Flink SQL, CEP, Table API) | Larger (MLlib, GraphX, Delta/Iceberg tooling, notebooks) |
| Back pressure | Native credit-based flow control | Handled by batch sizing/rate limits |
| Ops complexity | State management and tuning needs expertise | Familiar to data teams; heavier cluster autoscaling needs |
| Typical users | Uber, Alibaba, Netflix, fraud/risk, real-time analytics | Databricks users, lakehouse ETL, analytics, ML teams |

## How each handles the same problem (5-minute tumbling count per user)
- **Flink**: `keyBy(user).window(Tumbling 5m).aggregate(count)`; state per key and window; watermark closes windows; result emitted as soon as the window closes; late events routed to a side output.
- **Spark**: `groupBy(window(eventTime, "5 minutes"), user).count()` with a watermark; processed per micro-batch; emits per output mode (append/update/complete).

## Choosing in an interview
1. **Latency requirement < 1 s with complex state?** → Flink.
2. **Primarily batch/backfill/SQL with some near-real-time?** → Spark (one codebase for batch + stream).
3. **Source is Kafka and transformations are simple (filter/enrich/aggregate)?** → Kafka Streams or ksqlDB may be enough ([Kafka](kafka.md)).
4. **Fully managed on AWS?** → Kinesis Data Analytics (managed Flink), EMR/Glue (Spark), Databricks.
5. **Need exactly-once into a warehouse?** → either with transactional/idempotent sinks; say how.

## Shared concepts to name
- **Event time vs processing time**, **watermarks**, windows, late data policy ([Time & ordering](time_ordering.md)).
- **Checkpointing and replay** for recovery; **idempotent sinks** for end-to-end effects ([Delivery semantics](delivery_semantics.md)).
- **Partitioning/keying** for parallelism and state locality; skew mitigation ([Hot keys](hot_keys.md)).
- **Lambda vs Kappa**: separate batch + speed layers vs one streaming pipeline that can replay the log.

## Pitfalls
- Choosing streaming where a batch job every 5 minutes would do; unbounded state growth; ignoring late data; using processing time for correctness; skew on a hot key; no savepoint/upgrade plan.

# References
* [Flink docs](https://nightlies.apache.org/flink/flink-docs-stable/), [Spark Structured Streaming guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
* [Hello Interview: Flink](https://www.hellointerview.com/learn/system-design/deep-dives/flink)
