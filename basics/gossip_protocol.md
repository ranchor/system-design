# Defintion
The gossip protocol is a decentralized communication technique used in distributed systems to disseminate information among network nodes. 
Resembling the spread of rumors in a social network, nodes exchange updates with a subset of peers, rapidly propagating information across the network.

## Gossip Algorithm 
**Gossip protocol or epidemic protocol** works as follows:
* Each node maintains a node membership list, which contains member IDs and heartbeat counters.
* Each node periodically increments its heartbeat counter.
* Each node periodically sends heartbeats to a set of random nodes, which in turn propagate to another set of nodes.
* During gossip exchange, nodes merge received metadata with their own, prioritizing higher version numbers.
* Once nodes receive heartbeats, membership list is updated to the latest info.
* If the heartbeat has not increased for more than predefined periods, the member is considered as offline.
* Peer node selection can be based on factors like random selection, least contacted nodes, or network topology awareness.

## Strategies to Spread Messages
1. **Push-Based Gossip:**
   - Nodes actively broadcast updates to randomly selected peers.
   - Promotes rapid dissemination of updates throughout the network.

2. **Pull-Based Gossip:**
   - Nodes periodically request updates from a subset of peers.
   - Reduces network overhead by minimizing unnecessary message broadcasts.

3. **Hybrid Gossip:**
   - Combines push and pull strategies to balance efficiency and overhead.
   - Dynamically adapts to network conditions and update availability.

## **Types of Gossip Protocols**

1. **Anti-Entropy Gossip Protocol:**
   - **Overview:** Ensures consistency and synchronization of data across distributed systems.
   - **Strategy:** Nodes exchange data summaries to detect and repair inconsistencies.
   - **Use Cases:** Widely used in distributed databases like Cassandra and Riak for data repair and synchronization.

2. **Rumor-Mongering(Dissemination) Gossip Protocol:**
   - **Overview:** Probabilistic dissemination technique for spreading rumors or events.
   - **Strategy:** Nodes randomly propagate rumors to peers, gradually spreading information.
   - **Use Cases:** Commonly employed in peer-to-peer networks, social networks, and decentralized messaging systems.

3. **Aggregation Gossip Protocol:**
   - **Overview:** Aggregates data or statistics across distributed nodes to compute global metrics.
   - **Strategy:** Nodes exchange local data, compute aggregates, and disseminate global results.
   - **Use Cases:** Found in distributed monitoring systems, sensor networks, and analytics platforms for decentralized computation of global metrics.


## Gossip Protocol Use Cases
- Database replication
- Information dissemination
- Failure detection
- Cluster membership maintenance
- Leader election

### Examples
* Dynamo & Cassandra use gossip protocol which allows each node to keep track of state information about the other  
* nodes in the cluster, like which nodes are reachable, what key ranges they are responsible for, etc

## Advantages
- Scalable
- Fault-tolerant
- Robust
- Convergent consistency
- Decentralized
- Simple to implement
- Integration and interoperability
- Bounded load

## Disadvantages
- Eventual consistency
- Unawareness of network partitions
- Bandwidth consumption
- Increased latency
- Debugging and testing challenges
- Scalability limitations
- Prone to computational errors

# References
* https://highscalability.com/gossip-protocol-explained/
* https://docs.google.com/presentation/d/15zn16VbJ-HRvshGb71AV4aYU7K-DDXTflnVJjiQ_flA/edit#slide=id.g12562b89a86_0_0

