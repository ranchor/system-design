# Amazon SQS (Managed Queue)

> **TL;DR** — SQS is AWS's fully managed **message queue**: producers `SendMessage`, consumers **poll** (`ReceiveMessage`), process, then `DeleteMessage`. If they don't delete in time the
> message becomes visible again (**visibility timeout**) and is redelivered, so SQS is **at-least-once** and consumers must be **idempotent**. **Standard** queues give nearly unlimited
> throughput, best-effort ordering; **FIFO** queues give ordering per **message group** and deduplication, at lower throughput. It has built-in **DLQs**, delays, long polling and
> autoscaling-friendly metrics. Use it for work distribution and decoupling; use [Kafka](kafka.md) for replayable streams ([comparison](kafka_vs_sqs.md)). See also [Asynchronism](asynchronism.md).

## Core model
- **Queue** holds messages until a consumer deletes them (retention default 4 days, up to 14). Each message is delivered to **one** consumer (competing consumers); no replay after deletion.
- **Visibility timeout** (default 30 s, max 12 h): after receive, the message is hidden. Set it above worst-case processing time (or extend it while working). Delete on success.
- **Long polling** (`WaitTimeSeconds` up to 20 s) reduces empty receives and cost.
- **Dead-letter queue (DLQ)**: after `maxReceiveCount` failed receives, the message moves to a DLQ for inspection and **redrive**.
- **Delay queues / message timers**: delay up to 15 min. **Batching**: up to 10 messages per call (cost and throughput).
- **Message size**: historically 256 KiB (check current docs, it was raised in 2025); use the **extended client** or pointers to S3 for large payloads ([Blob Storage](blob_storage.md)).
- **Security/ops**: IAM policies, SSE encryption, VPC endpoints; CloudWatch metrics (`ApproximateNumberOfMessagesVisible`, `ApproximateAgeOfOldestMessage`).

## Standard vs FIFO
| | Standard | FIFO |
|---|---|---|
| Ordering | Best effort | Strict **per MessageGroupId** |
| Delivery | At-least-once (duplicates possible) | **Exactly-once processing semantics** via deduplication (5-minute window by `MessageDeduplicationId`/content hash) |
| Throughput | Nearly unlimited | Limited per queue (hundreds of msgs/s by default, thousands with batching; "high throughput" mode raises it; check quotas) |
| Parallelism | Any number of consumers | Parallel across groups, serial within a group (one in-flight batch per group) |
| Use | Most background jobs | Per-entity ordering (account ledger, per-order state), dedup of producer retries |

- FIFO caveat: a **poison message** blocks its group until it goes to the DLQ; choose group IDs that match the ordering you actually need (not one global group).

## Patterns
- **Work queue + autoscaling**: scale workers on queue depth / message age ([Resilience](resilience_patterns.md)).
- **Fan-out**: SNS topic → many SQS queues (each service gets its copy), or EventBridge rules.
- **Load leveling**: absorb spikes in front of a rate-limited database or API.
- **Retries with backoff**: let the visibility timeout expire (or change visibility per attempt) and use DLQ + redrive; add jitter.
- **Idempotent consumer**: dedupe on a business key (DB unique constraint, idempotency table) ([Idempotency](idempotency.md)).
- **Delayed/scheduled work**: delay queue for short delays; use EventBridge Scheduler/Step Functions for long ones.
- **Transactional outbox → SQS** for reliable DB + message publishing ([Reliable messaging](reliable_messaging.md)).

## Limits and gotchas
- No consumer-driven replay, no message-level retention after deletion, no ordering in Standard, no priorities (use separate queues).
- Visibility timeout too short → duplicate processing; too long → slow recovery after a crash.
- In-flight message limits (approx 120k standard); throttling on very high API rates; `Approximate*` metrics are eventually consistent.
- Cost is per request: batch and long-poll.

## When to use
- **SQS**: background jobs, email/notification sending, image processing, order fulfilment steps, buffering writes, simple decoupling on AWS with minimal ops.
- **Not SQS**: streaming analytics, replay, multiple independent consumers of the same stream with offsets (Kafka/Kinesis), sub-millisecond brokering, complex routing (RabbitMQ).

## Interview follow-ups
- *"What if a worker dies while processing?"* Visibility timeout expires, another worker receives it; hence idempotent handlers.
- *"How do you handle a poison message?"* `maxReceiveCount` → DLQ, alarm, inspect, fix, redrive.
- *"Need ordering?"* FIFO with a message group per entity; accept lower throughput.
- *"Need to process the same event in 3 services?"* SNS → 3 SQS queues (fan-out).
- *"Exactly once?"* FIFO dedup covers producer duplicates in a window; end-to-end still needs idempotent consumers ([Delivery semantics](delivery_semantics.md)).

# References
* [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
* [SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html), [Dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
