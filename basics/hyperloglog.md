# HyperLogLog (Cardinality Estimation)

> **TL;DR** — **HyperLogLog (HLL)** estimates the number of **distinct elements** (cardinality) of a huge stream using a tiny, fixed amount of memory (e.g. **~12 KB for ~0.8% error** in Redis, regardless
> of whether there are a thousand or a billion uniques). It hashes each item, uses the first bits to pick a **register**, and records the **longest run of leading zeros** seen; the harmonic mean of
> registers gives the estimate. HLLs are **mergeable** (union by max per register), so they work across shards and time windows. Use for unique visitors, distinct counts per key per day, approximate
> `COUNT(DISTINCT)`. Not for membership ([Bloom filter](bloom_filter.md)) or frequencies ([Count-Min Sketch](count_min_sketch.md)).

## The problem
Exact `COUNT(DISTINCT user_id)` needs memory proportional to the number of distinct values (a set/hash table; 1 billion 8-byte ids ≈ 8 GB+), and shards can't just add their counts (overlaps). Many
questions tolerate ~1% error: unique viewers, daily active users, distinct IPs, unique search queries.

## Intuition
- Hash each item to a uniform random bit string. In a random bit string, the probability that it starts with `k` zeros is `2^-k`. If the **maximum run of leading zeros** you've seen is `k`, you've likely seen about `2^k` distinct items.
- One estimator is noisy, so split items into `m = 2^p` **buckets (registers)** by the first `p` bits; each register stores the max leading-zero count (+1) of the remaining bits; combine with the **harmonic mean** (reduces outlier influence) and apply bias/range corrections (HyperLogLog++ adds sparse representation and empirical bias correction).

```
add(x):  h = hash64(x);  j = h >> (64 - p);  w = h << p;  M[j] = max(M[j], leading_zeros(w) + 1)
count():  E = alpha_m * m^2 / sum(2^-M[j])   (with small/large range corrections)
merge(A,B): M[j] = max(A[j], B[j])
```

## Properties
| Property | Value |
|---|---|
| Memory | `m` registers × ~6 bits; Redis dense = 12 KB (m = 16,384) |
| Standard error | ≈ **1.04 / √m** (m = 16,384 → ≈ 0.81%) |
| Add / count | O(1) / O(m) |
| **Mergeable** | Yes: union = element-wise max (distributed aggregation, rollups) |
| Deletions | Not supported (use windowed sketches, or recompute) |
| Intersections | Not directly (inclusion-exclusion is inaccurate); use theta sketches/MinHash |
| Accuracy | Relative error is stable across cardinalities; small counts handled exactly by the sparse mode in HLL++ |

## Where it's used
- **Redis**: `PFADD`, `PFCOUNT`, `PFMERGE` (12 KB per key).
- **Databases/analytics**: Postgres (`postgresql-hll`), BigQuery `APPROX_COUNT_DISTINCT`, Presto/Trino `approx_distinct`, Spark `approx_count_distinct`, Redshift `HLL`, Druid/Pinot/ClickHouse, Elasticsearch `cardinality` aggregation.
- **Designs**: unique visitors per page per day, DAU/MAU dashboards, distinct advertisers reached, unique IPs for DDoS detection, [Top-K & ad click aggregation](../problems/ad_click_aggregration.md), [Distributed counters](../problems/distributed_counter.md).

## Design patterns
1. **Per-key daily HLLs**: store an HLL per `(page, day)`; weekly uniques = merge 7 HLLs (no re-scan). Store HLL bytes in a column or Redis.
2. **Sharded aggregation**: each worker keeps an HLL; the coordinator merges ([MapReduce](mapreduce.md) combiner-style).
3. **Streaming**: update HLLs in [Flink/Spark](flink.md) windows; emit sketches downstream rather than raw ids.
4. **Hybrid**: exact counting for small sets (sparse mode / set up to a threshold), switch to registers when large.

## Choosing the right sketch
| Question | Structure |
|---|---|
| "How many distinct items?" | **HyperLogLog** |
| "Have I seen this item?" | [Bloom filter](bloom_filter.md) |
| "How often did item X occur?" / heavy hitters | [Count-Min Sketch](count_min_sketch.md) |
| "Distinct count of A ∩ B" | Theta sketch / MinHash |
| Quantiles (p99 latency) | t-digest / DDSketch / HDR histogram |

## Pitfalls
- Using it where exactness is required (billing, compliance); expecting deletions or intersections; poor hash function (use 64-bit murmur/xxhash) or too few registers (precision too low).
- Treating the estimate as exact in dashboards (different queries disagree slightly); forgetting error bounds in the narrative.

## Interview follow-ups
- *"Count unique visitors for 1B events/day cheaply?"* HLL per page-day (12 KB each), merge for ranges; exact IDs not stored.
- *"Why is it mergeable?"* Registers hold maxima; union of sets = max of registers.
- *"What accuracy and memory?"* ~1.04/√m error; 12 KB → ~0.8%.
- *"Can you subtract / intersect?"* Not reliably; use theta sketches or keep exact data for small cohorts.

# References
* [HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm (Flajolet et al.)](https://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
* [HyperLogLog in Practice (Google, HLL++)](https://research.google/pubs/pub40671/), [Redis HyperLogLog](https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/)
