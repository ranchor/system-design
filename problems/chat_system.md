## Problem Statement

## Requirements
### Functional Requirements
* One-one chat
* Group chat
* Read receipt
* Online status
* Push notifications
* share multimedia
* Multi device support
### Non-Functional Requirements
* **Low latency**: Users should be able to receive messages with low latency.
* **Consistency**: Messages should be delivered in the order they were sent. Also, users must see the same chat history on all of their devices.
* **Availability**: The system should be highly available. However, consistency is more important than availability.
* **Security**: The system must be secure via end-to-end encryption. We need to ensure that only the communicating parties can see the content of messages. Nobody in between, not even we as service owners, should have access.
* **Scalability**: The system should be highly scalable to support an increasing number of users and messages per day.

## Back of Envelope Estimations/Capacity Estimation & Constraints

## High-level API design 
## Database Design
## High Level System Design and Algorithm
## References
* Alex Wu - Vol1 - [Chapter 12](https://bytebytego.com/courses/system-design-interview/design-a-chat-system)
* https://medium.com/@m.romaniiuk/system-design-chat-application-1d6fbf21b372
* https://leetcode.com/discuss/study-guide/2066150/chat-system-design-complete-guide
* https://www.geeksforgeeks.org/designing-whatsapp-messenger-system-design/
* https://systemdesignprep.com/newsfeed