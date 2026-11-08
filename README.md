# System Design Notes

Revision notes for system design interviews. Each concept note follows the same shape: **TL;DR → core ideas →
diagram → trade-offs → interview follow-ups → references**. Diagrams are in `resources/**`; each `.svg` has a
matching `.excalidraw` source you can open and edit at [excalidraw.com](https://excalidraw.com).

Items marked _(planned)_ don't have notes yet.

## System Design Key Concepts

### Chapter 1: Foundations, Networking & Traffic
- [Scalability](basics/scalability.md)
- [Availability](basics/availability.md)
- [Reliability](basics/reliability.md)
- [Consistency Models & Patterns](basics/consistency.md)
- [CAP Theorem and PACELC Theorem](basics/cap_theorem.md)
- [Back of Envelope Estimation](basics/back_of_envelope_estimation.md)
- [OSI Model](basics/osi_model.md)
- [TCP vs UDP (and HTTP/1.1, 2, 3)](basics/tcp_udp.md)
- [Domain Name System (DNS)](basics/dns.md)
- [Content Delivery Network (CDN)](basics/cdn.md)
- [Proxy (Forward & Reverse)](basics/proxy.md)
- [Load Balancing](basics/load_balancing.md)
- [API Gateway](basics/api_gateway.md)
- [REST vs GraphQL vs gRPC](basics/api_styles.md)
- [Long Polling, WebSockets, SSE, Webhooks](basics/lp_ws_sse_wh.md)
- [Caching](basics/caching.md)
    - [Distributed Caching](basics/caching.md#distributed-cache)
    - [Memcached vs Redis](basics/redis_memcached.md)
- [Consistent Hashing](basics/consistent_hashing.md)
- [Security](basics/security.md)
    - [AuthN and AuthZ (sessions, JWT, OAuth2/OIDC, RBAC/ReBAC)](basics/authn_authz.md)
- [Monitoring & Observability](basics/monitoring.md)

### Chapter 2: Data & Distributed Systems
- [Types of Databases](basics/database_types.md)
- **[SQL vs NoSQL](basics/database_sql_vs_nosql.md)**
- [Database Indexes (B-Tree vs LSM-Tree)](basics/database_indexing.md)
- **[Replication & Data Redundancy](basics/data_replication.md)**
- [Data Partitioning / Sharding](basics/data_partitioning.md)
- [Quorum, Sloppy Quorum & Hinted Handoff](basics/quorum.md)
- **[Transactions, ACID & Isolation Levels](basics/transactions.md)**
- [Distributed Transactions (2PC, 3PC, Saga)](basics/distributed_transactions.md)
- [Consensus (Raft, Paxos)](basics/consensus.md)
- **[Coordination Services (ZooKeeper, etcd)](basics/zookeeper.md)**
- [Distributed Locking](basics/distributed_locking.md)
- [Gossip Protocol](basics/gossip_protocol.md)
- [Heartbeats & Failure Detection](basics/heartbeats.md)
- [Checksums](basics/checksums.md)
- [Idempotency](basics/idempotency.md)
- [Service Discovery & Service Mesh](basics/service_discovery.md)
- [Resilience Patterns (Timeouts, Retries, Circuit Breaker, Bulkhead)](basics/resilience_patterns.md)
- **[Batch Processing vs Stream Processing](basics/bp_sp.md)**
- [Synchronous vs Asynchronous Communication (Queues, Pub/Sub)](basics/asynchronism.md)
- **[Blob / Object Storage](basics/blob_storage.md)**
- [Data File Formats (Parquet, ORC, Avro)](basics/file_formats.md)
- [Bloom Filters](basics/bloom_filter.md)
- [Count-Min Sketch](basics/count_min_sketch.md)

### Chapter 3: Technologies Deep Dive
- [Apache Flink](basics/flink.md)
- Message Brokers _(planned; basics in [Asynchronism](basics/asynchronism.md))_
- Kafka _(planned)_
- SQS _(planned)_
- Kafka vs SQS _(planned; summary in [Asynchronism](basics/asynchronism.md#interview-follow-ups))_
- Apache Spark _(planned)_
- Flink vs Spark _(planned)_
- MongoDB _(planned)_
- Cassandra _(planned)_
- Elasticsearch _(planned)_
- Geohashing and Quadtrees _(planned; used in [Proximity Service](problems/proximity_service.md))_
- Geospatial Indexes _(planned)_

### Chapter 4: Architecture & API Design
- **[Architecture Patterns](basics/architecture_patterns.md)**
- [Design Principles (SOLID)](https://blog.bytebytego.com/p/mastering-design-principles-solid)
- [API Design](basics/api_design.md)

## System Design Interview Problems
Template for new problems: [template.md](template.md)

### Depth Oriented Problems
- Design Consistent Hashing _(planned; concept in [Consistent Hashing](basics/consistent_hashing.md))_
- **[Design Distributed Key-Value Store](problems/key_value_store.md)**
- Design Content Delivery Network (CDN) _(planned; concept in [CDN](basics/cdn.md))_
- **[Design Distributed Cache](problems/distributed_cache.md)**
- [Design Distributed Message Queue](problems/distributed_message_queue.md)
- **Design a File Storage System / GFS / HDFS** _(planned)_
- **Design an Object Storage System / S3** _(planned; concept in [Blob Storage](basics/blob_storage.md))_
- **Design Distributed Locking Service** _(planned; concept in [Distributed Locking](basics/distributed_locking.md))_
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
- Design a Local Delivery Service _(planned)_

### Social Media and Communication Systems
- [Design Chat System/messenger](problems/chat_system.md)
- [Design News Feed](problems/newsfeed_system.md)
- [Design Live Comments](problems/live_comment.md)
- [Design Instagram](problems/instagram.md)
- [Design YouTube](problems/youtube.md)
- Design Google Photos/Photos Storage _(planned)_
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
- Design a Digital Wallet _(planned)_
- [Design a Payment Webhook](problems/payment_webhook.md)
- [Design a Ledger System](problems/distributed_ledger.md)

### User Requests Traffic
- [Design LeetCode](problems/leetcode.md)
- [Design Dropbox/Google Drive/File Sharing Service](problems/dropbox.md)

## 📖 Book Notes
- [Designing Data-Intensive Applications](ddia/readme.md)

## 🗞️ Must-Read Distributed Systems Papers
- [Paxos: The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)
- [MapReduce: Simplified Data Processing on Large Clusters](https://research.google.com/archive/mapreduce-osdi04.pdf)
- [The Google File System](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)
- [Dynamo: Amazon’s Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Kafka: a Distributed Messaging System for Log Processing](https://notes.stephenholiday.com/Kafka.pdf)
- [Spanner: Google’s Globally-Distributed Database](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)
- [Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
- [ZooKeeper: Wait-free coordination for Internet-scale systems](https://www.usenix.org/legacy/event/usenix10/tech/full_papers/Hunt.pdf)
- [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
- [The Chubby lock service for loosely-coupled distributed systems](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf)

## 📚 Books
- [Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/B08VL1BLHB/)
- [System Design Interview – An insider's guide](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF/)

## Blog Posts/Github Links to Follows
- [awesome-system-design-resources](https://github.com/ashishps1/awesome-system-design-resources/blob/main/README.md)
- [Jordan System Design Basics PowerPoint](https://drive.google.com/drive/folders/1ChodcbMZ4KqS9WP9gin4sLVdCsgD3uoE)
- https://github.com/karanpratapsingh/system-design
- [highscalability](https://highscalability.com/)
- [ByteByteGoHq](https://github.com/ByteByteGoHq/system-design-101)
- [designing-data-intensive-applications notes](https://github.com/keyvanakbary/learning-notes/blob/master/books/designing-data-intensive-applications.md)
- [Alex Wu Books Notes](https://github.com/preslavmihaylov/booknotes/tree/master/system-design/system-design-interview)
- [systemdesign one](https://systemdesign.one/system-design-interview-cheatsheet/)

## YouTube Channels
- [ByteByteGo](https://www.youtube.com/@ByteByteGo)
- [System Design Interview](https://www.youtube.com/@SystemDesignInterview)
- [codeKarle](https://www.youtube.com/@codeKarle)
- [System Design Fight Club](https://www.youtube.com/@SDFC)
- [Jordan has no life](https://www.youtube.com/@jordanhasnolife5163/playlists)