## Problem Statement
Design a system that provides real-time analytics on the most popular (Top-K) YouTube videos based on the number of views. The system should handle high traffic and provide accurate results with low latency.

## Clarification Questions to Interviewer 

## Requirements
### Functional Requirements
* Clients should be able to query the top K videos (max 1000) for a given time period.
* Time periods should be limited to 1 {hour, day, month} and all-time.
#### Below the line (out of scope)
* Arbitrary time periods.
* Arbitrary starting/ending points (we'll assume all queries are looking back from the current moment).
### Non-Functional Requirements
- **Scalability**: Support a peak of millions of views per second.
- **Low Latency**: Provide real-time analytics with sub-second response time.
- **Fault Tolerance**: Ensure no data loss and accurate data collection.
- **Real-time Processing**: Data should be available for querying as soon as possible.
- **Consistency**: Ensure consistency in the view counts across different replicas.
#### Below the line (out of scope)

## Back of Envelope Estimations/Capacity Estimation & Constraints
## High-level API design 
## Data Model
## High Level System Design
## Deep Dive
## References
* https://www.hellointerview.com/learn/system-design/answer-keys/top-k