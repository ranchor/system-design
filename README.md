# System Design Notes

Revision notes for system design interviews, organised in five parts so you can study in order or jump straight to what you need.

| Part | Question it answers |
|---|---|
| [Part 0: Interview Framework](#part-0-interview-framework) | How do I run the interview and size the problem? |
| [Part 1: Key Technologies](#part-1-key-technologies) | Which building block do I pick, and when? |
| [Part 2: Core Concepts](#part-2-core-concepts) | Why do these systems behave the way they do? |
| [Part 3: Common Patterns](#part-3-common-patterns) | What are the reusable solutions to recurring problems? |
| [Part 4: Interview Problems](#part-4-system-design-interview-problems) | Can I combine all of it into a full design? |
| [Part 5: Resources](#part-5-resources) | What should I read or watch next? (books, papers, blogs, videos) |

Each note follows the same shape: **TL;DR → core ideas → diagram → trade-offs → interview follow-ups → references**. Diagrams are in `resources/**`; diagrams are SVG files that render directly on GitHub.

**Suggested path:** Part 0 → Part 1 (skim) → Part 3 → Part 4, and dip into Part 2 whenever a "why" comes up.

---

## Part 0: Interview Framework
- [Delivery framework: requirements → API → data model → high-level design → deep dives](basics/interview_framework.md)
- [Functional vs non-functional requirements checklist](basics/requirements_checklist.md)
- [Back of Envelope Estimation](basics/back_of_envelope_estimation.md)

---

## Part 1: Key Technologies
_What to reach for. Each page: what it is, when to use it, when not to, scaling model, consistency/durability guarantees, failure behaviour, and links to the concepts it relies on._

### 1.1 Databases
- [Types of Databases (landscape & how to choose)](basics/database_types.md)
- [SQL vs NoSQL (and NewSQL / distributed SQL)](basics/database_sql_vs_nosql.md)
- [Database Indexes (B-Tree vs LSM-Tree)](basics/database_indexing.md)
- [PostgreSQL & MySQL (relational deep dive)](basics/postgres_mysql.md)
- [DynamoDB](basics/dynamodb.md)
- [Cassandra](basics/cassandra.md)
- [MongoDB](basics/mongodb.md)
- [Distributed SQL deep dive: Amazon Aurora DSQL](basics/aurora_dsql.md)
- [Time Series Databases](basics/time_series_databases.md)
- [Change Data Capture (CDC)](basics/change_data_capture.md)

### 1.2 Blob / Object Storage
- [Blob / Object Storage](basics/blob_storage.md)
- [Data File Formats (Parquet, ORC, Avro)](basics/file_formats.md)

### 1.3 Search
- [Elasticsearch](basics/elasticsearch.md)
- [Vector Databases](basics/vector_databases.md) (RAG view: [RAG & Vector Search](basics/rag_vector_search.md))

### 1.4 Caching
- [Caching](basics/caching.md)
- [Redis (and Redis vs Memcached)](basics/redis_memcached.md)
- [Content Delivery Network (CDN)](basics/cdn.md)

### 1.5 Queues & Pub/Sub
- [Synchronous vs Asynchronous Communication (Queues, Pub/Sub)](basics/asynchronism.md)
- [Kafka](basics/kafka.md)
- [SQS](basics/sqs.md)
- [Kafka vs SQS](basics/kafka_vs_sqs.md)
- [Temporal (durable workflows)](basics/temporal.md)

### 1.6 Stream Processing
- [Batch Processing vs Stream Processing](basics/bp_sp.md)
- [Apache Flink](basics/flink.md)
- [Flink vs Spark](basics/flink_vs_spark.md)

### 1.7 Batch Processing
- [Apache Spark](basics/spark.md)
- [MapReduce](basics/mapreduce.md)

### 1.8 Coordination & Locks
- [Coordination Services (ZooKeeper, etcd)](basics/zookeeper.md)
- [Distributed Locking](basics/distributed_locking.md)

### 1.9 Edge & Traffic
- [Domain Name System (DNS)](basics/dns.md)
- [Proxy (Forward & Reverse)](basics/proxy.md)
- [Load Balancing](basics/load_balancing.md)
- [API Gateway](basics/api_gateway.md)

### 1.10 Service Communication
- [REST vs GraphQL vs gRPC](basics/api_styles.md)
- [Long Polling, WebSockets, SSE, Webhooks](basics/lp_ws_sse_wh.md)
- [Service Discovery & Service Mesh](basics/service_discovery.md)
- [API Design](basics/api_design.md)

### 1.11 Geospatial
- [Geohashing and Quadtrees](basics/geohash_quadtree.md) (used in [Proximity Service](problems/proximity_service.md))
- [Geospatial Indexes (S2, H3)](basics/s2_h3.md)

### 1.12 LLM Inference & Serving
- [LLM Inference Basics](basics/llm_inference_basics.md)
- [Continuous Batching & Scheduling](basics/continuous_batching_scheduling.md)
- [KV Cache, Prefix Caching & Cache-Aware Routing](basics/kv_cache_prefix_caching.md)
- [Parallelism for Serving (TP/PP/DP/EP/CP)](basics/llm_parallelism.md)
- [Speculative Decoding & Quantization](basics/speculative_decoding_quantization.md)
- [Disaggregated Serving](basics/disaggregated_serving.md)
- [Prefill/Decode Disaggregated Serving](basics/prefill_decode_disaggregation.md)
- [LLM Serving Platform: Gateway, Routing, Autoscaling](basics/llm_serving_platform.md)
- [RAG & Vector Search at Scale](basics/rag_vector_search.md)
- [Distributed Training Overview](basics/distributed_training.md)

---

## Part 2: Core Concepts
_Why they work. Theory and trade-offs; each page ends with "where it shows up" links back to Part 1._

### 2.1 Foundations
- [Scalability](basics/scalability.md)
- [Availability](basics/availability.md)
- [Reliability](basics/reliability.md)
- [OSI Model](basics/osi_model.md)
- [TCP vs UDP (and HTTP/1.1, 2, 3)](basics/tcp_udp.md)
- [Data Modeling (entities, keys, indexes, normalization)](basics/data_modeling.md)

### 2.2 Distributing Data
- [Data Partitioning / Sharding](basics/data_partitioning.md)
- [Consistent Hashing](basics/consistent_hashing.md)
- [Replication & Data Redundancy](basics/data_replication.md)
- [Quorum, Sloppy Quorum & Hinted Handoff](basics/quorum.md)

### 2.3 Consistency & Ordering
- [Consistency Models & Patterns](basics/consistency.md)
- [CAP Theorem and PACELC Theorem](basics/cap_theorem.md)
- [Time & Ordering (physical clocks, Lamport / vector / hybrid logical clocks)](basics/time_ordering.md)

### 2.4 Transactions & Messaging Guarantees
- [Transactions, ACID & Isolation Levels](basics/transactions.md)
- [Distributed Transactions (2PC, 3PC, Saga)](basics/distributed_transactions.md)
- [Idempotency](basics/idempotency.md)
- [Delivery semantics (at-most / at-least / exactly-once)](basics/delivery_semantics.md)

### 2.5 Agreement
- [Consensus (Raft, Paxos)](basics/consensus.md)
- [Leader election](basics/leader_election.md)

### 2.6 Failure Handling
- [Heartbeats & Failure Detection](basics/heartbeats.md)
- [Gossip Protocol](basics/gossip_protocol.md)
- [Resilience Patterns (Timeouts, Retries, Circuit Breaker, Bulkhead)](basics/resilience_patterns.md)
- [Checksums](basics/checksums.md)

### 2.7 Data Structures & Storage Internals
- [Bloom Filters](basics/bloom_filter.md)
- [Count-Min Sketch](basics/count_min_sketch.md)
- [HyperLogLog](basics/hyperloglog.md)
- [Merkle Trees](basics/merkle_tree.md)
- [Write-ahead log, LSM compaction](basics/wal_lsm.md)

### 2.8 Security & Observability
- [Security](basics/security.md)
    - [AuthN and AuthZ (sessions, JWT, OAuth2/OIDC, RBAC/ReBAC)](basics/authn_authz.md)
- [Monitoring & Observability](basics/monitoring.md)

---

## Part 3: Common Patterns
_Reusable solutions that combine technologies and concepts. Format: problem it solves → approach → technologies used → trade-offs → example problems._

### 3.1 Scaling & Performance
- [Scaling reads (replicas, caching, CDN)](basics/scaling_reads.md)
- [Scaling writes (sharding, batching, write buffering via queues)](basics/scaling_writes.md)
- [Hot keys / hot partitions](basics/hot_keys.md)
- [Fan-out on write vs fan-out on read](basics/fanout_write_vs_read.md)
- [Rate limiting, backpressure & load shedding](basics/backpressure_load_shedding.md) (see also [Resilience Patterns](basics/resilience_patterns.md), [Rate Limiter](problems/rate_limiter.md))
- [Multi-region / geo-distribution](basics/multi_region.md)

### 3.2 Correctness & Reliability
- [Handling contention (optimistic vs pessimistic locking, distributed locks, queue serialisation)](basics/contention.md) (see also [Transactions](basics/transactions.md), [Distributed Locking](basics/distributed_locking.md))
- [Multi-step processes (Saga, orchestration vs choreography, workflows)](basics/multi_step_processes.md) (see also [Distributed Transactions](basics/distributed_transactions.md))
- [Reliable messaging (outbox, retries, DLQ, deduplication)](basics/reliable_messaging.md) (see also [Idempotency](basics/idempotency.md))
- [Long-running tasks (async jobs, workers, scheduling)](basics/long_running_tasks.md) (see also [Job Scheduler](problems/distributed_job_scheduler.md))
- [Real-time updates (polling vs SSE vs WebSockets vs pub/sub)](basics/realtime_updates.md) (see also [Long Polling, WebSockets, SSE](basics/lp_ws_sse_wh.md))
- [Large blobs (presigned URLs, chunked / resumable upload)](basics/large_blobs.md) (see also [Blob Storage](basics/blob_storage.md))
- [Counting & dedup at scale (Bloom, CMS, HLL)](basics/counting_dedup_at_scale.md)

### 3.3 Architecture Patterns
- [Architecture Patterns (SOA, microservices, event-driven)](basics/architecture_patterns.md)
- [Event sourcing, CQRS, change data capture](basics/event_sourcing_cqrs_cdc.md)
- [Design Principles (SOLID)](https://blog.bytebytego.com/p/mastering-design-principles-solid)

---

## Part 4: System Design Interview Problems
Each problem lists the **building blocks** (Part 1), **concepts** (Part 2) and **patterns** (Part 3) it uses. Template for new problems: [template.md](template.md)

### Depth Oriented Problems
- [Design Consistent Hashing](problems/consistent_hashing_design.md) (concept: [Consistent Hashing](basics/consistent_hashing.md))
- **[Design Distributed Key-Value Store](problems/key_value_store.md)**
- [Design Content Delivery Network (CDN)](problems/cdn_design.md) (concept: [CDN](basics/cdn.md))
- **[Design Distributed Cache](problems/distributed_cache.md)**
- [Design Distributed Message Queue](problems/distributed_message_queue.md)
- **[Design a File Storage System / GFS / HDFS](problems/file_storage_gfs_hdfs.md)**
- **[Design an Object Storage System / S3](problems/object_storage_s3.md)** (concept: [Blob Storage](basics/blob_storage.md))
- **[Design Distributed Locking Service](problems/distributed_locking_service.md)** (concept: [Distributed Locking](basics/distributed_locking.md))
- **[Design Distributed Counters](problems/distributed_counter.md)**
- [Design Distributed Logging](problems/distributed_logging.md)
- [Design Metrics Monitoring and Alerting System](problems/distributed_metrics.md)
- [Design Metrics Aggregation](problems/metrics_aggregration.md)

### Networking and API Systems
- [Design API Rate Limiter](problems/rate_limiter.md)
- [Design a Unique Id Generator & Key Generator](problems/unique_id_generator.md)
- [Design a Url Shortener or Design a tiny url](problems/url_shortener.md)
- [Design notification system](problems/notification_system.md)
- [Design Job Scheduler/Distributed Task Scheduler](problems/distributed_job_scheduler.md)

### Search and Discovery Systems
- [Design a Search AutoComplete system](problems/typehead_suggestion.md)
- [Design a Google Search](problems/distributed_search.md)
- [Design Web Crawler](problems/web_crawler.md)

### Streaming/Continuous Data Flow
- [Design Ad-Click Aggregation](problems/ad_click_aggregration.md)
- [Design Trending Topics (Twitter)/Top K/Heavy Hitters](problems/top_k.md)
- [Design Google Docs](problems/google_docs.md)

### Location-Based Systems
- [Design Location Based Service like Yelp](problems/proximity_service.md)
- [Design Nearby Friends](problems/nearby_friends.md)
- [Design Uber](problems/uber.md)
- [Design Google Maps](problems/google_maps.md)
- [Design a Local Delivery Service](problems/local_delivery_service.md)

### Social Media and Communication Systems
- [Design Chat System/messenger](problems/chat_system.md)
- [Design News Feed](problems/newsfeed_system.md)
- [Design Live Comments](problems/live_comment.md)
- [Design Instagram](problems/instagram.md)
- [Design YouTube](problems/youtube.md)
- [Design Google Photos/Photos Storage](problems/google_photos.md)
- [Design Privacy Controls](problems/privacy_controls.md)
- [Design a Localization Feature](problems/localization_feature.md)

### E-commerce and Reservation Systems
- [Design a Ecommerce Website](problems/ecommerce.md)
- [Design a Shopping Cart](problems/shopping_cart.md)
- [Design a Inventory Management System](problems/inventory_management.md)
- [Design a Ticketmaster](problems/ticketmaster.md)
- [Design a Hotel Reservation System](problems/hotel_reservation_system.md)
- [Design Price Alert Online System](problems/price_tracker.md)
- [Design an Online Auction / Bidding System](problems/bidding_system.md)

### Financial Systems
- [Design a Payment System](problems/payment_gateway.md)
- [Design a Digital Wallet](problems/digital_wallet.md)
- [Design a Payment Webhook](problems/payment_webhook.md)
- [Design a Ledger System](problems/distributed_ledger.md)

### User Requests Traffic
- [Design LeetCode](problems/leetcode.md)
- [Design Dropbox/Google Drive/File Sharing Service](problems/dropbox.md)

## Part 5: Resources
_Where to go deeper. Papers are paired with the note they back up._

### 5.1 Books & Book Notes
- [Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/B08VL1BLHB/) — [my notes](ddia/readme.md)
- [System Design Interview – An insider's guide](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF/)

### 5.2 Must-Read Distributed Systems Papers
| Paper | Pairs with |
|---|---|
| [Paxos: The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) | [Consensus](basics/consensus.md) |
| [The Chubby lock service](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf) | [Distributed Locking](basics/distributed_locking.md), [Coordination Services](basics/zookeeper.md) |
| [ZooKeeper: Wait-free coordination](https://www.usenix.org/legacy/event/usenix10/tech/full_papers/Hunt.pdf) | [Coordination Services](basics/zookeeper.md) |
| [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) | [Quorum](basics/quorum.md), [Consistent Hashing](basics/consistent_hashing.md), [Replication](basics/data_replication.md) |
| [Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) | [Database Indexes (LSM)](basics/database_indexing.md), [SQL vs NoSQL](basics/database_sql_vs_nosql.md) |
| [The Log-Structured Merge-Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf) | [Database Indexes (LSM)](basics/database_indexing.md) |
| [Spanner](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) | [Distributed Transactions](basics/distributed_transactions.md), [Consistency](basics/consistency.md) |
| [The Google File System](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf) | [Blob / Object Storage](basics/blob_storage.md) |
| [MapReduce](https://research.google.com/archive/mapreduce-osdi04.pdf) | [Batch vs Stream Processing](basics/bp_sp.md) |
| [Kafka: a Distributed Messaging System for Log Processing](https://notes.stephenholiday.com/Kafka.pdf) | [Queues & Pub/Sub](basics/asynchronism.md) |

### 5.3 Blogs & GitHub Repos to Follow
- [awesome-system-design-resources](https://github.com/ashishps1/awesome-system-design-resources/blob/main/README.md)
- [karanpratapsingh/system-design](https://github.com/karanpratapsingh/system-design)
- [ByteByteGo system-design-101](https://github.com/ByteByteGoHq/system-design-101)
- [DDIA notes (keyvanakbary)](https://github.com/keyvanakbary/learning-notes/blob/master/books/designing-data-intensive-applications.md)
- [Alex Xu book notes (preslavmihaylov)](https://github.com/preslavmihaylov/booknotes/tree/master/system-design/system-design-interview)
- [highscalability](https://highscalability.com/)
- [systemdesign.one cheatsheet](https://systemdesign.one/system-design-interview-cheatsheet/)
- [Jordan System Design Basics slides](https://drive.google.com/drive/folders/1ChodcbMZ4KqS9WP9gin4sLVdCsgD3uoE)

### 5.4 YouTube Channels
- [ByteByteGo](https://www.youtube.com/@ByteByteGo)
- [System Design Interview](https://www.youtube.com/@SystemDesignInterview)
- [codeKarle](https://www.youtube.com/@codeKarle)
- [System Design Fight Club](https://www.youtube.com/@SDFC)
- [Jordan has no life](https://www.youtube.com/@jordanhasnolife5163/playlists)
