## Problem Statement
An API rate limiter will throttle users based on the number of requests they are sending.

## Requirements
### Functional Requirements
* To limit the number of requests a client can send to an API within a time window.
* To make the limit of requests per window configurable.
* To make sure that the client gets a message (error or notification) whenever the defined threshold is crossed within a single server or combination of servers.
### Non-Functional Requirements
* **Availability**: Essentially, the rate limiter protects our system. Therefore, it should be highly available.
* **Low latency**: Because all API requests pass through the rate limiter, it should work with a minimum latency without affecting the user experience.
* **Scalability**: Our design should be highly scalable. It should be able to rate limit an increasing number of clients’ requests over time

## Back of Envelope Estimations/Capacity Estimation & Constraints

## Throttling Types
* Hard Throttling – Number of API requests cannot exceed the throttle limit
* Soft Throttling – Set the API request limit to exceed by some percentage. E.g, if the rate-limit = 100 messages/minute, and 10% exceed-limit, our rate limiter will allow up to 110 messages per minute
* Dynamic Throttling – The number of requests can exceed the limit if the system has some free resources available.

## High-level API design 

## Database Design
## High Level System Design and Algorithms
### Algorithms for Rate Limiting
#### Token Bucket
![](../resources/problems/rate_limiter/token_bucket_1.png)
* Steps
    * In the Token Bucket algorithm, we process a token from the bucket for every request.
    * New tokens are added to the bucket with rate r. The bucket can hold a maximum of b tokens
    * If a request comes and the bucket is full it is discarded.
![](../resources/problems/rate_limiter/token_bucket.png)
* Essential Parameters
    * Bucket capacity(C): The maximum number of tokens that can reside in the bucket.
    * Rate limit (R): The number of requests we want to limit per unit time.
    * Refill rate (1/R) : The duration after which a token is added to the bucket.
    * Requests count(N) : This parameter tracks the number of incoming requests and compares them with the bucket’s capacity.
* Pros
    * Simple and straightfoward to use
    * Smooth out the requests and process them at an approximately average rate.
    * Space efficient. The memory needed for the algorithm is nominal due to limited state
* Cons
    * Choosing an optimal value for the essential parameters is a difficult task.

#### Leaky Bucket
* The leaking bucket algorithm is a variant of the token bucket algorithm with slight modification where requests are processed at a fixed rate. 
![](../resources/problems/rate_limiter/leaky_bucket_1.png)
* Steps
    * When a request arrives, the system checks if the queue is full. If it is not full, the request is added to the queue.
    * Otherwise, the request is dropped.
    * Requests are pulled from the queue and processed at regular intervals.
    *  The algorithm process these requests at a constant rate in a first-in-first-out (**FIFO**) order.
![](../resources/problems/rate_limiter/leaky_bucket_2.png)
* Essential Parameters
    * Bucket capacity(C): It is equal to the queue size. The queue holds the requests to be processed at a fixed rate.
    * Outlfow Rate (Rout):  number of requests processed per unit time
* Pros
    *  Memory efficient given the limited queue size.
    * Requests are processed at a fixed rate therefore it is suitable for use cases that a stable outflow rate is needed.
    * smooths out bursts of requests and processes them at an approximately average rate
* Cons
    * A burst of traffic fills up the queue with old requests, and if they are not processed in time, recent requests will be rate limited.
    * no guarantee that requests get processed in a fixed amount of time
    * There are two parameters in the algorithm. Determining an optimal bucket size and outflow rate is a challenge.

#### Fixed Window Counter
* This algorithm divides the time into fixed intervals called windows and assigns a counter to each window. 
* When a specific window receives a request, the counter is incremented by one. 
* Once the counter reaches its limit, new requests are discarded in that window.
* Essential Parameters

* Pros
    * Memory efficient due to constraints on the rate of requests.
    * Easy to understand.
    * Resetting available quota at the end of a unit time window fits certain use cases.
* Cons
    * Spike in traffic at the edges of a window could cause more requests than the allowed quota to go through.
#### Sliding(Rolling) Window Log
* Essential Parameters
* Pros:
    * Rate limiting implemented by this algorithm is very accurate. In any rolling window, requests will not exceed the rate limit.
* Cons:
    * The algorithm consumes a lot of memory because even if a request is rejected, its timestamp might still be stored in memory.
#### Sliding(Rolling) Window Counters

### Rate Limiting in Distributed Systems
## References
* https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/design-of-a-rate-limiter
* https://github.com/Salah856/System-Design/blob/main/Design%20Rate%20Limiter.md
* https://cloudxlab.com/blog/system-design-how-to-design-a-rate-limiter/
* https://aaronice.gitbook.io/system-design/system-design-problems/designing-an-api-rate-limiter