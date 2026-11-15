# Handling Contention

> **TL;DR** — Contention = many actors updating the same data at once (last ticket, account balance, inventory). Options, from cheapest to heaviest: **atomic operations** (`UPDATE … SET x = x - 1 WHERE x > 0`),
> **optimistic concurrency** (version check, retry), **pessimistic locking** (`SELECT … FOR UPDATE`), **serialize through a queue / single writer**, **distributed locks with leases and fencing**,
> and **reservation with TTL** for user-facing holds. Prefer the one that keeps the critical section inside a *single database*; go distributed only when you must. See [Transactions](transactions.md),
> [Distributed locking](distributed_locking.md), [Idempotency](idempotency.md), [Ticketmaster](../problems/ticketmaster.md).

## Options
| Technique | How | Best when | Watch out |
|---|---|---|---|
| **Atomic / conditional write** | `UPDATE inv SET qty=qty-1 WHERE id=? AND qty>0`; DynamoDB `ConditionExpression`; Redis `INCR`/Lua | Simple counters, stock decrement | Check rows-affected; one row can become a [hot key](hot_keys.md) |
| **Optimistic locking** | `version` column; `UPDATE … WHERE id=? AND version=?`; retry on 0 rows | Low contention, long think-time edits | Retry storms under high contention |
| **Pessimistic locking** | `SELECT … FOR UPDATE`, row/advisory locks, `SKIP LOCKED` for work queues | High contention, short transactions | Deadlocks (lock ordering), lock waits, held across network calls ([Postgres & MySQL](postgres_mysql.md)) |
| **Isolation levels** | Serializable / snapshot with retries | Need invariants across rows | Serialization failures must be retried; cost ([Transactions](transactions.md)) |
| **Single writer / queue serialization** | Route all updates for a key to one partition/actor; process sequentially | Extreme contention, ordering needed | Throughput per key limited ([Kafka](kafka.md), [Async](asynchronism.md)) |
| **Distributed lock** | Redis/ZooKeeper/etcd lease around the critical section | Cross-service exclusive work | Needs **fencing tokens**; lease expiry/GC pauses ([Distributed locking](distributed_locking.md)) |
| **Reservation with TTL** | Create a `HELD` row/key with expiry; confirm on payment; auto-release | Checkout seats, bookings | Expiry sweeper, UX for expired holds |
| **Sharded resource** | Split stock across N buckets | Flash sale hot item | Rebalancing, small-bucket stockouts |
| **CRDT / commutative ops** | Merge-able updates (counters, sets) | Multi-region, collaborative | Not for hard invariants |

## Patterns
- **Double-booking prevention**: unique constraint on `(resource_id, slot)`; the DB rejects the second insert, which is simpler and safer than application checks.
- **Seat/ticket hold**: `status: AVAILABLE → HELD(expires_at) → SOLD`; conditional update `WHERE status='AVAILABLE' OR expires_at < now()`; background cleanup ([Ticketmaster](../problems/ticketmaster.md)).
- **Check-then-act race**: never `SELECT` then `UPDATE` without a lock/version; collapse into one atomic statement.
- **Idempotent retries**: contention often causes retries; guard with idempotency keys ([Idempotency](idempotency.md)).
- **Work-claiming**: `UPDATE jobs SET owner=? WHERE id IN (SELECT id … FOR UPDATE SKIP LOCKED LIMIT 10)`.
- **Cross-service invariants**: avoid distributed locks by redesigning ownership (one service owns the invariant) or using sagas ([Sagas](multi_step_processes.md)).

## Distributed-lock safety checklist
Lease with TTL; renew heartbeat; **fencing token** checked by the resource; handle lock holder pauses; prefer a consensus-backed lock (etcd/ZooKeeper) for correctness, Redis lock (`SET NX PX`) only for efficiency not correctness; keep critical sections short ([Leader election](leader_election.md)).

## Choosing quickly
1. Can one SQL statement do it atomically? → Do that.
2. Rare conflicts? → Optimistic + retry.
3. Frequent conflicts on same rows? → Pessimistic lock or queue/single writer.
4. User holds resource across minutes? → Reservation with TTL.
5. Multiple services/stores? → Redesign ownership, saga, or lease-based lock with fencing.

## Pitfalls
- Check-then-act without atomicity; holding DB locks while calling external services; locks without TTL (orphaned); lock ordering deadlocks; trusting Redis locks for correctness; unbounded optimistic retries; forgetting that retried requests need idempotency.

## Interview follow-ups
- *"Two users buy the last item."* Atomic conditional decrement or version check; loser gets a clear error.
- *"Optimistic vs pessimistic?"* Low vs high conflict rate; cost of retry vs blocking.
- *"Why not just use a Redis lock?"* Lease expiry + pauses can yield two holders; use fencing tokens or DB constraints for correctness.
- *"How do you scale beyond one row's throughput?"* Shard the resource, queue per key, or relax with approximate counts and reconcile.

# References
* [Hello Interview: Dealing with contention](https://www.hellointerview.com/learn/system-design/patterns/dealing-with-contention) · [Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
