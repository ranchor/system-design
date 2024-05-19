## Problem Statement
Design a distributed system for an online ticket booking platform like Ticketmaster, which can handle a high volume of traffic and provide a seamless user experience for searching, reserving, and purchasing tickets for various events.

## Requirements
### Functional Requirements
* **Book Tickets:** Users should be able to book tickets to events
* **View Events:** Users should be able to view events
* **Search Events:** Users should be able to search for events

### Non-Functional Requirements
* **Low latency**: Low latency search.
* **Availability**: System should prioritize availability for searching & viewing events
* **Consistency**: System should prioritize strong consistency for booking events (no double booking) 
* System is read heavy, and thus needs to be able to support high read throughput
* **Scalability**: Scalability to handle surges from popular events
* **Security**: Purchase path should be secure. Encrypt data in transist & data in rest.

### Out of scope
* GDPR Compilance
* Fault Tolerance
* Backups
* CI/CD



## Back of Envelope Estimations/Capacity Estimation & Constraints
## High-level API design 
## Database Design
## High Level System Design and Algorithm
## References
* https://www.hellointerview.com/learn/system-design/answer-keys/ticketmaster