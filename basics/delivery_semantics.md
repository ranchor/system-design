# Delivery Semantics: At-most-once, At-least-once, Exactly-once

> **TL;DR** — Messages and RPCs can be lost, duplicated or delayed. **At-most-once**: send once, never retry (may lose). **At-least-once**: retry until acknowledged (may duplicate). **Exactly-once
> delivery is impossible in the general case** (Two Generals), but **exactly-once *effects*** are achievable by combining **at-least-once delivery + idempotent/deduplicated processing** (or atomic
> commit of the output with the input offset). Default design: *at-least-once + idempotency key*. See [Idempotency](idempotency.md), [Kafka](kafka.md), [SQS](sqs.md), [Reliable messaging](reliable_messaging.md).

## The three semantics
| Semantics | Mechanism | Failure behavior | Use for |
|---|---|---|---|
| **At-most-once** | Send, don't retry (ack before processing / fire-and-forget) | Loss on failure, no duplicates | Metrics, logs where loss is OK, real-time telemetry, UDP-like |
| **At-least-once** | Retry until ack; commit offset/delete **after** processing | Duplicates on retry/crash, no loss | Most business events; needs idempotent consumers |
| **Exactly-once (effect)** | At-least-once + dedup/idempotency, or transactional read-process-write | Each message's *effect* applied once | Payments, inventory, ledgers |

## Why exactly-once delivery can't exist
If the consumer processes a message then crashes before acking, the broker can't know whether the effect happened: it either redelivers (maybe duplicate) or doesn't (maybe loss). Likewise a
client timeout doesn't tell if the server executed the request. So systems choose redelivery and make processing **idempotent**.

## Where duplicates come from
1. **Producer retry** after a timeout when the first send actually succeeded.
2. **Broker redelivery** (visibility timeout expired, consumer crashed before commit, rebalance).
3. **Replays** (reprocessing a log, backfill, DLQ redrive).
4. **Upstream retries** (client double-click, gateway retry, at-least-once webhooks, [Payment webhook](../problems/payment_webhook.md)).

## Making processing exactly-once in effect
1. **Idempotency key**: client or producer generates a unique id per logical operation; the server stores it (unique constraint) and returns the previous result for repeats ([Idempotency](idempotency.md)).
2. **Natural idempotence**: `SET x = 5`, upsert by primary key, `PUT` not `POST`, "mark order paid" (state machine) instead of `+= 10`.
3. **Deduplicate (inbox pattern)**: record processed message ids in the same DB transaction as the effect (`INSERT inbox(id)` fails on duplicates), with a retention window.
4. **Transactional offset + output**: commit consumer offset and results atomically: Kafka transactions (read-process-write inside Kafka), or store the offset in the same DB transaction as the result.
5. **Versioning / conditional writes**: `UPDATE … WHERE version = ?`; DynamoDB conditional writes ([DynamoDB](dynamodb.md)).
6. **Two-phase commit sinks** (Flink's TwoPhaseCommitSinkFunction) for exactly-once to external systems ([Flink](flink.md)).
7. **Outbox**: atomic DB write + event, relay is at-least-once, consumers dedupe ([Reliable messaging](reliable_messaging.md)).

## How popular systems phrase it
| System | Default | Exactly-once options |
|---|---|---|
| **Kafka** | At-least-once | Idempotent producer (no dups per partition), transactions for atomic multi-partition writes + offsets; consumers use `read_committed` |
| **SQS Standard** | At-least-once | None; consumers dedupe |
| **SQS FIFO** | Exactly-once *processing* semantics within a 5-minute dedup window | Plus idempotent consumers for crash/visibility redelivery |
| **Flink / Spark Streaming** | Exactly-once state via checkpoints | Transactional/idempotent sinks for end-to-end |
| **HTTP (client→server)** | Client retries → at-least-once | `Idempotency-Key` header (Stripe-style) |
| **RabbitMQ** | At-least-once with acks | Consumer dedupe |
| **Databases** | Transactions are atomic | Unique constraints / upserts |

## Ordering and delivery
Retries can reorder messages; ordering needs partition-per-key and a single in-flight message per key (or sequence numbers). Mention both when asked about "exactly-once and in order".

## Choosing in an interview
- Pick **at-least-once + idempotent handlers** unless loss is acceptable.
- State **where deduplication lives** (DB unique key, idempotency table, offset+output transaction) and **how long** you keep keys.
- For multi-service business flows use sagas with idempotent steps and compensations ([Distributed transactions](distributed_transactions.md)).

## Pitfalls
- Committing the offset before processing (silent loss) or auto-commit with slow processing; dedup store with no TTL or without atomicity with the effect.
- Non-idempotent side effects (emails, charges) inside a retried block without a guard; "exactly-once" marketing claims without scoping (it's per system boundary).
- Idempotency key reuse with different payloads (should be rejected); keys that aren't stable across retries.

## Interview follow-ups
- *"Can you guarantee exactly-once delivery?"* No; we guarantee exactly-once effect via at-least-once + idempotency.
- *"A payment call times out. What do you do?"* Retry with the same idempotency key; the server returns the original result; reconcile if still unknown ([Payment system](../problems/payment_gateway.md)).
- *"How does Kafka achieve exactly-once?"* Idempotent producer sequence numbers + transactions + `read_committed` consumers, within Kafka boundaries.
- *"How long do you keep dedup records?"* At least the maximum retry/redelivery window (plus clock/replay margin).

# References
* [Exactly-once Semantics are Possible: Here's How Kafka Does it](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
* [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) · Designing Data-Intensive Applications, Chapter 11
