# Long-Running Tasks

> **TL;DR** — Don't hold an HTTP request open for work that takes seconds to hours (video transcode, report export, ML inference, bulk import). **Accept → enqueue → return `202 Accepted` + job id**;
> **workers** pull jobs from a **queue**, report **progress/state** in a job store, and clients **poll, stream (SSE/WebSocket), or get a webhook** on completion. Make jobs **idempotent**, **checkpoint**
> progress, use **visibility timeouts + heartbeats**, **retries with backoff**, a **DLQ**, and **priority/fair queues**. For multi-step flows use a workflow engine. See [Async](asynchronism.md), [SQS](sqs.md), [Kafka](kafka.md), [Job scheduler](../problems/distributed_job_scheduler.md), [Sagas](multi_step_processes.md).

## Architecture
```
Client ─POST /jobs─▶ API ─▶ Job store (PENDING) + Queue ─▶ Worker pool ─▶ results (S3/DB)
   ▲  202 + jobId                                   │ heartbeat/progress
   └──── poll GET /jobs/{id} · SSE/WebSocket · webhook ◀── status updates
```

## Components
| Piece | Role | Options |
|---|---|---|
| **API** | Validate, create job, return id (idempotency key to dedupe submits) | REST `202 Accepted` + `Location` |
| **Job store** | State machine `PENDING → RUNNING → SUCCEEDED/FAILED/CANCELLED`, attempts, progress, result pointer | Postgres/DynamoDB |
| **Queue** | Buffer + load leveling; at-least-once | SQS, Kafka, RabbitMQ, Redis streams ([Kafka vs SQS](kafka_vs_sqs.md)) |
| **Workers** | Stateless, autoscaled on queue depth/age | ECS/K8s/Lambda; scale by backlog |
| **Scheduler** | Delayed/cron jobs, retries | Delay queues, scheduler tables, Temporal, Quartz ([Job scheduler](../problems/distributed_job_scheduler.md)) |
| **Result storage** | Large outputs in object storage; pointer in job record | [Blob storage](blob_storage.md) |
| **Notification** | Completion via polling, SSE/WS, webhook, push | [Real-time updates](realtime_updates.md) |

## Reliability rules
1. **Idempotency**: job handler safe to re-run (same inputs → same effect); use idempotency keys for side effects ([Idempotency](idempotency.md)).
2. **Visibility timeout / lease + heartbeat**: long jobs extend their lease; if the worker dies, the message reappears and another worker retries ([SQS](sqs.md)). Set timeout ≥ P99 runtime or heartbeat.
3. **Checkpointing**: persist progress (offset, step) so retries resume rather than restart; split big jobs into **chunks/subtasks** (map-reduce style, fan-out/fan-in).
4. **Retries**: exponential backoff + jitter, max attempts, classify retryable vs fatal errors; DLQ for poison jobs ([Reliable messaging](reliable_messaging.md)).
5. **Exactly-once effect** via dedupe table or conditional state transition (`UPDATE … WHERE status='PENDING'` claims the job) ([Delivery semantics](delivery_semantics.md)).
6. **Timeouts and cancellation**: cooperative cancel flag checked between steps; kill stuck jobs.
7. **Poison/slow jobs isolation**: separate queues per job class so heavy work doesn't starve light work; priority queues and per-tenant fairness/quotas ([Backpressure](backpressure_load_shedding.md)).
8. **Graceful shutdown**: finish or release in-flight jobs on SIGTERM/spot interruption.

## Choosing a mechanism
| Need | Use |
|---|---|
| Simple background tasks, retries, DLQ | SQS/RabbitMQ + workers |
| Ordered per-entity processing, replay | Kafka |
| Multi-step, long (days), human steps, timers | Workflow engine (Temporal, Step Functions) ([Temporal](temporal.md), [Sagas](multi_step_processes.md)) |
| Scheduled/recurring | Cron + leader election or scheduler service ([Leader election](leader_election.md)) |
| Very large batch | Spark/Flink on a cluster ([Spark](spark.md)) |
| Short (<15 min), spiky | Serverless functions with queue triggers |

## Client experience
- **Polling**: `GET /jobs/{id}` with `Retry-After`; cheap and simple.
- **SSE/WebSocket**: push progress for interactive UIs; **webhook** for server-to-server ([Real-time updates](realtime_updates.md), [LP/WS/SSE](lp_ws_sse_wh.md)).
- Return partial progress (`percent`, `stage`) and a result URL (presigned) when done ([Large blobs](large_blobs.md)).

## Common deep dives
**Handling failures**: workers crash or hang mid-job. Use a lease/visibility timeout plus heartbeats; an expired lease re-queues the job. Persist job state (`PENDING → RUNNING → SUCCEEDED/FAILED`) in the job store, not in worker memory.

**Handling repeated failures**: cap attempts, use exponential backoff + jitter, classify retryable vs permanent errors, and move **poison jobs to a DLQ** with alerting and a replay tool. Add a circuit breaker if a downstream dependency is the cause ([Resilience](resilience_patterns.md)).

**Preventing duplicate work**: at-least-once queues redeliver. Combine an **idempotency key** on submit (dedupe at the API), an **atomic claim** (`UPDATE … WHERE status='PENDING'`) and idempotent side effects ([Idempotency](idempotency.md)). Dedupe identical in-flight jobs by a deterministic job key.

**Managing queue backpressure**: bound queue length, reject or defer new work when the backlog age exceeds SLO (429 + `Retry-After`), autoscale workers on **oldest-message age**, and rate-limit per tenant ([Backpressure](backpressure_load_shedding.md)).

**Handling mixed workloads**: separate queues/worker pools per job class (short vs long, CPU vs GPU), priority queues, per-tenant fairness quotas, and timeouts per class so heavy jobs can't starve light ones.

**Orchestrating job dependencies**: for DAGs and multi-step flows use a workflow engine; Temporal/Step Functions track step state, retries, timers and compensation ([Temporal](temporal.md), [Sagas](multi_step_processes.md)). For simple fan-out/fan-in use a parent job with a counter or completion barrier in the job store; for scheduled batch DAGs, a scheduler such as Airflow.

## Pitfalls
- Synchronous requests for slow work (timeouts, thread exhaustion); visibility timeout shorter than the job; non-idempotent retries (double emails/charges); unbounded queues; no DLQ; jobs holding DB locks for minutes; losing state on worker crash because progress lives in memory; one queue for everything.

## Interview follow-ups
- *"A job takes 30 minutes and the worker crashes at minute 20."* Lease expires, job re-queued; resume from checkpoint or restart idempotently.
- *"How do you prevent two workers running the same job?"* Atomic claim (conditional update) + lease; still make the handler idempotent.
- *"How do you scale workers?"* Autoscale on queue depth/oldest-message age; per-class queues.
- *"How does the client learn it's done?"* Poll, SSE/WS, or webhook; choose by client type.

# References
* [Hello Interview: Managing long-running tasks](https://www.hellointerview.com/learn/system-design/patterns/long-running-tasks) · [AWS Builders' Library: Avoiding insurmountable queue backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
