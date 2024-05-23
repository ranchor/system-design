## Problem Statement
Design a scalable and efficient news feed system  similar to those used in social media platforms(like Twitter, Facebook, and Instagram). The system should allow users to see a feed of recent updates from their friends or followed accounts, display posts, and handle high traffic and large amounts of data.


## Requirements
### Functional Requirements
* Feed publishing: Users should be able to create/update posts.(text, images, videos).
* Users should be able to friend/follow people.
* Newsfeed building: Users should be able to view a feed of posts from people they follow, in chronological order.
* Users should be able to page through their feed.
* The feed should be personalized and ordered by relevance and recency.

#### Below the line (out of scope):
* Users should be able to like and comment on posts.
* Posts can be private or have restricted visibility.

### Non-Functional Requirements
* **Availability(AP):** The system should be highly available (prioritizing availability over consistency). 
* **Consistency**: Ensure eventual consistency in the feed updates. Tolerate up to 2 minutes for eventual consistency.
* **Performance**: Posting and viewing the feed should be fast, returning in < 500ms.
* **Scalability**
  * The system should be able to handle a massive number of users (2B).
  * Users should be able to follow an unlimited number of users, users should be able to be followed by an unlimited number of users.
* **Fault Tolerance**: The system should gracefully handle server failures.
* **Security**: Protect user data and ensure privacy settings are respected.

## Back of Envelope Estimations/Capacity Estimation & Constraints
* User Base: 2 billion  users
* Active Users: 20M DAU
* Reads per User per Day: 50 (feed loads)
* Average Posts per User per Day: 2
* R:W Ratio = 25:1

### Traffic Estimates
* Read Requests per Second (RPS): 20 million daily active users * 50 reads/day / 86400 seconds/day ~ 5000
* Write Requests per Second (RPS): 20 million daily active users * 2 posts/day / 86400 seconds/day ~ 200
### Storage Estimates
* Each post (including metadata) ~ 1 KB
* Total posts per day = 20 million users * 2 posts = 40 million posts
* Storage per day = 40 million * 1 KB = 40 GB/day

## High-level API design 
### Post APIs
* Create a new POST
```
POST /posts
Request {
    auth_token
    content:{

    }
}
Response {
    "postId": // ...
}
```
* Get post details
```
GET /posts/{id}
Request {
    auth_token: <<text>>
    id: <<text>>
}
Response {
    "postId": // ...
}
```

### Feed APIs
* Get the feed for a user
```
GET /feed
Request {
    auth_token: <<text>>
    max_items: <<int>>
    next_token: <<text>>
}
Response: 
{
    items: POST[{

    }]
}
```
## Database Design

### UserDB (PostgreSQL)

**Users Table**

| Field      | Type      | Description              |
|------------|-----------|--------------------------|
| user_id    | VARCHAR   | Primary Key              |
| name       | VARCHAR   |                          |
| email      | VARCHAR   | Unique                   |
| created_at | TIMESTAMP | Default: CURRENT_TIMESTAMP |
| updated_at | TIMESTAMP | Default: CURRENT_TIMESTAMP |

### PostDB (DynamoDB)

**Posts Table**

| Attribute  | Type   | Description                           |
|------------|--------|---------------------------------------|
| user_id    | String | Partition Key (Primary Key)           |
| timestamp  | String | Sort Key (ISO 8601 timestamp format)  |
| post_id    | String | Unique Identifier for the post        |
| content    | String | Content of the post                   |
| media_url  | String | URL of the media associated with post |
| created_at | String | ISO 8601 timestamp format             |
| updated_at | String | ISO 8601 timestamp format             |


### FollowersDB (Neo4j)

**User Node**

| Property   | Type    | Description |
|------------|---------|-------------|
| user_id    | STRING  | Primary Key |
| name       | STRING  |             |
| email      | STRING  |             |
| created_at | INTEGER | Timestamp   |

**FOLLOWS Relationship**

| Property   | Type    | Description |
|------------|---------|-------------|
| since      | INTEGER | Timestamp   |

### FeedDB (Cassandra)

**Feeds Table**

| Column     | Type       | Description                              |
|------------|------------|------------------------------------------|
| user_id    | TEXT       | Primary Key                              |
| feed_items | LIST<FROZEN<TUPLE<TEXT, TEXT>>> | List of (friend_id, post_id) |
| updated_at | TIMESTAMP  | Last updated timestamp                   |

### Summary

