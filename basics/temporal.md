# Temporal (Durable Workflow Orchestration)

> **TL;DR** — **Temporal** runs **durable, long-lived workflows written as ordinary code**. The Temporal service records every step in an **event history**; if a worker crashes, another worker **replays** the history to rebuild state and continues exactly where it left off. You get retries, timers (sleep for days), signals, queries, child workflows, sagas/compensation and visibility without hand-building state machines on queues.
> Use it for **multi-step, long-running, failure-prone business processes** (payments, onboarding, order fulfilment, media pipelines). Avoid it for simple fire-and-forget jobs or very high-throughput, low-latency request paths. Related: [Long-running tasks](long_running_tasks.md), [Multi-step processes / sagas](multi_step_processes.md), [Reliable messaging](reliable_messaging.md), [Idempotency](idempotency.md), [Job scheduler](../problems/distributed_job_scheduler.md).

## Core concepts
| Concept | Meaning |
|---|---|
| **Workflow** | Deterministic code defining the orchestration; its state is reconstructed from history, so it can run for days or years |
| **Activity** | A unit of side-effecting work (call API, charge card, transcode). Non-deterministic is fine; has **timeouts, retry policy, heartbeats**. Should be **idempotent** (at-least-once) |
| **Worker** | Your process that polls a **task queue** and executes workflow/activity code. Stateless, scale horizontally |
| **Task queue** | Named queue the service dispatches tasks to; route by workload (e.g. GPU vs CPU activities) |
| **Event history** | Append-only log per workflow execution; the source of truth |
| **Signal / Query / Update** | Send data into a running workflow / read its state / synchronous validated mutation |
| **Timer** | Durable sleep (`workflow.sleep`) that survives restarts |
| **Child workflow, continue-as-new** | Decompose; reset history to keep it bounded |
| **Namespace** | Isolation unit (tenant/environment) |

## Architecture
```
Client/API ─start workflow─▶ Temporal Service (frontend, history, matching, worker svc) ─▶ persistence (Cassandra/PostgreSQL/MySQL)
                                        ▲ task queues │ poll
                                   Workers (your code: workflows + activities) ── call external systems
```
- The service persists history and timers and hands out tasks; **it does not run your code**. Workers do.
- **Replay**: to resume, a worker re-executes workflow code against the history, with recorded activity results returned instead of re-run. This demands **deterministic workflow code**.

## Determinism rules (the main gotcha)
Inside workflow code do **not**: read the clock or random numbers directly, do I/O, use threads/non-deterministic iteration, or call non-versioned changed logic. Use the SDK's `workflow.now()`, `workflow.random`, activities for I/O, and **versioning/patching** when changing code with in-flight executions. History size is bounded (use `continue-as-new` for loops/long-lived entities).

## Reliability model
- **Activities**: at-least-once → idempotent with idempotency keys (workflow ID + activity ID are convenient). Timeouts: schedule-to-start, start-to-close, schedule-to-close, heartbeat; retry with exponential backoff, non-retryable error types.
- **Workflows**: exactly-once *workflow* execution semantics per workflow ID (ID reuse policy prevents duplicates), effectively exactly-once orchestration over at-least-once activities.
- **Sagas**: record compensating actions as you go and run them on failure ([Sagas](multi_step_processes.md)).
- **Human-in-the-loop & waiting**: wait for a signal with a timeout; no polling worker or DB state machine needed.
- **Scheduling**: cron/schedules replace external cron + leader election ([Leader election](leader_election.md)).

## When Temporal vs a queue + workers
| Need | Queue + workers | Temporal |
|---|---|---|
| Single idempotent task, retries, DLQ | ✓ simplest | overkill |
| Multi-step flow with branching, timers, compensation | hand-rolled state machine | ✓ code-as-workflow |
| Long waits (days), human approvals, signals | awkward | ✓ |
| Visibility of "where is this order?" | custom | built-in history/UI |
| Very high-throughput tiny tasks | ✓ | per-workflow overhead (history writes) |
| Operational footprint | low | run/pay for a cluster (self-hosted or Temporal Cloud) |

Alternatives: AWS Step Functions (declarative JSON, managed), Cadence (Temporal's predecessor), Airflow (batch DAG scheduling, not request-driven orchestration), Camunda/Conductor.

## Design patterns
- **Order/payment saga**: reserve inventory → charge → ship; compensations on failure.
- **Entity workflow**: one long-lived workflow per user/order that serializes updates via signals (natural per-entity ordering, avoids distributed locks).
- **Fan-out/fan-in**: child workflows or parallel activities; cap concurrency.
- **Polling → signals**: avoid busy polling; have the external system call back and signal the workflow.
- **Large payloads**: keep inputs/results small; store blobs in object storage and pass references ([Large blobs](large_blobs.md)).

## Pitfalls
- Non-deterministic workflow code (replay failures); non-idempotent activities; unbounded history (no `continue-as-new`); giant payloads in history; changing workflow code without versioning; tiny activities at massive scale; treating it as a message queue; forgetting heartbeats on long activities (slow failure detection).

## Interview follow-ups
- *"How does Temporal survive a worker crash?"* State lives in the service's event history; another worker replays it and resumes after the last completed activity.
- *"Why must workflow code be deterministic?"* Replay must reproduce the same decisions from the same history.
- *"Exactly-once?"* Activities are at-least-once; make them idempotent, and the workflow engine guarantees the orchestration progresses once.
- *"When not to use it?"* Simple jobs, ultra-low-latency paths, extreme task rates with trivial work.

# References
* [Temporal documentation](https://docs.temporal.io/)
* [Temporal: Workflows](https://docs.temporal.io/workflows) · [Activities](https://docs.temporal.io/activities) · [Workers](https://docs.temporal.io/workers)
* [Hello Interview: Temporal](https://www.hellointerview.com/learn/system-design/deep-dives/temporal)
