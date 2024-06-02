# Introduction
* A cache is a temporary storage area that stores the result of expensive responses or frequently accessed data in 
memory so that subsequent requests are served more quickly.
* Take advantage of the locality of reference principle: recently requested data is likely to be requested again.
* Exist at all levels in architecture, but often found at the level nearest to the front end.

# Considerations for using Cache
* Decide when to use cache - Data is read frequently but modified infrequently.
* Expiration Policy : Good Practice to implement expiration policy for cached data.
* Consistency: Make sure the data store and cache in sync.
* Mitigating Failures: Single cache server represents a potential single point of failure.
* Eviction Policy: Once the cache is full, any requests to add items to the cache might cause existing items to be 
  removed. This is called cache eviction. Use proper eviction policy.

# Types of Cache
## Application server cache
- Cache placed directly on a request layer node.
- Cache on request layer node can exists in memory and on the node's local disk.
- When a request layer node is expanded to many nodes
    - Load balancer randomly distributes requests across the nodes.
    - The same request can go to different nodes.
    - Increase cache misses.
    - Solutions:
        - Global caches
        - Distributed caches
## Distributed cache
- Each request layer node owns part of the cached data.
- Entire cache is divided up using a consistent hashing function.
- Pro
    - Cache space can be increased easily by adding more nodes to the request pool.
- Con
    - A missing node leads to cache lost.
## Global cache
- A server or file store that is faster than original store, and accessible by all request layer nodes.
- Two common forms
    - Cache server handles cache miss.
        - Used by most applications.
    - Request nodes handle cache miss.
        - Have a large percentage of the hot data set in the cache.
        - An architecture where the files stored in the cache are static and shouldn’t be evicted.
        - The application logic understands the eviction strategy or hot spots better than the cache
## Content Delivery (or Distribution) network (CDN)
- For sites serving large amounts of static media.
- Process
    - A request first asks the CDN for a piece of static media.
    - CDN serves that content if it has it locally available.
    - If content isn’t available, CDN will query back-end servers for the file, cache it locally and serve it to the requesting user.
- If the system is not large enough for CDN, it can be built like this:
    - Serving static media off a separate subdomain using lightweight HTTP server (e.g. Nginx).
    - Cutover the DNS from this subdomain to a CDN later.

# Cache Update Strategies/Cache Invalidation
- Keep cache coherent with the source of truth. Invalidate cache when source of truth has changed.

## Cache-Aside Cache
* Most commonly used caching approach.
* Cache sits on the side and the application directly talks to both the cache and the database.
![](../../../resources/sd/cache_aside.png)
``` python
def get_user(self, user_id):
    user = cache.get("user.{0}", user_id)
    if user is None:
        user = db.query("SELECT * FROM users WHERE user_id = {0}", user_id)
        if user is not None:
            key = "user.{0}".format(user_id)
            cache.set(key, json.dumps(user))
    return user
```
### Use-Cases
* Work best for read-heavy workloads.
* _Memcached_ and _Redis_ are widely used
### Pros
* Resilient to cache failures
* Data model in cache can be different than the data model in database
### Cons
* Higher latency for read operations as application updates cache during cache miss scenario.
* Cache may become inconsistent with the database.

## Read-Through Cache
* Cache sits in-line with the database
![](../../../resources/sd/cache_read_through.png)
* **Difference between read-through and cache-aside:**
  * In cache-aside, the application is responsible for fetching data from the database and populating the cache. 
  In read-through, this logic is usually supported by the library or stand-alone cache provider.
  * Unlike cache-aside, the data model in read-through cache cannot be different than that of the database.
### Use-Cases
* Work best for read-heavy workloads like news story
### Pros
* Lower latency as compared to cache aside.
### Cons
* Cache may become inconsistent with the database.

## Write-Through cache
* Data is written into the cache and permanent storage at the same time.
* Cache sits in-line with the database and writes always go through the cache to the main database.
![](../../../resources/sd/cache_write_through.png)
Application code:
```
set_user(12345, {"foo":"bar"})
```
Cache code:
```
def set_user(user_id, values):
    user = db.query("UPDATE Users WHERE id = {0}", user_id, values)
    cache.set(user_id, user)
```
### Use-Cases
* When paired with read-through caches, we get all the benefits of read-through and we also get data consistency 
  guarantee, freeing us from using cache invalidation techniques.
* [DynamoDB Accelerator (DAX)](https://aws.amazon.com/dynamodb/dax/) is a good example of read-through / write-through cache
### Pros
* Fast retrieval, complete data consistency, robust to system disruptions.
### Cons
* Higher latency for write operations as data is written to the cache first and then to the main database
* Most data written might never be read, which can be minimized with a TTL

## Write-Around cache
- Data is written directly to permanent storage, bypassing the cache.
- Only the data that is read makes it way into the cache.
![](../../../resources/sd/cache_write_around.png)

### Use-Cases
* Provides good performance in situations where data is written once and read less frequently or never.
* Eg: real-time logs or chatroom messages.
* this pattern can be combined with cache-aside as well.
### Pros
- Reduce the cache being flooded with write operations that will not subsequently be re-read
### Cons
- Query for recently written data creates a cache miss and higher latency.

## Write-Back (Write-Behind) cache
- Data is only written to cache.
- Write to the permanent storage is done later on asynchronously.
![](../../../resources/sd/cache_write_back.png)
### Use-Cases
* Write back caches improve the write performance and are good for write-heavy workloads.
* Can reduce overall writes to the database, which decreases the load and reduces costs
### Pros
- Low latency, high throughput for write-intensive application
### Cons
- Risk of data loss in case the cache goes down prior to its contents hitting the data store.
- More complex to implement write-behind than  to implement write-around or write-through.


# Cache Eviction Policies
* **First In First Out (FIFO)**: The cache evicts the first block accessed first without any regard to how often or how many times it was accessed before.
* **Last In First Out (LIFO)**: The cache evicts the block accessed most recently first without any regard to how often or how many times it was accessed before.
* **Least Recently Used (LRU)**: Discards the least recently used items first.
* **Most Recently Used (MRU)**: Discards, in contrast to LRU, the most recently used items first.
* **Least Frequently Used (LFU)**: Counts how often an item is needed. Those that are used least often are discarded first.
* **Random Replacement (RR)**: Randomly selects a candidate item and discards it to make space when necessary.

Least-recently-used (LRU) is the most popular cache eviction policy

# References
* [grokking-the-system-design-interview](https://www.educative.io/courses/grokking-the-system-design-interview/3j6NnJrpp5p)
* https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/
* [System Design Interview - An Insider's Guide book (Volume 1)](https://amzn.to/3ggPKAG)
* [Introduction to architecting systems](https://lethain.com/introduction-to-architecting-systems-for-scale/)