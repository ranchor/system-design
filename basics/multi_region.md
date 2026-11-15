# Multi-Region / Geo-Distribution

> **TL;DR** — Go multi-region for **latency** (serve users from nearby), **availability/DR** (survive a region outage), and **data residency** (GDPR etc.). The hard part is **data**: the speed of light
> (~70-150 ms cross-continent RTT) forces a choice per dataset between **single-writer (home region)** with cross-region async replicas, **multi-writer with conflict resolution** (LWW, CRDTs), or **globally
> consistent consensus** (Spanner-style, higher write latency). Stateless tiers are easy (GeoDNS/anycast + local stacks); decide **active-passive vs active-active**, RPO/RTO, and what to do on failover.
> See [CAP & PACELC](cap_theorem.md), [Replication](data_replication.md), [CDN](cdn.md), [Load balancing](load_balancing.md), [Availability](availability.md).

## Deployment models
| Model | Description | RPO / RTO | Cost / complexity |
|---|---|---|---|
| **Single region, multi-AZ** | Survives AZ loss | ~0 / minutes | Baseline (usually enough!) |
| **Active-passive (pilot light/warm standby)** | Standby region with replicated data, scaled up on failover | RPO = replication lag; RTO = minutes-hours | Medium |
| **Active-active, partitioned (home region per user/tenant)** | Each user's data homed in one region; others proxy or read replicas | RPO small, RTO small per partition | Medium-high; routing needed |
| **Active-active, multi-writer** | Any region accepts writes; merge conflicts | Near-zero | High; conflict semantics |
| **Global consensus DB** | Spanner/CockroachDB/DSQL: synchronous quorum across regions | RPO 0 | Higher write latency, cost ([Aurora DSQL](aurora_dsql.md)) |

## Routing traffic
- **GeoDNS / latency-based DNS** (Route 53), **anycast** (Cloudflare, Global Accelerator), **global load balancers**; health checks to evacuate a region ([Load balancing](load_balancing.md)).
- **CDN** at the edge for static/cacheable content and TLS termination ([CDN](cdn.md)); edge compute for light logic.
- Pin sessions: user → home region; stateless tokens (JWT) so any region can authenticate.

## Data strategies
1. **Home region per entity** (user/tenant/document): all writes go to the home region (single leader), reads local-or-home. Simple consistency; cross-region writes pay latency; moving users is an operation.
2. **Async cross-region replicas**: low-latency local reads, stale by replication lag; failover may lose last seconds (RPO>0) ([Replication](data_replication.md)).
3. **Multi-leader / leaderless with conflict resolution**: DynamoDB Global Tables (last-writer-wins), Cassandra multi-DC with `LOCAL_QUORUM`, CRDTs for counters/sets/collaborative docs ([DynamoDB](dynamodb.md), [Cassandra](cassandra.md), [Time & ordering](time_ordering.md)).
4. **Synchronous global consensus**: Spanner (TrueTime), CockroachDB, Aurora DSQL: strong consistency across regions at the price of cross-region round trips per commit ([PACELC: EC](cap_theorem.md)).
5. **Caches**: regional caches with TTL; invalidation across regions via pub/sub; accept staleness.
6. **Event streams**: replicate Kafka topics (MirrorMaker 2, Cluster Linking) for regional processing ([Kafka](kafka.md)).
7. **Blobs**: cross-region replication of object storage ([Blob storage](blob_storage.md)).

## Choosing per data class
| Data | Typical choice |
|---|---|
| Profile/settings | Home region + async replicas; stale reads ok |
| Payments/ledger/inventory | Single-writer or consensus; no LWW ([Payment](../problems/payment_gateway.md)) |
| Social feed, likes | Multi-region eventual, CRDT counters |
| Sessions | Replicated cache or signed stateless tokens |
| Static assets | CDN + multi-region bucket |
| Compliance data | Pinned to residency region; no replication out |

## Failover
- **Detect** (multi-signal health checks), **decide** (automated vs human; avoid flapping), **promote** the standby (fencing the old primary to prevent split-brain, [Leader election](leader_election.md)), **redirect** traffic (DNS TTL, anycast), **reconcile** lost writes after the old region returns.
- Test with **game days**; keep runbooks; ensure capacity in the surviving region (N+1 regions: each must absorb the other's load).
- **Dependencies**: global services (auth, config, DNS, secrets) must not be single-region SPOFs.

## Pitfalls
- Assuming the app works multi-region because the DB does (chatty cross-region calls, sticky local state); LWW on money; DNS TTL too long for failover; untested failover; replication lag unmonitored; ignoring data residency; thundering herd on the standby when traffic shifts.

## Interview follow-ups
- *"Design for a region outage."* Active-passive or active-active with health-based routing, replicated data with stated RPO/RTO, fencing, tested runbooks.
- *"How do you keep users consistent globally?"* Home region per user, or consensus DB; explain the latency/consistency trade-off (PACELC).
- *"Two regions update the same item."* Single-writer avoids it; otherwise LWW/CRDT/app-level merge; never LWW for monetary balances.
- *"Do you need multi-region at all?"* Often multi-AZ is enough; add regions for latency, DR requirements, or residency.

# References
* [AWS: Multi-Region Fundamentals](https://docs.aws.amazon.com/whitepapers/latest/aws-multi-region-fundamentals/aws-multi-region-fundamentals.html), [Spanner paper](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)
* Designing Data-Intensive Applications, Chapters 5 & 9
