## Problem Statement
Design a scalable and efficient mapping service similar to Google Maps, providing users with accurate navigation, real-time traffic updates, and location-based services.

## Requirements
### Functional Requirements
- **Maps**: Display maps with various layers such as satellite view, terrain view, and traffic view.
- **Navigation**: Provide turn-by-turn navigation instructions from a starting point to a destination, including multiple routes based on traffic conditions.
- **Search**: Allow users to search for places, addresses, and points of interest (POIs) on the map.
- **Location Sharing**: Enable users to share their real-time location with others and receive location updates.
- **Traffic Updates**: Provide real-time traffic updates, congestion alerts, and alternative routes to optimize travel time.
- **Geocoding/Reverse Geocoding**: Convert addresses into geographic coordinates (latitude and longitude) and vice versa.
- **Route Planning**: Plan routes based on various criteria such as shortest distance, fastest route, and avoidance of tolls or highways.

### Non-Functional Requirements
- **Scalability**: The system should handle millions of concurrent users and scale horizontally to accommodate increasing demand.
- **Reliability**: Ensure high availability and reliability to minimize downtime and service disruptions.
- **Performance**: Deliver fast response times for map rendering, navigation calculations, and search queries.
- **Accuracy**: Provide accurate and up-to-date map data, navigation instructions, and traffic information.
- **Data and battery usage** - Client should use as little data and battery as possible. Important for mobile devices.
- **Smooth navigation** - Users should experience smooth map rendering
- **Security**: Implement robust security measures to protect user data, including encryption of sensitive information and authentication mechanisms.
- **Offline Support**: Allow users to access maps and navigation features offline by caching data locally on their devices.

## Back of Envelope Estimations/Capacity Estimation & Constraints
## High-level API design 
## Database Design
## High Level System Design
## Deep Dive
## References
* https://github.com/preslavmihaylov/booknotes/tree/master/system-design/system-design-interview/chapter19
* https://github.com/codekarle/system-design/blob/master/system-design-prep-material/architecture-diagrams/Google%20Maps%20Design.png
* https://experiencestack.co/google-maps-system-design-84174a1e23de