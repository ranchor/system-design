## Problem Statement
Design a **distributed locking service** (like Chubby, ZooKeeper locks, etcd leases, or a Redis-based lock) that lets processes on different machines get **mutual exclusion** on a named resource, safely
under failures, partitions, and pauses. Basics: [Distributed locking](../basics/distributed_locking.md), [Leader election](../basics/leader_election.md), [Consensus](../basics/consensus.md).

## Clarification Questions to Interviewer
- Purpose: **efficiency** (avoid duplicate work; occasional double-run is OK) or **correctness** (double-run corrupts data)? This decides the design.
- Lock types: exclusive only, or shared (read/write)? Re-entrant? Fair (FIFO)?
- Scale: locks/s, number of clients, lock hold times (ms vs minutes)?
- Availability vs safety under partition (CP expected)? Multi-region?
- Do protected resources support fencing tokens?

## Requirements
### Functional Requirements
- `acquire(lock, ttl, wait?)` → lease + **fencing token**; `release(lock, token)`; `renew(lock, token)`.
- Locks auto-expire if the holder dies (**lease/TTL**); optional **watch/notify** when a lock frees; optional shared (read) locks and queued waiters.
#### Below the line (out of scope)
- Deadlock detection across locks, transactions across many locks, lock hierarchies.

### Non-Functional Requirements
- **Safety first**: at most one holder at a time (given bounded clock drift); stale holders are rejected via fencing.
- **Liveness**: a crashed holder doesn't block forever; progress when a quorum is up.
- **Availability** via replication (3-5 nodes), failover in seconds; **low latency** (single-digit ms in a region).
- Linearizable lock state.
#### Below the line (out of scope)
- Global multi-region locking with sub-100 ms latency (physically limited).

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 100k clients, 10k lock ops/s peak: each op is a replicated write (consensus) → a 5-node Raft cluster handles ~10-50k small writes/s; shard by lock name range if more.
- Lease renewals dominate: 100k holders × 1 heartbeat per 3 s ≈ 33k/s → batch/multiplex heartbeats per **session** (ZooKeeper/Chubby sessions) rather than per lock.
- Lease TTL 10-30 s: failover detection vs false expiry (GC pauses) trade-off ([Heartbeats](../basics/heartbeats.md)).

## High-level API design
```
POST /locks/{name}/acquire   {clientId, ttlMs, waitMs}   → {granted, token (monotonic), expiresAt}
POST /locks/{name}/renew     {clientId, token}           → {expiresAt}
POST /locks/{name}/release   {clientId, token}
GET  /locks/{name}                                     → {holder, token, expiresAt}   (diagnostics)
WATCH /locks/{name}                                    → event stream (released/expired)
```
Token is a **monotonically increasing fencing token**; resources must reject writes with a token lower than the highest seen.

## Data Model
`lock(name, holder_id, token, lease_expiry, waiters[])` stored in a **replicated state machine** (Raft log). Token counter increments on every grant (Raft index / revision works: etcd `revision`, ZooKeeper `zxid`).
Sessions: `session(client_id, ttl, last_heartbeat)` with ephemeral locks tied to the session.

## High Level System Design
```
Clients ─▶ any node ─▶ Raft leader (linearizable writes) ─▶ replicate to quorum ─▶ apply (grant/expire)
   │  lease heartbeats via session                         └─ followers serve watches / stale reads
   └─ holder writes to resource with fencing token ─▶ Resource rejects token < max_seen
```
- A **3-or-5 node consensus cluster** (Raft/Paxos/Zab) stores locks; only the leader accepts writes; majority commit ensures a single linearizable history ([Consensus](../basics/consensus.md), [Quorum](../basics/quorum.md)).
- **Acquire** = conditional write ("create if absent" / compare-and-swap) committed through the log; waiting clients register a **watch** on the predecessor (ZooKeeper sequential ephemeral nodes: avoids herd effect).
- **Expiry** is decided by the **leader's** clock/timers (single time source), not by clients ([Time & ordering](../basics/time_ordering.md)).
- **Release** = delete if token matches.

## Deep Dive
### Why a naive Redis `SET NX PX` is not enough for correctness
Single-node Redis: failover loses the key (async replication) → two holders. Redlock (N independent nodes) relies on bounded clock drift and process pauses; Kleppmann's critique: a paused holder can act after its lease expired. Use it only for *efficiency* locks, and use **fencing tokens** for correctness.

### Fencing tokens
```
Client A acquires (token 33) → GC pause → lease expires → Client B acquires (token 34) → B writes (34 accepted)
→ A wakes and writes with 33 → storage rejects (33 < 34)
```
Requires the protected resource to check tokens (DB conditional update `WHERE token <= ?`, storage service support). If it can't, the lock is only advisory.

### Lease management
- TTL + **heartbeats** over a session; client library renews at TTL/3; on failure to renew it must assume the lock is lost and stop work (**self-fencing**).
- Server keeps a **safety margin**; leader election on the lock service itself invalidates in-flight leases conservatively (new leader waits a full TTL or uses epoch ordering) ([Leader election](../basics/leader_election.md)).

### Fairness and herd effect
FIFO queue of waiters; wake only the next waiter (watch the previous sequence node) instead of waking all.

### Shared (read/write) locks
Writers exclusive, readers shared; prevent writer starvation with a queue; implement via ordered sequence nodes as in ZooKeeper recipes ([ZooKeeper](../basics/zookeeper.md)).

### Availability and failure modes
- **Leader failure** → election (seconds), leases extended/handled conservatively.
- **Minority partition** → cannot grant locks (CP); clients there lose leases and must stop.
- **Client crash** → lease expires, lock freed. **Network delay** → duplicate/late requests deduped by `(clientId, requestId)` ([Idempotency](../basics/idempotency.md)).
- **Scale-out**: shard locks across multiple consensus groups by hash of name ([Consistent hashing](consistent_hashing_design.md)); each shard is independent.

### Alternatives and when not to lock
Database row lock / unique constraint / conditional write ([Contention](../basics/contention.md)); single-writer partition ownership; idempotent operations making locks unnecessary; a queue with one consumer per key.

### Pitfalls
- Locks without TTL; relying on local clocks; no fencing; holding locks across slow remote calls; lock service as hot path for high-frequency operations; renewing in the same thread as blocked work.

## References
* [Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html), [The Chubby lock service (OSDI '06)](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/)
* [ZooKeeper recipes](https://zookeeper.apache.org/doc/current/recipes.html), [etcd concurrency API](https://etcd.io/docs/latest/dev-guide/api_concurrency_reference_v3/)
