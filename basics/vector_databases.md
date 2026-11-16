# Vector Databases

> **TL;DR** — A **vector database** stores high-dimensional **embeddings** (plus metadata) and answers **nearest-neighbour (similarity) queries** fast. Exact kNN is O(N·d), so production systems use **approximate nearest neighbour (ANN)** indexes (HNSW, IVF, product quantization, DiskANN-style) trading a little **recall** for orders of magnitude lower latency.
> Use for semantic search, recommendations, RAG retrieval, deduplication, image/audio similarity. Key design questions: **index type, recall vs latency vs memory, filtering, sharding/replication, freshness, embedding versioning.** Options: dedicated engines (Pinecone, Milvus, Weaviate, Qdrant), extensions to existing stores (**pgvector**, Elasticsearch/OpenSearch kNN, Redis), libraries (FAISS).
> The RAG pipeline view is in [RAG & vector search](rag_vector_search.md); this page is the database/storage view. See also [Elasticsearch](elasticsearch.md), [PostgreSQL](postgres_mysql.md), [Data partitioning](data_partitioning.md).

## Data model and queries
- **Record** = `id` + `vector` (e.g. 384-3072 floats) + **metadata/payload** (tags, tenant, timestamps, source text).
- **Similarity metrics**: cosine, dot product, L2 (Euclidean). Normalize vectors if using cosine/dot; the metric must match how the embedding model was trained.
- **Queries**: top-k ANN, **filtered ANN** (metadata predicates), **hybrid** (vector + BM25 keyword, fused by reciprocal rank), range/threshold search, batch queries, and sometimes multi-vector (late-interaction) retrieval.
- Embeddings come from a model; **same model/version for indexing and querying**. Changing models means re-embedding and re-indexing (blue/green index).

## Index families
| Index | How it works | Strengths | Weaknesses |
|---|---|---|---|
| **Flat / brute force** | Scan everything (often on GPU) | Exact, simple; fine up to ~10^5-10^7 | Linear cost |
| **HNSW** | Multi-layer small-world graph; greedy descent + beam (`efSearch`) | Best recall/latency tradeoff, incremental inserts | High RAM (vectors + links), costly deletes/rebuilds, build time |
| **IVF (inverted file)** | k-means partitions; probe `nprobe` nearest cells | Lower memory, tunable | Needs training; recall drops at boundaries |
| **PQ / SQ / binary quantization** | Compress vectors into short codes or lower-precision | 4-32x memory savings | Recall loss; often re-rank with full vectors |
| **IVF-PQ** | Partition + compress | Billion-scale in RAM | Tuning complexity |
| **Disk-based graph (DiskANN-style)** | Graph on SSD, compressed copies in RAM | Billion-scale cheaply | Higher latency, SSD bandwidth |
| **LSH** | Hash similar items together | Simple, streaming-friendly | Generally lower recall than graphs |

Rule of thumb: pick by **dataset size × RAM budget × recall target × update rate**. Measure recall@k against exact search on your own data.

## Distributed architecture
- **Sharding**: split vectors across nodes (hash on id, or by tenant/collection). A query **fans out** to all shards and merges top-k, so tail latency = slowest shard; more shards ≠ more speed for a single query ([Partitioning](data_partitioning.md)).
- **Replication**: replicas for QPS and availability; leader-based or leaderless depending on engine ([Replication](data_replication.md)).
- **Segments / LSM-like writes**: writes land in a mutable buffer/WAL, flush to **immutable index segments**, background **compaction/merge** rebuilds larger indexes; deletes via tombstones ([WAL/LSM](wal_lsm.md)). Search covers buffer + segments, so freshness is near-real-time but not instant.
- **Separation of storage and compute**: segments in object storage, stateless query nodes with caches (serverless vector DBs) trade cold latency for cost ([Blob storage](blob_storage.md)).
- **Filtering**: pre-filter (restrict candidates, can break graph connectivity), post-filter (can return too few), or integrated filtered traversal; very selective filters may switch to brute force on the candidate set. Per-tenant partitions avoid filters entirely.
- **Consistency**: usually eventual/near-real-time visibility; no multi-document transactions; if the source of truth is elsewhere, sync with **CDC** ([CDC](change_data_capture.md)).

## Choosing a store
| Option | Good when | Watch out |
|---|---|---|
| **pgvector** (PostgreSQL) | Already on Postgres, up to millions of vectors, need joins/transactions with relational data | Memory/index build limits at larger scale; HNSW/IVFFlat tuning |
| **Elasticsearch/OpenSearch kNN** | Need hybrid text + vector + filters in one engine | Heap/memory pressure; segment merges |
| **Dedicated vector DB** (Milvus, Qdrant, Weaviate, Pinecone) | Large scale, many collections, advanced filtering, managed ops | Another system to sync and operate |
| **FAISS / in-process library** | Offline/batch, embedded in a service | You build persistence, sharding, updates |
| **Redis vector search** | Low-latency, in-memory, moderate size | RAM cost |

## Capacity math
`memory ≈ N × d × bytes_per_dim × overhead`. Example: 100M vectors × 768 dims × 4 B ≈ 307 GB raw before index overhead and replicas; with 8-bit scalar quantization ~77 GB; with PQ far less. HNSW adds graph links (`M` × 2 × id size per vector).

## Pitfalls
- Mixing embedding models in one index; ignoring recall measurement; huge `efSearch` driving p99; filters that destroy recall; heavy update/delete churn without compaction; storing large payloads in the index instead of object storage; no ACL enforcement at query time; assuming "vector DB" replaces the source-of-truth database; forgetting re-embedding cost when the model changes.

## Interview follow-ups
- *"Why not exact search?"* O(N) per query; ANN gives ~ms latency at 95-99% recall.
- *"HNSW vs IVF-PQ?"* HNSW: fastest/highest recall but RAM-heavy; IVF-PQ: compact, slightly lower recall.
- *"How do you keep it in sync with the primary DB?"* CDC → embedding workers → upsert with versioned ids; backfill on model change.
- *"How do you do filtered search at scale?"* Partition by tenant, filtered traversal, or hybrid with an inverted index.

# References
* [Efficient and robust approximate nearest neighbor search using HNSW graphs](https://arxiv.org/abs/1603.09320)
* [FAISS](https://github.com/facebookresearch/faiss) · [The Faiss library](https://arxiv.org/abs/2401.08281)
* [pgvector](https://github.com/pgvector/pgvector)
* [DiskANN (Microsoft Research)](https://github.com/microsoft/DiskANN)
* [Hello Interview: Vector databases](https://www.hellointerview.com/learn/system-design/deep-dives/vector-databases)
