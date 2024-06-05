## Problem Statement
Design a scalable and efficient system for a ride-hailing service similar to Uber. This system should allow users to request rides, match them with drivers, track ride status in real-time, handle payments, and ensure a seamless user experience.

## Requirements
### Functional Requirements
* **Ride Fare Estimation**: Riders should be able to input their current location and desired destination to see an estimated fare.
* **Real-time Ride Matching**: Riders should be able to request a ride and be matched with a nearby available driver in real-time.
* **Driver Navigation**: Drivers should be able to accept ride requests and navigate to the user's location and destination.

#### Below the line (out of scope)
- **Rating System**:
  - Riders should be able to rate their ride and driver post-trip.
  - Drivers should be able to rate passengers.
- **Ride Scheduling**:
  - Riders should be able to schedule rides in advance.
- **Ride Categories**:
  - Riders should be able to request different categories of rides (e.g., X, XL, Comfort).
- **Admin Dashboard**: Provide an admin dashboard for monitoring system health, managing users, and resolving disputes.
- **Payment Processing**: Handle payment processing securely and efficiently.

### Non-Functional Requirements
- **Low Latency**: The system should prioritize low latency for ride matching to ensure quick response times for users and drivers.
- **High Availability**: The system should be highly available and reliable, minimizing downtime and ensuring that ride requests can be processed 24/7.
- **Strong/High Consistency**: The system should ensure strong consistency in ride matching to prevent any driver from being assigned multiple rides simultaneously.
- **High Scalability**: The system should be able to handle high throughput, especially during peak hours or special events.
- **Security**: Implement robust security measures to protect user data, including encryption of sensitive information and authentication mechanisms.

#### Below the line (out of scope)
- **Resilience**: The system should be resilient to failures, with redundancy and failover mechanisms in place.
- **Monitoring and Logging**: The system should have robust monitoring, logging, and alerting to quickly identify and resolve issues.
- **Updates and Maintenance**: The system should facilitate easy updates and maintenance without significant downtime (CI/CD pipelines).

## Back of Envelope Estimations/Capacity Estimation & Constraints
## High-level API design 
## Database Design
## High Level System Design
## Deep Dive
## References
* https://www.hellointerview.com/learn/system-design/answer-keys/uber
* https://medium.com/@karan99/system-design-uber-33593137a4fe
* Code Karle
  * https://www.youtube.com/watch?v=Tp8kpMe-ZKw
  * https://www.codekarle.com/system-design/Uber-system-design.html
  * https://github.com/codekarle/system-design/blob/master/system-design-prep-material/architecture-diagrams/Uber%20System%20Design.png
* https://medium.com/towards-data-science/ace-the-system-design-interview-uber-lyft-7e4c212734b3  