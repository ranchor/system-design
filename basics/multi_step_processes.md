# Multi-Step Processes: Sagas, Orchestration & Workflows

> **TL;DR** — A business flow that spans services (order → payment → inventory → shipping) can't use one ACID transaction. Use a **saga**: a sequence of **local transactions**, each with a **compensating
> action** that semantically undoes it if a later step fails. Coordinate by **choreography** (services react to each other's events) or **orchestration** (a central coordinator/workflow engine drives steps).
> For long, branching, human-in-the-loop flows use a **durable workflow engine** (Temporal, AWS Step Functions, Cadence). Steps must be **idempotent**, and state must be **persisted** so the process
> resumes after crashes. See [Distributed transactions](distributed_transactions.md), [Reliable messaging](reliable_messaging.md), [Idempotency](idempotency.md).

## Why not 2PC
Two-phase commit blocks on coordinator failure, holds locks across services, and isn't supported by most SaaS/NoSQL/queues. Sagas trade **isolation** for **availability and autonomy**: intermediate states are visible.

## Saga basics
- Steps `T1…Tn` with compensations `C1…Cn-1`. If `Tk` fails, run `Ck-1 … C1` in reverse.
- **Compensation ≠ rollback**: it's a new business action (refund, release stock, cancel reservation) and may itself need retries until success.
- **Pivot / point of no return**: after a non-compensable step (e.g. item shipped), only forward-recovery (retry) remains. Order steps so risky/compensable ones come first and the pivot comes late.
- **Semantic locks / pending states**: mark entities `PENDING` so other flows don't act on half-complete data ("countermeasures" to missing isolation).

## Orchestration vs choreography
| | Orchestration | Choreography |
|---|---|---|
| Control | Central orchestrator calls steps and tracks state | Each service emits/consumes events |
| Visibility | Easy: one place shows the flow's state | Hard: flow implicit across services |
| Coupling | Orchestrator knows all participants | Loose, but cyclic dependencies creep in |
| Change | Edit one workflow | Touch many services |
| Failure handling | Explicit retries/timeouts/compensation | Each service must implement |
| Best for | Complex/long flows, many branches, auditing | Simple 2-4 step flows, event-driven domains |

## Example: Order saga (orchestrated)
```
CreateOrder(PENDING) → ReserveInventory → ChargePayment → CreateShipment → Order(CONFIRMED)
  fail at ChargePayment → ReleaseInventory → Order(CANCELLED)
  fail at CreateShipment → RefundPayment → ReleaseInventory → Order(CANCELLED)
```

## Durable workflow engines
- **Temporal/Cadence, AWS Step Functions, Azure Durable Functions, Camunda**: the engine persists workflow state (event history), retries activities with backoff, supports timers (wait 3 days), signals (human approval), child workflows, and deterministic replay.
- Benefits: crash-safe, built-in retries/timeouts, visibility, versioning. Costs: operational dependency, determinism constraints in workflow code, history growth.
- Use for: payments/refunds, onboarding, KYC, provisioning, [job pipelines](long_running_tasks.md).

## Making steps reliable
1. **Idempotent step handlers** keyed by `(workflow_id, step)`; safe to retry ([Idempotency](idempotency.md), [Delivery semantics](delivery_semantics.md)).
2. **Persist before acting**: write the saga state (outbox) in the same transaction as the local change, then publish ([Reliable messaging](reliable_messaging.md)).
3. **Timeouts and deadlines** per step; escalate to compensation or manual review.
4. **Compensation must be retriable and idempotent**; track compensation state.
5. **Observability**: workflow id as correlation id; dashboards for stuck sagas; DLQ for poison steps ([Monitoring](monitoring.md)).
6. **Ordering**: use per-entity partitioning so events for one order stay ordered ([Kafka](kafka.md)).
7. **Concurrency**: detect concurrent modification with versions ([Contention](contention.md)).

## Alternatives
- **Single-service transaction**: redesign boundaries so the invariant lives in one service/database. Often the best answer.
- **TCC (Try-Confirm-Cancel)**: reserve resources in a Try phase, then Confirm or Cancel; resource owners expose three operations.
- **Event-driven eventual consistency with reconciliation**: periodic jobs detect and repair drift ([Payment gateway](../problems/payment_gateway.md)).

## Pitfalls
- Compensations that can fail permanently with no operator path; non-idempotent steps; dual writes (DB + message) without an outbox; hidden cyclic choreography; no timeout so sagas hang; exposing intermediate states to users without explanation; unbounded workflow history.

## Interview follow-ups
- *"Payment succeeded but inventory failed."* Compensate with a refund (idempotent, retried), mark order cancelled, notify.
- *"Orchestration or choreography?"* Orchestration for complex/long flows needing visibility; choreography for small event-driven flows.
- *"Why not 2PC?"* Blocking, availability cost, unsupported by components, lock duration.
- *"How do you survive an orchestrator crash?"* State is persisted; a new worker resumes from the last completed step.

# References
* [Microservices.io: Saga pattern](https://microservices.io/patterns/data/saga.html), [Temporal docs](https://docs.temporal.io/), [AWS Step Functions](https://docs.aws.amazon.com/step-functions/)
* [Hello Interview: Multi-step processes](https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes) · Garcia-Molina & Salem, "Sagas" (1987)
