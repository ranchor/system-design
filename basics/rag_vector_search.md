# RAG & Vector Search at Scale

> **TL;DR** — **Retrieval-Augmented Generation** grounds an LLM in your data: **ingest** (chunk → embed → index), then at query time **embed the query, retrieve top-k similar chunks** (often hybrid vector + keyword, then rerank), and **stuff them into the prompt**.
> The distributed-systems core is the **vector index**: exact search doesn't scale, so use **approximate nearest neighbor (ANN)** (HNSW graphs, IVF + product quantization) with sharding, replication, metadata filtering and freshness handling. Quality is a retrieval problem first, generation second.
> See [Elasticsearch](elasticsearch.md), [Data partitioning](data_partitioning.md), [KV cache & prefix caching](kv_cache_prefix_caching.md), [LLM inference basics](llm_inference_basics.md), [Event sourcing/CDC](event_sourcing_cqrs_cdc.md).

## Pipeline
```
Ingest:  sources → parse → chunk → embed (batch GPU) → upsert (vector + metadata + text) → index
Query:   question → (rewrite) → embed → ANN top-k (+ filters, + BM25) → rerank → build prompt → LLM → answer + citations
```
- **Chunking**: by structure (headings, paragraphs) with overlap; chunk size trades precision (small) vs context (large). Store parent/section IDs for expansion.
- **Embeddings**: model choice fixes dimension and quality; queries and documents must use the **same model/version**. Changing models means **re-embedding everything** (dual-index migration).
- **Hybrid search**: combine dense vectors (semantics) with BM25/keyword (exact terms, IDs) via reciprocal-rank fusion; Elasticsearch/OpenSearch and many vector DBs support both ([Elasticsearch](elasticsearch.md)).
- **Reranking**: a cross-encoder rescoring the top 50-100 improves precision at extra latency/cost.
- **Prompt assembly**: dedupe, order by relevance, respect context budget, add citations; long shared prefixes (system prompt, instructions) benefit from prefix caching ([KV cache](kv_cache_prefix_caching.md)).

## ANN indexes
| Index | Idea | Trade-offs |
|---|---|---|
| **Flat (exact)** | Compare against all vectors | Perfect recall; O(N); fine up to ~millions with GPUs |
| **HNSW** | Layered proximity graph; greedy search from top layer | High recall/low latency; **memory-heavy** (graph + vectors); inserts OK, deletes awkward |
| **IVF** | Cluster vectors (k-means); search nearest `nprobe` clusters | Smaller memory; recall depends on `nprobe`; needs training |
| **PQ / scalar quantization** | Compress vectors into codes | Large memory savings, some recall loss; often combined as IVF-PQ |
| **DiskANN-style** | Graph on SSD with compressed in-memory data | Billion-scale on a single node/cheaper RAM; higher latency |
FAISS is the reference library for flat, IVF, PQ and HNSW indexes (CPU/GPU) and scales to billions of vectors in memory on a server; managed and open-source vector databases wrap such indexes with APIs, filtering and clustering.

Key knobs: HNSW `M`, `efConstruction`, `efSearch`; IVF `nlist`, `nprobe`; distance metric (cosine/dot/L2, normalize consistently). Measure **recall@k vs latency** on your data.

## Scaling the index (distributed)
- **Sharding**: partition vectors across nodes (by hash/ID, or tenant/collection); queries **scatter to all shards** and merge top-k (tail latency = slowest shard) ([Data partitioning](data_partitioning.md)). Clustering-based partitioning can reduce fan-out at some recall cost.
- **Replication**: replicas for read throughput and availability; build indexes offline and swap (immutable segments + background merge, LSM-like, [WAL/LSM](wal_lsm.md)).
- **Filtering**: metadata/ACL filters interact badly with ANN (pre-filter hurts graph traversal; post-filter loses results); engines use filtered traversal, partitions per tenant, or fall back to brute force on small candidate sets.
- **Freshness**: new documents appear after segment refresh; deletes via tombstones; stream updates from source DB via **CDC** into the index and embedding pipeline ([CDC](event_sourcing_cqrs_cdc.md)); embedding is a GPU batch/queue workload ([Reliable messaging](reliable_messaging.md)).
- **Multi-tenancy & security**: per-tenant namespaces or filters; **enforce document ACLs at retrieval time**, never rely on the LLM to hide data.
- **Cost**: RAM dominates (vectors × dimension × bytes × replicas); use quantization, tiered storage, and smaller dimensions (Matryoshka-style embeddings) where quality allows.

## Latency budget (typical)
embed query (10-50 ms) → ANN search (5-50 ms) → rerank (50-300 ms) → LLM TTFT (hundreds of ms+) → stream. Cache query embeddings and frequent retrievals; run retrieval in parallel with safety checks.

## Evaluation and quality
- Offline: recall@k / MRR / nDCG on labelled queries; end-to-end: faithfulness (answer supported by context), citation accuracy, refusal when no evidence.
- Online: user feedback, no-answer rates, retrieval-hit logging. Track drift when content or embedding model changes.
- Common fixes: better chunking, hybrid search, query rewriting, reranking, metadata filters, more context vs less noise.

## Beyond basic RAG
- **Agentic / multi-step retrieval** (iterate, decompose queries, call tools), **long-context** models (fewer chunks but higher cost; prefix-cache static corpora), **GraphRAG** (entity/relationship graph), **semantic cache** of Q&A ([Serving platform](llm_serving_platform.md)).

## Pitfalls
- Mixing embedding models/versions in one index; no ACL enforcement at retrieval; chunks too big/small; relying on vector-only search for IDs, codes and rare terms; ignoring deletes and freshness; testing only generation, not retrieval; sharded fan-out causing p99 spikes; embedding pipeline blocking ingestion; prompt injection through retrieved documents (treat retrieved text as untrusted).

## Interview follow-ups
- *"Why ANN instead of exact?"* Exact kNN is O(N·d); ANN trades a little recall for orders of magnitude lower latency.
- *"HNSW vs IVF-PQ?"* HNSW: fast, high recall, RAM-heavy; IVF-PQ: compact, slightly lower recall, trainable.
- *"How do you keep an index fresh?"* CDC/streaming ingest, immutable segments with merge, tombstones, versioned embeddings.
- *"How do you enforce permissions?"* Filter/ACL at query time in the retriever; per-tenant partitions.

See also [Vector databases](vector_databases.md) for index and storage design, and [CDC](change_data_capture.md) for keeping indexes in sync.

# References
* [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)
* [Efficient and robust approximate nearest neighbor search using HNSW graphs](https://arxiv.org/abs/1603.09320)
* [FAISS (facebookresearch/faiss)](https://github.com/facebookresearch/faiss)
* [The Faiss library (Douze et al., 2024)](https://arxiv.org/abs/2401.08281)
