# Kafka vs SQS (log vs queue)

> **TL;DR** — **SQS** is a managed **queue**: messages are for *one* worker, deleted on ack, no replay, zero ops. **Kafka** is a **log**: records are retained, partitioned, ordered per
> partition, readable by many consumer groups and **replayable**. Choose **SQS** for simple task distribution on AWS; choose **Kafka** for high-throughput event streams,
> multiple independent consumers, ordering by key, replay and stream processing. In an interview, name the property you need (replay? fan-out? ordering? ops burden?) and pick accordingly.
> Details: [Kafka](kafka.md), [SQS](sqs.md), [Asynchronism](asynchronism.md).

| Dimension | **Kafka** (distributed log) | **SQS** (managed queue) |
|---|---|---|
| Model | Append-only partitioned log; consumers track **offsets** | Queue of messages; consumers **receive then delete** |
| Consumers | Many **consumer groups** each read all data independently | One logical consumer pool; fan-out needs SNS → multiple queues |
| Replay / reprocess | Yes (retention window, reset offsets) | No (once deleted, gone; DLQ redrive only) |
| Ordering | **Per partition** (key-based) | Standard: best effort; FIFO: per message group |
| Throughput | Very high (MB/s per partition, millions msg/s cluster) | Standard: ~unlimited; FIFO: limited per queue (check quotas) |
| Delivery | At-least-once; exactly-once within Kafka via idempotent producer + transactions | At-least-once (Standard); FIFO dedup window for producer retries |
| Retries / DLQ | Application-implemented (retry topics, DLT) | **Built in**: visibility timeout, maxReceiveCount, DLQ, redrive |
| Per-message features | None (offset-based) | Delay, visibility timeout, long polling; no priorities |
| Parallelism | ≤ partitions per group | Any number of consumers |
| Latency | Low ms; batching trade-offs | Tens of ms; polling-based |
| Retention | Days to forever (compaction, tiered storage) | Up to 14 days |
| Ops | Cluster to run (or managed MSK/Confluent) | Fully serverless |
| Message size | ~1 MB default | 256 KiB historically (check current docs) |
| Cost model | Cluster/storage/throughput | Per request + data |
| Best for | Event streaming, CDC, event sourcing, analytics pipelines, ordered per-entity processing, replay | Background jobs, work queues, buffering, simple decoupling, serverless (Lambda) triggers |

## Decision guide
1. **Need to replay or have several independent consumers of the same data?** → Kafka (or Kinesis).
2. **Need per-entity ordering at high throughput?** → Kafka partitions by key (SQS FIFO if throughput modest).
3. **Just need "run this job once-ish with retries and a DLQ, no ops"?** → SQS.
4. **Already on AWS and want Lambda consumers?** → SQS (or Kinesis/MSK triggers).
5. **Stream processing / joins / windowing?** → Kafka + Flink/Kafka Streams ([Flink](flink.md)).
6. **Priority queues, complex routing, TTL per message?** → RabbitMQ (or multiple SQS queues).

## Worked examples
- **Order events consumed by billing, inventory, analytics**: Kafka topic `orders` with three groups; analytics can replay last 7 days.
- **Resize uploaded images**: S3 event → SQS → workers; failures to DLQ; scale on queue depth.
- **Per-account ledger updates in order**: Kafka key = accountId, or SQS FIFO with MessageGroupId = accountId.
- **Spiky email sends**: SQS buffers; consumers rate-limited to the provider's quota.

## Common interview answer
> "I'd use SQS for the job queue because each task needs to be processed once by one worker, with retries and a DLQ and no cluster to operate. For the event stream that
> several services and the analytics pipeline consume, and which we may need to replay, I'd use Kafka partitioned by `userId` to keep per-user ordering."

## Pitfalls to mention
- Kafka for a trivial job queue = operational overhead; SQS for replayable streaming = you'll miss replay and independent consumers.
- Both are at-least-once in practice → idempotent consumers ([Idempotency](idempotency.md), [Delivery semantics](delivery_semantics.md)).
- Both need the outbox pattern for reliable DB + publish ([Reliable messaging](reliable_messaging.md)).

# References
* [Kafka](kafka.md), [SQS](sqs.md), [Distributed Message Queue (problem)](../problems/distributed_message_queue.md)
* [Hello Interview: Kafka](https://www.hellointerview.com/learn/system-design/deep-dives/kafka)
