## Problem Statement
Design a system that provides real-time analytics on the most popular (Top-K) YouTube videos based on the number of views. The system should handle high traffic and provide accurate results with low latency.

## Clarification Questions to Interviewer
1. What is the maximum number of concurrent users we should support?
2. Are there any specific geographical regions that we need to prioritize?
3. Should the system provide a breakdown of views by region?
4. Are there any constraints on the database technology or architecture?
5. What is the expected peak traffic in terms of views per second?
6. Is there a requirement for historical data storage and retrieval?

## Requirements
### Functional Requirements
- Clients should be able to query the top K videos (max 1000) for a given time period.
- Time periods should be limited to 1 {hour, day, month} and all-time.

#### Below the line (out of scope)
- Arbitrary time periods.
- Arbitrary starting/ending points (we'll assume all queries are looking back from the current moment).

### Non-Functional Requirements
- **Scalability**: Support a peak of millions of views per second.
- **Low Latency**: Provide real-time analytics with sub-second response time.
- **Fault Tolerance**: Ensure no data loss and accurate data collection.
- **Real-time Processing**: Data should be available for querying as soon as possible.
- **Consistency**: Ensure consistency in the view counts across different replicas.

#### Below the line (out of scope)
- Details of the user interface.
- Complex filtering and sorting beyond the specified requirements.

## Back of Envelope Estimations/Capacity Estimation & Constraints
- **Views per second**: Assume peak traffic of 1 million views per second.
- **Storage**: Assuming each view record is 100 bytes, we need approximately 100 MB per second for raw view data.
- **Daily Storage**: Approximately 8.64 TB per day.
- **Query Frequency**: High frequency, real-time queries for top K videos.

## High-level API design
- **Get Top K Videos**:
  ```http
  GET /v1/videos/top
  ```
  **Query parameters**:
  - `period` - time period (e.g., `hour`, `day`, `month`, `all-time`).
  - `k` - number of top videos (e.g., 100, 500, 1000).

  **Response**:
  ```json
  {
    "period": "string",
    "top_videos": [
      {
        "video_id": "string",
        "view_count": "integer"
      },
      ...
    ]
  }
  ```

## Data Model
### VideoViewEvent
| Field              | Type      |
|--------------------|-----------|
| `video_id`         | String    |
| `view_timestamp`   | Timestamp |
| `user_id`          | String    |
| `region`           | String    |
| `device`           | String    |

### VideoViewCount
| Field              | Type      |
|--------------------|-----------|
| `video_id`         | String    |
| `period`           | String    |
| `start_time`       | Timestamp |
| `end_time`         | Timestamp |
| `view_count`       | Integer   |

### OLAP Schema for Top-K Videos

#### Minute-level Aggregates
| Field        | Type      |
|--------------|-----------|
| `video_id`   | String    |
| `date`       | Date      |
| `hour`       | Integer   |
| `minute`     | Integer   |
| `view_count` | Integer   |

#### Hour-level Aggregates
| Field        | Type      |
|--------------|-----------|
| `video_id`   | String    |
| `date`       | Date      |
| `hour`       | Integer   |
| `view_count` | Integer   |

#### Day-level Aggregates
| Field        | Type      |
|--------------|-----------|
| `video_id`   | String    |
| `date`       | Date      |
| `view_count` | Integer   |

#### TopKVideos
| Field        | Type      | Description                                 |
|--------------|-----------|---------------------------------------------|
| `period`     | String    | Time period (e.g., `hour`, `day`, `month`)  |
| `start_time` | Timestamp | Start time of the period                    |
| `end_time`   | Timestamp | End time of the period                      |
| `video_id`   | String    | ID of the video                             |
| `view_count` | Integer   | Number of views in the given period         |
| `rank`       | Integer   | Rank of the video in the Top-K list         |

### Data Example in OLAP for Querying

#### Minute-level Aggregates
| video_id | date       | hour | minute | view_count |
|----------|------------|------|--------|------------|
| vid123   | 2024-07-15 | 14   | 30     | 150        |
| vid456   | 2024-07-15 | 14   | 30     | 120        |

#### Hour-level Aggregates
| video_id | date       | hour | view_count |
|----------|------------|------|------------|
| vid123   | 2024-07-15 | 14   | 900        |
| vid456   | 2024-07-15 | 14   | 720        |

#### Day-level Aggregates
| video_id | date       | view_count |
|----------|------------|------------|
| vid123   | 2024-07-15 | 14400      |
| vid456   | 2024-07-15 | 11520      |

#### TopKVideos Table
| period | start_time          | end_time            | video_id | view_count | rank |
|--------|---------------------|---------------------|----------|------------|------|
| hour   | 2024-07-15 14:00:00 | 2024-07-15 15:00:00 | vid123   | 150        | 1    |
| hour   | 2024-07-15 14:00:00 | 2024-07-15 15:00:00 | vid456   | 120        | 2    |
| day    | 2024-07-15 00:00:00 | 2024-07-16 00:00:00 | vid123   | 900        | 1    |
| day    | 2024-07-15 00:00:00 | 2024-07-16 00:00:00 | vid456   | 720        | 2    |
| month  | 2024-07-01 00:00:00 | 2024-08-01 00:00:00 | vid123   | 14400      | 1    |
| month  | 2024-07-01 00:00:00 | 2024-08-01 00:00:00 | vid456   | 11520      | 2    |


## High Level System Design
1. **Data Ingestion**: Use Kafka to handle incoming view events from multiple sources.
2. **Real-time Processing**: Use Apache Flink for real-time aggregation and processing of view counts.
3. **Storage**: Store aggregated view counts in a scalable NoSQL database like Cassandra.
4. **Query Service**: Implement a query service to fetch top K videos for the specified time period from the NoSQL database.
5. **Caching Layer**: Use Redis to cache frequently requested top K results to reduce load on the database.


## Deep Dive
### Time
- **Event time**: When the view occurs.
- **Processing time**: When the server processes the event.
Using both timestamps helps ensure accuracy despite network delays.

### Aggregation Window
- **Tumbling (fixed) window**: For aggregating views by hour, day, month.
- **Sliding window**: For querying top K videos in real-time.

### Real-time Processing with Apache Flink
- **Ingestion**: Kafka captures view events.
- **Stream Processing**: Flink processes events, updates view counts.
- **State Management**: Flink manages state using RocksDB, taking periodic snapshots.

### Top-K Computation
- Use Flink's built-in Top-K function to maintain and update the top K videos.
- Periodically output the top K results to Cassandra.

### Fault Tolerance
- **Kafka**: Data persistence in Kafka ensures no data loss.
- **Flink**: Checkpointing and state snapshots ensure recovery.

### Querying Top-K Videos
- **API Service**: Fetch top K videos from Redis cache.
- **Cache Miss**: If not in cache, query Cassandra, then update Redis cache.

### Pre-Aggregation with MapReduce
- **Periodic Jobs**: MapReduce jobs for hourly, daily, monthly aggregations.
- **Storage**: Store results in Cassandra for quick retrieval.

## References
* https://www.hellointerview.com/learn/system-design/answer-keys/top-k
* https://medium.com/@techturtles51/system-design-interview-top-k-heavy-hitters-ab49640f3ec9
* https://medium.com/@ishwarya1011.hidkimath/system-design-top-k-songs-played-on-music-streaming-applications-12d7104834d2