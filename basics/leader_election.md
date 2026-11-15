# Leader Election

> **TL;DR** — Many systems need **exactly one node acting as leader** (primary DB, partition owner, scheduler, lock holder) to avoid conflicting work. **Leader election** chooses it and replaces it when it
> fails. Practical approaches: **consensus-based** (Raft/Paxos: majority vote with terms), **lease-based via a coordination service** (ZooKeeper/etcd/Consul/DynamoDB conditional write: hold a lease and
> renew it), or classic algorithms (Bully, Ring). The hard parts are **failure detection** (you can't tell slow from dead), **split-brain** (two leaders), and **fencing** (stopping the old leader's writes).
> Default answer: *use etcd/ZooKeeper/Raft, a leader holds a lease with a fencing token.* See [Consensus](consensus.md), [ZooKeeper](zookeeper.md), [Distributed locking](distributed_locking.md), [Heartbeats](heartbeats.md).

## Why leaders
- A single writer simplifies ordering and consistency (single-leader replication, [Replication](data_replication.md)).
- Coordination of exclusive work: cron-like schedulers, shard/partition owners, cluster managers, config/metadata (Kafka controller, HDFS NameNode, Kubernetes controllers).
- Trade-off: the leader is a throughput bottleneck and a failure point; election time = unavailability window.

## Approaches
### 1. Consensus (Raft / Paxos)
- Nodes are followers/candidates/leader; time is divided into **terms**. A follower that stops hearing **heartbeats** within a randomized **election timeout** becomes a candidate, increments its term and requests votes;
  a **majority** (quorum) elects it ([Quorum](quorum.md)). Only one leader per term; stale leaders step down on seeing a higher term. Randomized timeouts avoid repeated ties.
- Safety: a candidate must have an up-to-date log (Raft), so committed entries are never lost. Used inside etcd, Consul, CockroachDB ranges, Kafka KRaft, MongoDB replica sets ([MongoDB](mongodb.md)).

### 2. Lease-based election via a coordination service
- All candidates try to create/acquire a key atomically: ZooKeeper **ephemeral sequential znodes** (lowest sequence is leader; others watch their predecessor to avoid herd), etcd **lease + transaction**,
  Consul sessions, Redis `SET NX PX` (weaker), **DynamoDB/Postgres conditional write** on a row with expiry.
- The leader **renews** the lease before it expires; if it crashes or is partitioned, the lease lapses and another node takes over.
- **Fencing token**: each election returns a monotonically increasing number (ZooKeeper `zxid`/etcd revision/term). Resources reject writes with a stale token, preventing a paused ex-leader from corrupting data ([Distributed locking](distributed_locking.md)).

### 3. Classic algorithms
- **Bully**: highest-id live node wins; a node that detects failure messages higher ids; simple but O(n²) messages and assumes reliable failure detection.
- **Ring**: pass an election message around a logical ring carrying the highest id. Mostly textbook.
- **Gossip/membership-based** (Cassandra/Redis Cluster): no single leader for data (leaderless) but elect coordinators/masters for slot failover via epochs ([Gossip](gossip_protocol.md)).

## Failure detection and timing
- Followers infer leader death from missing heartbeats ([Heartbeats](heartbeats.md)); choose timeouts ≫ network RTT and GC pauses but short enough for fast recovery (typically 150 ms-10 s). Too short → needless elections (flapping); too long → long outages.
- Phi-accrual detectors adapt to network conditions (Cassandra, Akka).

## Split-brain and safety
- A network partition can leave two groups each thinking they should lead. Prevent with **majority quorums** (only the majority side elects), **odd node counts**, **leases + fencing tokens**, and **STONITH** (shoot the other node in the head) for traditional HA pairs.
- **Lease clock assumptions**: lease expiry relies on bounded clock drift; use conservative margins and fencing for correctness ([Time & ordering](time_ordering.md)).
- A leader that loses quorum contact must **step down** (check-quorum) rather than keep serving.

## Patterns and examples
| System | How the leader is chosen |
|---|---|
| etcd / Consul / Kafka KRaft controller | Raft |
| ZooKeeper (Zab) | Quorum-based leader; apps use ephemeral znodes for their own election |
| Kubernetes controllers/schedulers | Lease object in etcd (`leader-election` with renew deadline) |
| MongoDB replica set | Raft-like election, majority, priority/votes |
| PostgreSQL HA (Patroni) | Leader key with TTL in etcd/Consul/ZooKeeper |
| Redis Sentinel/Cluster | Majority of sentinels / masters vote with epochs |
| Cron-like job scheduler | Lease in DB/ZooKeeper; one active scheduler, standby takes over ([Job Scheduler](../problems/distributed_job_scheduler.md)) |

## When *not* to elect
- If work can be **partitioned** (consistent hashing, partition ownership), prefer many leaders each owning a slice over one global leader ([Sharding](data_partitioning.md), [Consistent hashing](consistent_hashing.md)).
- If duplicate work is harmless (idempotent), skip election and let all workers compete with a claim table (`SELECT … FOR UPDATE SKIP LOCKED`, [Postgres & MySQL](postgres_mysql.md)).

## Pitfalls
- Even node counts or two-node clusters without a tiebreaker; no fencing so a GC-paused leader overwrites data; renewing leases with the same thread that does heavy work; ignoring clock drift in lease math.
- Election storms due to tight timeouts on noisy networks; leader doing all reads and writes without scaling plan.

## Interview follow-ups
- *"How do you ensure only one scheduler runs the job?"* Lease in etcd/ZooKeeper (or DB row with TTL) + fencing token on side effects + idempotent jobs.
- *"What happens during a partition?"* Minority side can't reach quorum → no leader/steps down; majority elects a new one; old leader is fenced.
- *"How long does failover take?"* Detection timeout + election round (+ catch-up). Tune timeouts vs flapping.
- *"Raft vs ZooKeeper election?"* Raft embeds election in the replicated log; ZooKeeper offers primitives (ephemeral nodes/watches) for application-level election.

# References
* [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf), [The Raft site](https://raft.github.io/)
* [Martin Kleppmann: How to do distributed locking (fencing tokens)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
* [ZooKeeper recipes: Leader election](https://zookeeper.apache.org/doc/current/recipes.html#sc_leaderElection)
