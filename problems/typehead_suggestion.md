## Problem Statement

## Requirements
### Functional Requirements
* The system should return search suggestions as the user types in the search box.
* The system should return top 10 results for each query.
* The system should prioritize more relevant suggestions (e.g., based on popularity or other relevance metrics).
### Non-Functional Requirements
* Low latency
* Fault tolerance
* Scalability

## Back of Envelope Estimations/Capacity Estimation & Constraints
- **Traffic estimation**
   - Number of searches per day = 5 billion (Assumed)
   - Number of searches per second (QPS) = Number of searches per day / 24 hours / 3600 seconds = 60000 times/s
- **Storage estimation**
   - Types
      - Data: Yes
      - File: No
   - Capacity
      - Number of terms need to build an index
         - 5 billion searches per day.
         - Only 20% of these will be unique. (Assumed)
         - We only want to index the top 50% of the search terms. (Assumed)
         - Number of terms we need to build an index per day = 5 billion x 20% x 50% = 5 million
      - Size for storing the index
         - Each search has 3 words.
         - Each words has 5 characters.
         - Each character needs 2 bytes
         - Total size for storing the index per day = 5 million x 3 x 5 x 2 = 15 GB

## High-level API design 
* Get suggestion
```
GET /suggestions?q={search-term}
```
Response should include a list of suggested terms, ordered by relevance:
```
{
    "suggestions": ["suggestion1", "suggestion2", ..., "suggestionn"]
}
```
* Add trending queries to the database
```
addToDatabase(query)
```

## Database Design
## High Level System Design and Algorithm
## References