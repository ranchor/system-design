# Table of contents

- [Chapter 1: Reliable, Scalable, and Maintainable Applications](#chapter-1-reliable-scalable-and-maintainable-applications)
    - [Reliability](#reliability)
    - [Scalability:](#scalability)
    - [Maintainability:](#maintainability)
- [Chapter 2: Data Models and Query Languages](#chapter-2-data-models-and-query-languages)
    - [Relational model vs document model](#relational-model-vs-document-model)
    - [Query languages for data](#query-languages-for-data)
    - [Graph-like data models](#graph-like-data-models)
- [Chapter 3: Storage and Retrieval](#chapter-3-storage-and-retrieval)
    - [Transactional optimized (OLTP)](#transactional-optimized-oltp)
    - [Analytics optimized (OLAP):](#analytics-optimized-olap)
    - [Transactional optimized (OLTP) vs Analytics optimized (OLAP)](#transactional-optimized-oltp-vs-analytics-optimized-olap)
    - [Column-Oriented Storage](#column-oriented-storage)
- [Chapter 5: Replication](#chapter-5-replication)
    - [Single-Leader Replication](#single-leader-replication)
    - [Multi-leader replication](#multi-leader-replication)
    - [Leaderless replication](#leaderless-replication)
- [Chapter 6: Partitioning](#chapter-6-partitioning)
    - [](#)
- [References](#references)

## Chapter 1: Reliable, Scalable, and Maintainable Applications

### Reliability

* Simple definition of reliability: Systems should work correctly even in the face of adversity, including human errors.
* Typical expectations:
    * Application performs the function the user expected
    * Tolerate the user making mistakes
    * Its performance is good.
    * The system prevents abuse.
* **A fault is usually defined as one component of the system deviating from its spec** _whereas_
* **Failure is when the system as a whole stops providing the required service to the user.**
* Generally prefer tolerating faults over preventing faults.
* **Hardware Faults**
    * Single machines are made more resilient through redundant hardware
    * Larger data and computing demands have driven move to multi-machine redundancy, often using software
      fault-tolerance techniques
* **Software Errors**
    * Software errors are a systematic error within the system, they tend to cause many more system failures than
      uncorrelated hardware faults.
* **Human Errors**
    * Can combine several approaches to deal with human error, including:
        * Minimize opportunities for human error
        * Provide fully featured non-production sandbox environments where people can explore and experiment safely.
        * Automated testing
        * Decouple places where people make the most mistakes from places where mistakes can cause failures
        * Allow quick & easy recovery from problems
        * Use detailed and clear monitoring
        * Provide good management & training

### Scalability:

* Scalability Definition: As a system grows (including complexity) there should be reasonable ways of dealing with that
  growth.
* _Latency_: duration that a request is waiting to be handled - during which is latent.
* _Response time_: what the client sees.
* Approaches for Coping with Load
    * Can scale up(vertical scaling) or scale out(horizontal scaling)
    * Stateless services are easy to scale out, but scaling out stateful systems can introduce a lot of complexity

### Maintainability:

* Be able to work on it productively.Three design principles for software systems:
* **Operability**: Making Life Easy for Operations. Good operability includes:
    * Providing visibility into the runtime behavior and internals
    * Support for automation and integration with standard tools
    * Self-healing
* **Simplicity**: Managing Complexity
    * Good to remove accidental complexity (not inherent in problem, just the implementation)
    * A good abstraction hides implementation details behind easy to understand interfaces.
* **Evolvability**: Make it easy for engineers to make changes to the system in the future
    * Agile working patterns provide a framework for adapting to change
    * Evolvability is defined as agility at a large data system level
    * Simplicity and good abstractions can go a long way toward evolvability.

## Chapter 2: Data Models and Query Languages

### Relational model vs document model

### Query languages for data

### Graph-like data models

## Chapter 3: Storage and Retrieval

* Two Broad Categories of Storage engines: **OLTP** or **OLAP**.
    * **Online transactional processing(OLTP)** - Usually a single user is involved in the transaction. Usually needs a
      small set of data, and can result in random writes. E.g. MySQL, Oracle.
    * **Online analytic processing(OLAP)** - Businesses making sense of their data or third-party (3P) data about their
      customers. One can use materialized view, and keep it away from OLTP flow. Usually, such materialized views have
      100s of columns, so using column-based DB like Snowflake is suggested.

### Transactional optimized (OLTP)

1. Log structured:
    * Hash Indexes
        * Like in-memory HashMaps or Hash Table
        * All keys must fit in memory. Each key is mapped to a byte offset in data file for the value to be found.
        * Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset
          of the data you just wrote.
        * Bitcask storage in Riak.
        * Write-only file segments, compaction, tombstones like Kafka.
            * Compaction means throwing away duplicate keys in the log, and keeping only the most recent update for
              each key.
            * Special deletion record to the data file (tombstone) that tells the merging process to discard
              previous
              values.
        * Each segment now has its own in-memory hash table, mapping keys to file offsets
        * Cons/Limitations
            * The hash table must fit in memory. It is difficult to make an on-disk hash map perform well.
            * Range queries are not efficient.
    * SSTables (Stored String Table)
        * As Hash indexes but segment files sorted by key.
        * Pros(Advantages over log segments with hash indexes)
            * Merging segments is simple and efficient (we can use algorithms like mergesort
            * You no longer need to keep an index of all the keys in memory
                * Less memory.
                * Find value by scanning between two other keys.
            * Block read compressed to save IO/disk space
        * LSM-Tree (Log-Structured Merge-Tree), parts:
            * Memtable for current segment (Read-back tree or AVL tree).
                * When write comes in, add it to an in-memory balanced tree structure (memtable)
                * Writen to disk as a SSTable when reaches some threshold.
                * On a read request, try to find the key in the memtable, then in the most recent on-disk segment,
                  then in the next-older segment, etc
            * Periodic compaction (leveled or size-tiered).
            * Unordered log for recovery.
        * LevelDB, RocksDB, Cassandra, InfluxDB, HBase, ScyllaDB, BigTable, Lucene for the term dictionary.
            * LSM-tree algorithm can be slow when looking up keys that don't exist in the database. To optimise
              this,
              storage engines often use additional _Bloom filters_ (a memory-efficient data structure for
              approximating
              the contents of a set).
2. Page-oriented(B-Trees)
    * Most widely used indexing structure.
    * Key-value pairs sorted by key, which allows efficient key-value lookups and range queries.
    * Break the database down into fixed-size blocks or pages, traditionally _4KB_.
    * One page is designated as the _root_, and you start from there. The page contains several keys and references to
      child pages.
    * If you want to update the value for an existing key in a B-tree, you search for the leaf page containing that key,
      change the value in that page, and write the page back to disk
    * f you want to add new key, find the page and add it to the page. If there isn't enough free space in the page to
      accommodate the new key, it is split in two half-full pages, and the parent page is updated to account for the new
      subdivision of key ranges.
    * Trees remain _balanced_. Depth of _O(log n)_ Branching factor is ~= 3-4 levels typically.
    * underlying write operation of a B-tree is to overwrite a page on disk with new data.
    * _Write-ahead log_(WAL, also know as the redo log) for resilience.
3. LSM-tree vs B-Trees
    * Advantages of LSM-trees
        * LSM-trees are typically able to sustain higher write throughput than B-trees, party because they sometimes
          have lower write amplification: a write to the database results in multiple writes to disk. The more a
          storage engine writes to disk, the fewer writes per second it can handle.
        * LSM-trees can be compressed better, and thus often produce smaller files on disk than B-trees. B-trees
          tend to leave disk space unused due to fragmentation.
    * Downsides of LSM-trees
        * LSM-trees compaction process can cause operational issues whereas B-trees can be more predictable. The
          bigger the database, the more disk bandwidth is required for compaction. Compaction cannot keep up
          with the rate of incoming writes, if not configured properly you can run out of disk space.
        * On B-trees, each key exists in exactly one place in the index. This offers strong transactional semantics.
          Transaction isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks can
          be directly attached to the tree.

### Analytics optimized (OLAP):

* Data Warehousing
    * A data warehouse is a separate database that analysts can query to their heart's content without affecting OLTP
      operations
    * It contains read-only copy of the dat in all various OLTP systems in the company.
    * Data is extracted out of OLTP databases (through periodic data dump or a continuous stream of update), transformed
      into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse **(process
      Extract-Transform-Load or ETL).**
    * The data model for data warehouses was relational for a long time, but divergence to other models such as Hadoop
      has occurred, independent of whether SQL is still the query language
    * Stars Schema(aka dimensional modeling)
        * At the center of the schema is a **fact table**, with facts representing individual events, usually very wide
          with
          lots of columns.
        * The other, typically smaller, tables surrounding the fact table are dimension tables.
        * Facts have event-specific data elements in some columns, but foreign key references (e.g. product ID) in
          others
          dimension tables.
        * Dimension tables represent the who, what, when, how, why of the event.
    * Snowflake schema
        * when dimensions are broken down in subdimensions.e.g. countries having subdimensions for regions, and possibly
          regions broken into states, then the addition branching makes the diagram look more like a snowflake.
        * More normalized but harder to work with.
    * A data warehouse may have multiple fact tables, thus multiple stars/snowflakes

### Transactional optimized (OLTP) vs Analytics optimized (OLAP)

### Column-Oriented Storage

## Chapter 5: Replication

* Replication is keeping an entire copy of the data on multiple machines.
* Reasons for Replication
    * High availability: Keeping the system running, even when one machine (or several machines, or an entire
      datacenter) goes down
    * Disconnected operation: Allowing an application to continue working when there is a network interruption
    * Latency: To keep data geographically close to your users which in turns reduce latency.
    * Scalability: Being able to handle a higher volume of reads than a single machine could han‐ dle, by performing
      reads on replicas
* Algorithms for replicating changes
    * Single Leader Replication
    * Multi-Leader Replication
    * Leaderless Replication

### Single-Leader Replication

#### Leaders and Followers

* One replica/node is designated the **leader**(also known as master or primary); all others are **followers**(read
  replicas, slaves, secondaries, or hot standbys.
* All writes must go through the leader; reads can come from any replicas.
* On writes, leader sends change log information to all followers
    * **Synchronous** - clear success/failure; clients must wait for all replicas
    * **Asynchronous** - lowest latency; no guarantee of durability after successful write
    * **Semi-Synchronous** - one of the followers is synchronous, and the others are asynchronous. If the synchronous
      follower becomes unavailable or slow, one of the asynchronous followers is made synchronous
    * Chain-replication (Microsoft Azure Storage).
    * Single synchronous follower improves durability.
* Adding followers usually involves a snapshot (like a backup) followed by applying changes since the snapshot.

#### Handling Node Outages

* Follower failure: Catch-up recovery
    * Follower can connect to the leader and request all the data changes that occurred during the time when the
      follower was disconnected.
* Leader failure: Failover
    * An Automatic failover consists of following steps:
        * _Determining that the leader has failed._ If a node does not respond in a period of time it's considered dead.
        * _Choosing a new leader._ The best candidate for leadership is usually the replica with the most up-to-date
          changes from the old leader.
        * _Reconfiguring the system to use the new leader_. The system needs to ensure that the old leader becomes a
          follower and recognises the new leader.
    * Things that could go wrong:
        * If asynchronous replication is used, the new leader may not have received all the writes from the old leader
          before it failed.
        * Discarding writes is especially dangerous if other storage systems outside of the database need to be
          coordinated with the database contents
        * It could happen that two nodes both believe that they are the leader (split brain). Data is likely to be lost
          or corrupted.
        * What is the right timeout before the leader is declared dead?

#### Implementation of Replication Logs

* **Statement-based replication:**
    * leader logs every users statement and sends it to its followers (every ``INSERT``, ``UPDATE`` or ``DELETE``).
    * Issues:
        * Non-deterministic functions, like ``rand()`` or ``now()`` will generate different values on replicas.
        * Autoincrement columns - Statements that depend on existing data
        * Side-effect functions. May result on different results on each replica.
    * Replace any nondeterministic function with a fixed return value in the leader.
* **Write-ahead log(WAL) shipping:**
    * Log is an append-only sequence of bytes containing all writes to the database. leader the exact same log across
      the network to its followers.
    * Log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk
      blocks.
    * Issue: Tightly coupled to storage format of storage engineer. Non-backward compatibilities issue between
      followers and leader can cause downtime.
* **Logical(row-based) log replication:**
    * use different log formats for replication for different types of storage engine.
    * _Logical log_ for a relational database is usually a sequence of records describing writes to database tables at
      the granularity of a row.
* **Trigger-based replication:**
    * Register custom application code that is automatically executed when a data change (write transaction) occurs in a
      database system
    * Issues:
        * Bigger overhead.
        * Bugs prone.

#### Problems with Replication Lag

* **Read-your-own-writes**
    * Write followed by a read from a stale replica will look to the user like their data was lost
    * Fixes to read-after-write consistency in a system:
        * Read from leader
            * if few things modifiable by user, then read them all from the leader
            * Could track most recent update and force reads from the leader for a short period, e.g. one minute
        * If client knows timestamp of last write, replicas can delegate reads if they aren’t upto date at least to that
          timestamp. The timestamp could be a logical timestamp.
    * Cross-device read-after-write consistency is even trickier
* **Monotonic reads**
    * User makes two reads, the second is from a more lagged replica. This will look to the user like information
      went backward in time.
    * Fix:
        * Consistently reading from same replica will solve this (user affinity hashing)
* **Consistent prefix reads**
    * If two writes appear to third person in the reverse order, may seem to violate causality. More common when also
      partitioning
    * In partitioned DBs, event1 happens-before event2, but if events are in different partitions, a client can see
      event2 before event1.
    * Fix:
        * Any writes that are causally related to each other are written to the same partition—but in some applications
          that cannot be done efficiently.

### Multi-leader replication

* Clients send each write to one of several leader nodes, any of which can accept writes. The leaders send streams of
  data change events to each other and to any follower nodes.
* Also known as master–master or active/active replication

#### Use-Cases

* Multi-data center operation
    * Within each datacenter, regular leader– follower replication is used; between datacenters, each datacenter’s
      leader replicates its changes to the leaders in other datacenters.
    * Pros: leader per data center improves performance and tolerates outages and slow traffic
      between data centers better
    * Cons: same data may be concurrently modified in two different datacenters, and those write conflicts must be
      resolved
* Clients with offline operation
    * offline clients act as leaders, then sync when online again
    * CouchDB
* Collaborative editing
    * Google Docs is essentially multi-leader replication

#### Handling Write Conflicts
* Conflict avoidance
  * simplest strategy for dealing with conflicts is to avoid them. If all writes for a particular record go through 
    the same leader, then conflicts cannot occur.
  * Assigning each user a home datacenter works until a failure or they move
* **Converging toward a consistent state**
    * Database must resolve the conflict in a convergent way, all replicas must arrive a the same final value when all
      changes have been replicated.
    * Different Approaches
        * Last write wins (data loss) - often based on somewhat arbitrary unique ID(timestamp, long random number, UUID)
          , pick the write with the highest ID as the winner and throw away the other writes
        * Higher numbered replica wins (data loss).
        * Concatenate values: Somehow merge the conflicting values being written
        * Preserve all values and let the user resolve.
* **Custom conflict resolution**
    * **Resolve On write**. As soon as the database system detects a conflict in the log of replicated changes, it 
      calls the
      conflict handler.
    * **Resolve On read.** All the conflicting writes are stored. On read, multiple versions of the data are returned 
      to the
      application. The application may prompt the user or automatically resolve the conflict. CouchDB works this way.

#### Automatic Conflict Resolution
* Conflict-free replicated data types (CRDTs) - data structures for sets, ordered lists, counters, etc. which have 
sensible conflict resolution rules built in
* Mergeable persistent data structures - track history and use three-way merge functions (not two-way merge)
* Operational transformation - the conflict resolution algorithm for ordered lists, which Google Docs uses (for an 
  ordered sequence of characters)

#### Multi-Leader Replication Topologies
* A replication topology describes the communication paths along which writes are propagated from one node to another.
* Common topologies include:
  * All-to-all - simplest 
  * Circular - simple version is unidirectional 
  * Star - one central hub; can be generalized to a tree 
* Circular and star topologies require multiple hops
  * Need to track nodes seen for each write 
  * Single node failure could cause replication failure 
* All-to-all could have the consistent prefix reads kind of problem 
  * Version vectors (later in this chapter) could be used, but not popular at time of printing of this boo


### Leaderless replication

* Any replica can directly accept writes from clients.
* Amazon Dynamo led the way for leaderless replication. Other examples: Riak, Cassandra, Voldemort
* Either the client directly sends writes to multiple replicas, or a coordinator node handles that on the behalf of
  client
* **Writing to the Database When a Node Is Down**
    * In a leaderless configuration, failover does not exist. Clients send the writes to all replicas in parallel.
    * Read requests are also sent to several nodes in parallel. The client may get different responses. Version numbers
      are used to determine which value is newer.
    * Catching up on stale data from missed writes
        * **Read repair** - when a client detects a stale read, it writes the newer value back
        * **Anti-entropy process** - background process that searches for differences between replicas and corrects
          them. No guarantee of order or time until repair.
* **Quorums for reading and writing**
    * The minimum number of nodes or votes required for the read or write to be valid is called a quorum reads and
      writes.
    * If there are n nodes, r are required for reads, w are required for writes, then ``w + r > n``
    * Typically, n is an odd number(typically 3 or 5) and to set ``w = r = (n + 1)/2 (rounded up)``
    * Writes can tolerate ``n - w`` nodes being down; reads can tolerate ``n - r`` nodes down.
      If more than that are down, the operation returns an error
* **Limitations**
    * Sloppy quorums (next section): the w writes may end up on different nodes than the r reads, so there is no longer
      a guaranteed overlap.
    * Concurrent writes
    * Concurrent write and read: If write happens concurrently with a read, the writes may be reflected on only some
      of the replicas.
    * Partial write failure in quorum: If a write operation fails to get a quorum, a subset of nodes won’t roll back,
      so will incorrectly have the newer value.
    * If a failed node A is restored from node B, anything stale on B is now stale on A, and both A and B can contribute
      toward a read quorum
* **Sloppy Quorums and Hinted Handoff**
    * Sloppy Quorum: writes and reads still require w and r successful responses, but those may include nodes that are
      not among the designated n "home" nodes for a value. Once the network interruption is fixed, any writes are sent
      to the appropriate "home" nodes **(hinted handoff)**.
    * This provides fault tolerance, but violates quorum properties, allowing stale reads
* **Concurrent Writes**
    * Concurrent writes can happen in multi-leader replication or leaderless. In leaderless replication they can also
      happen during read repair and hinted handoff.
    * **Last write wins (discarding concurrent writes)**: Biggest value of some “timestamp” with each write wins. Even
      though clients saw multiple successful writes, all but one will be lost.
    * The author shows a single-node algorithm for tracking writes. Some details are:
        * Server maintains a version with each key.
        * Clients must do a read, which includes version(s) and value(s), before they write.
        * Clients must merge multiple values read before doing write.
        * Server increments max version each write.
        * Server receiving write can discard data from that version or older, but keeps newer data.
    * _Version vectors_ is the multi-replica version of the above algorithm
        * This requires the servers to maintain a version per key and per replica
        * Servers now keep track of multiple versions & values from each replica
        * Collection of version numbers from all the replicas is called a version vector.
        * Version vector are sent from the database replicas to clients when values are read, and need to be sent back
          to the database when a value is subsequently written
        * Clients merging multiple sibling values must consider versions on each replica

## Chapter 6: Partitioning

###   

## References

* https://github.com/SanDiegoMachineLearning/bookclub/blob/master/designing-data-intensive-apps.md
* https://github.com/keyvanakbary/learning-notes/blob/master/books/designing-data-intensive-applications.md
* https://danlebrero.com/2021/09/01/designing-data-intensive-applications-summary/