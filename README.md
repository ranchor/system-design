## System Design Key Concepts

### Chapter1 

- [Domain Name System (DNS)](https://github.com/ranchor/system-design/blob/main/basics/dns.md)
- [Content Delivery Network (CDN)](https://github.com/ranchor/system-design/blob/main/basics/cdn.md)
- [Load Balancing](https://github.com/ranchor/system-design/blob/main/basics/load_balancing.md)
- [API Gateway](https://github.com/ranchor/system-design/blob/main/basics/api_gateway.md)
- [Caching](https://github.com/ranchor/system-design/blob/main/basics/caching.md)
- [Distributed Caching](https://redis.com/glossary/distributed-caching/)
- [Memcached vs Redis](https://github.com/ranchor/system-design/blob/main/basics/redis_memcached.md)
- [Availability]()
- [Consistency](https://systemdesign.one/consistency-patterns/)
- [Reliability]()
- [Scalability](https://newsletter.ashishps.com/p/system-design-vertical-vs-horizontal-scaling)
- [CAP Theorem and PACELC Theorem](https://github.com/ranchor/system-design/blob/main/basics/cap_theorem.md)
- [Consistent Hashing](https://github.com/ranchor/system-design/blob/main/basics/consistent_hashing.md)
- [Security](https://github.com/ranchor/system-design/blob/main/basics/security.md)
    - [AuthN and AuthZ]()
- [Monitoring](https://github.com/ranchor/system-design/blob/main/basics/monitoring.md)
- [REST, GraphQL, gRPC]()
- [Long Polling,WebSockets,SSE](https://github.com/ranchor/system-design/blob/main/basics/lp_ws_sse_wh.md)
- [OSI Model](https://github.com/ranchor/system-design/blob/main/basics/osi_model.md)
- [TCP vs UDP]()
- [Back of Envelope Estimation](https://github.com/ranchor/system-design/blob/main/basics/back_of_envelope_estimation.md)


### Chapter2
- [Databases](https://newsletter.ashishps.com/p/15-types-of-databases)
- **[SQL vs NoSQL](https://github.com/ranchor/system-design/blob/main/basics/database_sql_vs_nosql.md)**
- [Database Index](https://newsletter.ashishps.com/p/a-detailed-guide-on-database-indexes)
- **[Data Replication](https://redis.com/blog/what-is-data-replication/)**
- **[Data Redundancy](https://www.egnyte.com/guides/governance/data-redundancy)**
- [Data Partitioning](https://github.com/ranchor/system-design/blob/main/basics/data_partitioning.md)
- **[Transactions](https://github.com/ranchor/system-design/blob/main/basics/transactions.md)**
- [Distributed Transactions](https://github.com/ranchor/system-design/blob/main/basics/transactions.md)
- **[Batch Processing vs Stream Processing](https://newsletter.ashishps.com/p/d9442268-03d8-4f55-a103-7a3d4fb54661)**
- [Synchronous vs asynchronous communications](https://newsletter.ashishps.com/p/aec1cebf-6060-45a7-8e00-47364ca70761)
- [Gossip Protocol](https://github.com/ranchor/system-design/blob/main/basics/gossip_protocol.md)
- **[Coordination Services]()**
- **[Blob Storage]()**
- [Idempotency](https://blog.dreamfactory.com/what-is-idempotency/)
- [Parquet]()
- [Bloom Filters](https://www.enjoyalgorithms.com/blog/bloom-filter)
- [Consensus Algorithms](https://medium.com/@sourabhatta1819/consensus-in-distributed-system-ac79f8ba2b8c)
- [Checksums](https://newsletter.ashishps.com/p/what-are-checksums)
- [HeartBeats](https://newsletter.ashishps.com/p/heartbeats-in-distributed-systems)
- [Circuit Breaker](https://medium.com/geekculture/design-patterns-for-microservices-circuit-breaker-pattern-276249ffab33)


### Chapter3
- [Message Brokers]()
- [Message Queues]()
- [Publish-Subscribe]()
- [Kafka]()
- [SQS]()
- [Kafka vs SQS]()
- [Apache Flink]()
- [Apache Spark]()
- [Fink vs Spark]()
- [MongoDB]()
- [Cascandra]()
- [Elastic Search]()
- [Geohashing and Quadtrees]()
- [Geospatial Indexs]()


### Chapter4 
- **[Architecture Patterns](https://github.com/ranchor/system-design/blob/main/basics/system_design_patterns.md)**
- [Design Principles](https://blog.bytebytego.com/p/mastering-design-principles-solid)
- [API Design]()

## System Design Interview Problems

### Depth Oriented Problems
- [Design Consistent Hashing]()
- **[Design Distributed Key-Value Store](https://github.com/ranchor/system-design/blob/main/problems/key_value_store.md)**
- [Design API Rate Limiter](https://github.com/ranchor/system-design/blob/main/problems/rate_limiter.md)
- [Design a Unique Id Generator & Key Generator](https://github.com/ranchor/system-design/blob/main/problems/unique_id_generator.md)
- [Design a Url Shortener or Design a tiny url](https://github.com/ranchor/system-design/blob/main/problems/url_shortener.md)
- [Design Content Delivery Network (CDN)]()
- **[Design Distributed Cache](https://github.com/ranchor/system-design/blob/main/problems/distributed_cache.md)**
- [Design Distributed Message Queue](https://github.com/ranchor/system-design/blob/main/problems/distributed_message_queue.md)
- **[Design a File Storage System / GFS / HDFS]()**
- **[Design a object storage system / S3 / Dropbox]()**
- **[Design Distributed Locking Service]()**
- **[Design Distributed Counters]()**

### Batch Jobs
- [Design Job Scheduler/Distributed Task Scheduler](https://github.com/ranchor/system-design/blob/main/problems/distributed_job_scheduler.md)
- [Design Web Crawler](https://github.com/ranchor/system-design/blob/main/problems/web_crawler.md)
- **[Design notification system]()**
- [Design LeetCode](https://github.com/ranchor/system-design/blob/main/problems/leetcode.md)
- [Design Price Alert Online System](https://github.com/ranchor/system-design/blob/main/problems/price_tracker.md)


### Streaming / Continuous Data Flow
- [Design YouTube](https://github.com/ranchor/system-design/blob/main/problems/youtube.md)
- [Design Chat System/messenger](https://github.com/ranchor/system-design/blob/main/problems/chat_system.md)
- [Design News Feed](https://github.com/ranchor/system-design/blob/main/problems/newsfeed_system.md)
- [Design Location Based Service like Yelp](https://github.com/ranchor/system-design/blob/main/problems/proximity_service.md)
- [Design Uber](https://github.com/ranchor/system-design/blob/main/problems/uber.md)
- [Design Google Maps]()
- [Design Metrics Monitoring and Alerting System]()
- [Design Ad-Click Aggregration]()
- [Design Trending Topics (Twitter)/Top K/Heavy Hitters]()
- [Design Live Comments]()

### User Requests Traffic
- [Design a Ticketmaster]()
- [Design Dropbox]()
- [Design Instagram]()
- [Design a Search AutComplete system]()
- [Design a Google Search]()
- [Design a Local Delivery]()
- [Design a payment system]()



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


## YouTube Channels
- [ByteByteGo](https://www.youtube.com/@ByteByteGo)
- [System Design Interview](https://www.youtube.com/@SystemDesignInterview)
- [codeKarle](https://www.youtube.com/@codeKarle)
- [System Design Fight Club](https://www.youtube.com/@SDFC)
- [Jordan has no life](https://www.youtube.com/@jordanhasnolife5163/playlists)