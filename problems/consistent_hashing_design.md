## Problem Statement
Design a **consistent hashing library/service** that maps keys to a changing set of nodes (cache servers, DB shards, partitions) so that adding or removing a node **moves only ~1/N of keys**, load stays
balanced, and replicas can be chosen deterministically. Concept note: [Consistent hashing](../basics/consistent_hashing.md); related: [Data partitioning](../basics/data_partitioning.md), [Distributed cache](distributed_cache.md), [Key-value store](key_value_store.md).

## Clarification Questions to Interviewer
- Use case: distributed cache, sharded DB, load balancer affinity, Dynamo-style store? (Decides replication and rebalancing semantics.)
- Cluster size and churn (10s vs 1000s of nodes; autoscaling frequency)?
- Heterogeneous node capacity? Zone/rack awareness?
- Who holds the ring: client library, proxy/router, or coordination service?
- Consistency during membership changes (can clients briefly disagree)?

## Requirements
### Functional Requirements
- `getNode(key) → node` (and `getReplicas(key, n)`); `addNode`, `removeNode`; membership propagation.
- Support **weights** (capacity) and **zone-aware** replica placement.
- Rebalancing hooks: identify keys/ranges that must move on membership change.
#### Below the line (out of scope)
- The data transfer protocol itself, replica repair details ([Merkle trees](../basics/merkle_tree.md)).

### Non-Functional Requirements
- **Minimal disruption**: O(K/N) keys remapped per node change.
- **Balance**: max load within ~10-20% of mean; mitigate hot spots.
- **Fast lookup**: O(log V) (V = virtual nodes) or O(1); microseconds, in-memory.
- **Availability**: ring view consistent enough; tolerate stale views gracefully.
#### Below the line (out of scope)
- Absolute load balancing under extreme per-key skew (needs [hot-key handling](../basics/hot_keys.md)).

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 1,000 nodes × 150 vnodes = 150,000 ring entries × ~32 B ≈ 5 MB → fits in every client's memory; binary search ~17 comparisons.
- With N nodes and one hash point per node, load std-dev is high (~`1/√1` effect); with ~100-200 vnodes/node, imbalance falls to a few percent.
- Adding 1 node to 100 moves ≈ 1% of keys vs ~99% for `hash(key) % N` ([Sharding](../basics/data_partitioning.md)).

## High-level API design
```
ring = Ring(hashFn=murmur3_64, vnodesPerNode=128)
ring.addNode(id, weight=1, zone="us-east-1a")     ring.removeNode(id)
ring.getNode(key) → nodeId                         ring.getReplicas(key, n=3) → [nodeId…] (distinct zones/nodes)
ring.diff(oldRing, newRing) → [(range, from, to)]  // keys to migrate
```

## Data Model
- Ring = sorted array of `(hashPosition, nodeId)` for all virtual nodes; node table `(nodeId, address, weight, zone, state)`; **ring version/epoch** for consistency.
- Stored in a coordination service ([ZooKeeper](../basics/zookeeper.md)/etcd) or gossiped ([Gossip](../basics/gossip_protocol.md)); clients cache it.

## High Level System Design
```
hash(key) ─▶ position on 0…2^64 ring ─▶ walk clockwise to the first vnode ─▶ owning node
Replicas: continue clockwise collecting the next distinct physical nodes (skip same node/zone)
```
- **Ring**: hash both nodes (vnodes: `hash(nodeId#i)`) and keys onto the same circular space; a key belongs to the **next vnode clockwise**.
- **Lookup**: binary search over the sorted vnode array.
- **Membership changes**: add node → its vnodes take over ranges from successors (only those ranges move); remove node → its ranges fall to successors. Streaming data movement happens in the background with versioned ring epochs.
- **Distribution of ring state**: control-plane service publishes ring versions; clients/proxies refresh periodically and on `moved` redirects; or peers gossip (Dynamo, Cassandra) ([Cassandra](../basics/cassandra.md)).

## Deep Dive
### Virtual nodes
Each physical node owns many small ranges → smoother balance, spreads a failed node's load across many peers (not just one neighbor), enables **weights** (more vnodes for bigger machines). Trade-offs: larger ring metadata, more complex replication/repair, many small streams during rebalancing.

### Replication
Dynamo-style **preference list**: first N *distinct physical* nodes clockwise (rack/zone aware) ([Replication](../basics/data_replication.md), [Quorum](../basics/quorum.md)).

### Alternatives (mention and compare)
| Algorithm | Idea | Pros | Cons |
|---|---|---|---|
| **Ring + vnodes** | Above | Simple, proven (Dynamo, Cassandra, memcached ketama) | Metadata size, imperfect balance |
| **Rendezvous (HRW)** | Pick node with highest `hash(key,node)` | No ring state, perfect minimal disruption, easy weights | O(N) per lookup (fine for small N) |
| **Jump consistent hash** | Arithmetic mapping to `[0,N)` buckets | O(ln N), no memory, ideal balance | Only add/remove the *last* bucket; no arbitrary node removal |
| **Maglev** | Lookup table with permutation | Great balance for LBs, O(1) | Table rebuild on change |
| **Bounded-load CH** | Cap load at `c×avg`, spill to next | Prevents overload hot spots | Slightly more movement |
| **Slot-based (Redis Cluster)** | Fixed 16,384 hash slots mapped to nodes | Explicit, easy to move slots | Manual/orchestrated rebalancing |

### Hot spots and skew
Hashing balances keys, not traffic: a viral key still hits one node. Mitigate with bounded loads, replicas serving reads, local caching, key salting ([Hot keys](../basics/hot_keys.md)).

### Consistency during churn
Different clients may hold different ring versions: use **versioned ring + redirects** (`MOVED`/`ASK`), read from old+new owner during migration (dual-read), **hinted handoff** for temporarily down nodes, and migrate with **throttling**. For caches, a miss on the new owner is acceptable (warm up lazily).

### Failure handling
Node down → temporarily route to successor (keep ring membership until confirmed dead to avoid thrash); use failure detection with hysteresis ([Heartbeats](../basics/heartbeats.md)); permanent removal triggers re-replication.

### Implementation sketch
```
vnodes = sorted([(hash(f"{node}#{i}"), node) for node in nodes for i in range(v)])
def owner(key): i = bisect_left(positions, hash(key)); return vnodes[i % len(vnodes)].node
```

### Pitfalls
- Few/no vnodes (imbalance); weak hash (clustered positions); removing nodes on a single missed heartbeat (rebalance storms); inconsistent ring versions across clients; treating it as load balancing for skewed traffic; forgetting replica placement across failure domains.

## References
* [Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), [Consistent Hashing and Random Trees (Karger et al.)](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
* [Consistent Hashing with Bounded Loads](https://research.google/blog/consistent-hashing-with-bounded-loads/), [Jump Consistent Hash](https://arxiv.org/abs/1406.2294), [Maglev](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/)
