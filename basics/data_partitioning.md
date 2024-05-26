# Table of contents

- [Data Partitioning](#data-partitioning)
- [Partitioning methods](#partitioning-methods)
    - [Horizontal partitioning — also known as sharding](#horizontal-partitioning--also-known-as-sharding)
    - [Vertical partitioning](#vertical-partitioning)
    - [Directory-based partitioning](#directory-based-partitioning)
- [Partitioning criteria/techniques](#partitioning-criteriatechniques)
    - [Range partitioning](#range-partitioning)
    - [Key or hash-based partitioning](#key-or-hash-based-partitioning)
    - [List partitioning](#list-partitioning)
    - [Round-robin partitioning](#round-robin-partitioning)
    - [Composite partitioning](#composite-partitioning)
- [Common problems of sharding](#common-problems-of-sharding)
    - [Joins and denormalization](#joins-and-denormalization)
    - [Referential integrity](#referential-integrity)
    - [Rebalancing partitions](#rebalancing-partitions)
- [References](#references)

# Data Partitioning
**Data partitioning** in simple terms is a method of distributing data across multiple tables,
systems or sites to improve query processing performance and make the data more manageable.

# Partitioning methods
## Horizontal partitioning — also known as sharding
* Range-Based sharding/partitioning
* Split the table data horizontally based on the range of values defined by the **partition key.**
* Smaller partitions are called **shards**.
* Pros:
    * Horizontal partitioning spreads the load over more computers and ensures that the time required for a query is as minimal as possible.
    * Each partition has the same schema as the original database so we don't need to combine data from multiple
      partitions to answer any query.
* Cons:
    * If the value whose range is used for sharding isn’t chosen carefully, the partitioning scheme will lead to unbalanced servers.

![](../../../resources/sd/sharding_horizontal.png)
## Vertical partitioning
* Divides the table vertically (by columns), which means that the structure of the main table changes in the new ones.
* Divide data for a specific feature to their own server.
* **Pros**
    * Straightforward to implement.
    * Low impact on the application.
* **Cons**
    * System might need to combine data from multiple partitions to answer a query. For example,  a profile view
      request needs to combine a user profile, connections, and articles data. Increase overall complexity
    * To support growth of the application, a database may need further partitioning.

![](../../../resources/sd/sharding_vertical.png)

## Directory-based partitioning
* A lookup service that knows the partitioning scheme and abstracts it away from the database access code.
* Allow the addition of DB servers or change of partitioning schema without impacting the application.
* Cons
    * Can be a single point of failure.

![](../../../resources/sd/sharding_directory.png)

# Partitioning criteria/techniques
## Range partitioning
* This type of partitioning assigns rows to partitions based on column values falling within a given range. e.g,
  store_id is a column of the table.
```
PARTITION BY RANGE (store_id) (
    PARTITION p0 VALUES LESS THAN (6),
    PARTITION p1 VALUES LESS THAN (11),
    PARTITION p2 VALUES LESS THAN (16),
    PARTITION p3 VALUES LESS THAN (21)
);
```
## Key or hash-based partitioning
* Apply a hash function to some key  attribute of the entry to get the partition number. e.g. partition based on the
  year in which an employee was hired
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY HASH( YEAR(hired) )
PARTITIONS 4;
```
* Problems
    * Adding new servers may require changing the hash function, which would need redistribution of data and downtime for the service.
    * Workaround: consistent hashing.
## List partitioning
* Similar to partitioning by RANGE, except that the partition is selected based on columns matching one of a set of
  discrete values. Each partition is assigned a list of values. e.g.
```
PARTITION BY LIST(store_id) (
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);
```


![](../../../resources/sd/sharding_criteria.png)


## Round-robin partitioning
* With n partitions, the i tuple is assigned to partition i % n
## Composite partitioning
* Combine any of above partitioning schemes to devise a new scheme.
* Consistent hashing is a composite of hash and list partitioning
* **Key** -> **reduced key space through hash** -> **list** -> **partition.**
* Various types of composite partitioning are:
    * Composite Range–Range Partitioning
    * Composite Range–Hash Partitioning
    * Composite Range–List Partitioning
    * Composite List–Range Partitioning
    * Composite List–Hash Partitioning
    * Composite List–List Partitioning

![](../../../resources/sd/sharding_criteria_composite.png)

# Common problems of sharding
## Joins and denormalization
* Joins will not be performance efficient since data has to be compiled from multiple servers.
* Workaround: denormalize the database so that queries can be performed from a single table. But this can lead to
  data inconsistency.
## Referential integrity
* Difficult to enforce data integrity constraints (e.g. foreign keys).
* Workaround
    * Referential integrity is enforced by application code.
    * Applications can run SQL jobs to clean up dangling references.
## Rebalancing partitions
* Necessity of rebalancing
    * Data distribution is not uniform.
    * A lot of load on one shard.
* Create more db shards or rebalance existing shards changes partitioning scheme and requires data movement.

# References
* [grokking-the-system-design-interview](https://www.educative.io/courses/grokking-the-system-design-interview/mEN8lJXV1LA)
* https://medium.com/must-know-computer-science/system-design-sharding-data-partitioning-b7201596aafa
* https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning.html