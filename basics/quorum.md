# Definition
In a distributed environment, a quorum is the minimum number of servers on which a distributed operation needs to 
be performed successfully before declaring the operation’s overall success.

# What value should we choose for Quorum?
More than half of the number of nodes in cluster. ```(N/2 + 1)``` where ``N``is total number of nodes in cluster
Eg:
* In a 5-node cluster, 3 voters must be online to have majority.
* In a 4-node cluster, three nodes must be online to have a majority.
* With 5-node it can afford 2 node failures whereas with 4-node it can afford only 1 node failure, because of this 
  logic, it is recommended to always have an odd number of total nodes in the cluster.

# Strict Quorums (read-only from home nodes)
* Strict Quorum is achieved when nodes follow the below protocol: ``R + W > N`` where:
  * N = nodes in the quorum group or the  number of replicas
  * W = minimum write nodes
  * R = minimum read nodes
* If a distributed system follows ``R + W > N `` rule, then every read will see at least one copy of the latest 
  value written.Basically read consistency is almost guaranteed due to the overlap between nodes with updated value 
  and those who returned the read request
* Some possible configuration for R and W
  * If R = 1 and W = N, the system is optimized for a fast read.
  * If W = 1 and R = N, the system is optimized for fast write.
  * If W + R > N, strong consistency is guaranteed (Usually N = 3, W = R = 2).
  * If W + R <= N, strong consistency is not guaranteed.
* Best performance (throughput/availability) when ``1 < r < w < n`` , because reads are more frequent than writes in 
  most applications
![](../../../resources/sd/quorum_strict.png)
# Sloppy Quorums and Hinted Handoff
* In a large system, there are many nodes (N) serving different partitions of the database with each row of data 
  replicated on T nodes. If too many home nodes are down, leading to insufficient quorums, others from the remaining 
  (N-T) nodes will step in to accept writes requests temporarily until the home nodes are back online (handoff).
* Two options when it comes to read requests.
  * Enforce strict quorum (read-only from home nodes), which offers better consistency.
  * If higher availability is needed, one can enable sloppy quorum for reads as well. This trade-off leads to 
    potential stale reads as ``W + R > T`` does not guarantee an overlap between writes and reads.
![](../../../resources/sd/quorum_sloppy.png)

# Examples
* Dynamo replicates writes to a sloppy quorum of other nodes in the system, instead of a strict majority quorum like Paxos. 
All read/write operations are performed on the first ``N`` healthy nodes from the preference list, 
which may not always be the first ``N`` nodes encountered while walking the consistent hashing ring.
* As stated above, quorum is also used to ensure that at least one node receives the update in case of failures.  
  For instance, in Cassandra, to ensure data consistency, each write request can be configured to be successful only 
  if the data has been written to at least a quorum (or majority) of replica nodes.
* For leader election, Chubby uses Paxos, which use quorum to ensure strong consistency.

# References
* [Distributed Systems 5.2: Quorums YouTube Video](https://www.youtube.com/watch?v=uNxl3BFcKSA&t=1s)
* [grokking-adv-system-design-intvw](https://www.educative.io/courses/grokking-adv-system-design-intvw/q2Oyw67Z8BG)
* https://towardsdatascience.com/database-replication-explained-3-32d6deceeca7
* https://martinfowler.com/articles/patterns-of-distributed-systems/quorum.html