
# Unique ID Generator

## Problem Statement

## Requirements

### Functional Requirements
* **Uniqueness**: Each time the ID generator is called, it returns a key that is unique across all node points for the system.
* **Sequencing**: IDs mush be sortable or sequenced. The ID has a timestamp. A timestamp allows the user to order the entries according to the time they were created.


### Non-Functional Requirements
* **Scalability**: Generate at least a billion unique IDs per day.
* **Availability**: Since multiple events happen even at the level of nanoseconds, our system should generate IDs for all the events that occur.
* **Latency**:  Happens in real-time with minimal latency.
* **Security**: Encryption at rest (in data store), encryption in transist using https
* **Observability**:  Metrics, monitors, alarms, tracing, logging

## Back of Envelope Estimations/Capacity Estimation & Constraints
* Assumptions
    *  

## API Design

## High Level Design
### Option1: [Centralized] Ticket Server - Using a Database
* Use a centralized auto_increment feature in a single central database server called the Ticket Server.
* Pros
    * Sequenced ids
    * IDs could be numeric
* Cons
    * Single point of Failure (SPF). Not scalable and highly available solution.
    * Multiple ticket servers can be set up, but this introduces new challenges such as data synchronization.
    * No guarantee of sorting over time in multiple servers.

### Option2: [Centralized] Using a Range Handler

### Option3: [Decentralized] Using UUID
* UUIDs (universally unique IDs) are 128-bit(32 char) number
* UUID consists of 32 hexadecimal (base-16) digits, displayed in five groups separated by hyphens
``
* UUIDs have a very low probability of collision, making them suitable for generating unique IDs.
* Each server generating UUID indepdently.
* Pros
    * Generating UUID is simple
    * No coordination between servers is needed so there will not be any synchronization issues
    * Very low probability of getting collision.
    * Scalable as each web server can generate own id.
    * Available
* Cons
    * IDs are 128 bits long(32 char), so it can be slow to index/inserts and difficult to ,manage. Query performance takes a hit for big index.
    * IDs are not sequential or sorted by time
    * IDs could be non-numeric.
    

### Option4:  [Decentralized] Using Unix Time Stamps
### Option5:  [Decentralized] Using Twitter SnowFlake
* Divide an ID into different sections:
    * Sign (1 bit): Always be 0 (Reserved for future used).
    * Timestamp (41 bits): Milliseconds since the epoch or custom epoch.
    * Machine ID (10 bits): Hold up to 1024 machines.
    * Sequence number (12 bits): The sequence number is incremented by 1 and is reset to 0 every millisecond.
* Pros
    * IDs are unique. Numeric and 64-bit
    * IDs are time sorted
    * Highly available
    * Highly Scalale
* Cons
    * 
### Option6:  [Decentralized] Using Logical Clocks
### Option7:  [Decentralized] TrueTime API





## Reference
* https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/design-of-a-unique-id-generator
* https://towardsdatascience.com/ace-the-system-design-interview-distributed-id-generator-c65c6b568027
* https://medium.com/double-pointer/system-design-interview-scalable-unique-id-generator-twitter-snowflake-or-a-similar-service-18af22d74343