- **UserDB (PostgreSQL)**: Manages user data with strong ACID properties and complex queries.
- **PostDB (DynamoDB)**: Stores posts with `user_id` as the partition key and `timestamp` as the sort key for efficient querying by user and time.
- **FollowersDB (Neo4j)**: Handles follow relationships efficiently using graph-based data modeling.
- **FeedDB (Cassandra)**: Manages user feeds with high availability, fault tolerance, and efficient read/write operations, including friend_id along with post_id in feed_items.

## High Level System Design 

![](../resources/problems/news_feed/news_feed_system.png)

### [Feed Generation/Publishing] Creating a Post
* When a user creates a post, it is written to a database table for Posts.
* A change data capture (CDC) system like Kafka captures this insert and pushes the new post to a stream processing system.

### [Feed Generation/Publishing] Consuming the Post Stream - Fanout Service 
* The stream processor consumes the post stream from Kafka.
* For each new post, it looks up the author's followers from the Follow GraphsDB .
* It fanouts the new post to each follower's feed in a distributed feed cache like Redis.

### [Feed Retrieval] Retrieving the User's Feed
* When a user requests their feed, the application server queries the Redis feed cache for that user's feed data.
* If user is following any celebrity whose posts can be sync in feed then those latest posts are query from post db.
* The feed posts are rendered on the client with infinite scrolling.
* As the user scrolls, more posts are fetched from Redis via lazy loading.


### Feed Ranking
* The posts in the feed can be ranked based on factors like:
    * Closeness of relationship between user and post author
    * Post recency and time decay
    * Post engagement (likes, comments, etc.)
    * Machine learning models for personalized ranking
* Other critical components are a robust caching layer, content delivery networks for serving media, sharding and replication of databases and caches to scale.
* The key principles are separating read/write paths, using caching extensively, pre-computing/ranking feeds for optimized performance, and designing for horizontal scalability from the ground up.

## Deep Dive

### Handle users who are following a large number of users vs Handle users with a large number of followers?
* Handle users who are following a large number of users - Fanout on Write (Push Model)
* Handle users with a large number of followers - Fanout on Read (Pull Model)
* Go with Hyrbid Approach

#### Fanout Service Models: Summary and Hybrid Approach

**Fanout on Write (Push Model)**
- **Workflow**: Pre-compute and deliver news feed during write time.
- **Pros**:
  - Real-time updates, immediate push to friends.
  - Fast fetching as news feed is pre-computed.
- **Cons**:
  - Hotkey problem: Resource-intensive for users with many friends.
  - Wasted resources for inactive users.

**Fanout on Read (Pull Model)**
- **Workflow**: Generate news feed during read time, pulling recent posts when a user loads their home page.
- **Pros**:
  - Efficient for inactive users, no wasted resources.
  - No hotkey problem.
- **Cons**:
  - Slower fetching as news feed is not pre-computed.

**Hybrid Approach**
- **Strategy**: Combine both models for optimal performance.
- **Implementation**:
  - **Async Workers**: Handle post creates via a shared queue for precomputed feeds.
  - **Selective Pre-computation**:  For most users, precompute feeds to ensure fast fetching. For users with many followers (e.g., celebrities), use a flag in the Follow table to avoid precomputing feeds for these high-follow accounts.
  - **On-demand Merging**: Merge precomputed feeds with recent posts for non-precomputed accounts when fetching the feed.
- **Pros**:
  - Optimized performance with precomputed feeds for most users.
  - Efficient handling of high-follow accounts, avoiding overload.
  - Balanced system load.
- **Cons**:
  - Slightly more complex design and implementation.
  - Requires careful management of flags and merging logic.

**Tedious Hybrid Approach**
* In this approach once a user publishes a post; we can limit the fanout to only their online friends.
* Also, to get benefits of both the approaches, a combination of push to notify and pull for serving end users is a great way to go.
* Purely push or pull model is less versatile.


This hybrid approach leverages the strengths of both fanout on write and fanout on read, addressing their challenges for a scalable and responsive news feed system.

## References
* Alex Wu - Vol1 - Chapter 11
* https://vipulpachauri12.medium.com/news-feed-high-level-system-design-small-and-crisp-3caea8ab3b32
* https://www.hellointerview.com/learn/system-design/answer-keys/fb-news-feed
* https://medium.com/@tahir.rauf/system-design-news-feed-9dcbc36b9580
* https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/design-of-a-newsfeed-system
* https://astikanand.github.io/techblogs/high-level-system-design/design-facebook-newsfeed
* https://excalidraw.com/#json=cwPL5aZRK8_9LtYKA9fJH,KHLGvd2KgD4h_h99JaBUAw