## Problem Statement
Design a price drop tracker system that monitors the prices of products from amazon and notifies users when the price of a product they are interested in drops below a certain threshold. Very similar to camelcamelcamel.com


## Requirements
### Functional Requirements
* **User Management**: Allow users to create accounts, log in, and manage their profile.
* **Product Catalog**: Allow users to search for products and add them to their watchlist.
* **Price Monitoring**: Track prices of products and notify users when the price drops below their specified threshold.

### Non-Functional Requirements
* **Scalability**: The system should handle a large number of products and users.
* **Performance**: Price checks and notifications should be performed efficiently.
* **Reliability**: Ensure high availability and fault tolerance.


## Back of Envelope Estimations/Capacity Estimation & Constraints
1. **Number of Users**: Assume 1 million users.
2. **Number of Products per User**: Assume each user tracks 10 products on average.
3. **Total Products Tracked**: 10 million products.
4. **Price Check Frequency**: Assume price checks are performed every hour.
5. **Notifications per Day**: Assume 1% of products experience price drops daily. This results in 100,000 notifications per day.


## High-level API design 
## Data Model
### User Table
**Database Type**: Relational (e.g., PostgreSQL)
| Field      | Type      | Description                  |
|------------|-----------|------------------------------|
| user_id    | String (PK)| Unique identifier for the user |
| name       | String    | Name of the user             |
| email      | String    | Email address of the user    |
| password   | String    | Hashed password              |

### Product Table
**Database Type**: NoSQL (e.g., DynamoDB) for scalability and flexibility
| Field       | Type      | Description                      |
|-------------|-----------|----------------------------------|
| product_id  | String (PK)| Unique identifier for the product |
| name        | String    | Name of the product              |
| url         | String    | URL of the product page          |
| current_price | Float   | Current price of the product     |

### Watchlist Table
**Database Type**: NoSQL (e.g., DynamoDB)
| Field       | Type      | Description                      |
|-------------|-----------|----------------------------------|
| user_id     | String (PK)| Unique identifier for the user   |
| product_id  | String (PK)| Unique identifier for the product|
| target_price | Float    | Desired price set by the user    |

### Notification Table
**Database Type**: NoSQL (e.g., DynamoDB)
| Field       | Type      | Description                      |
|-------------|-----------|----------------------------------|
| notification_id | String (PK)| Unique identifier for the notification |
| user_id     | String    | Unique identifier for the user   |
| product_id  | String    | Unique identifier for the product|
| message     | String    | Notification message             |
| timestamp   | DateTime  | Time when the notification was sent|

## High Level System Design
### Workflow Explanation
1. **User Management**:
   - Users interact with the system through a web or mobile app.
   - Users can create accounts, log in, and manage their profile.

2. **Product Catalog**:
   - Users search for products on Amazon and add them to their watchlist through the app.
   - The app sends requests to the backend, which interacts with the Product Price DB (DynamoDB) to store product information.

3. **Price Monitoring**:
   - **Data Scrapers** periodically scrape Amazon for product prices and update the Product Price DB.
   - **Stale Price Sweeper** ensures that only up-to-date product prices are stored.
   - **Price Drop Service** monitors price changes using DDB Streams and updates the Price Watch DB.
   - When a price drop is detected, the Price Drop Service sends a notification to the Notification Queue.

4. **Notification Service**:
   - Processes notifications from the Notification Queue.
   - Sends notifications to users via email, SMS, or app notifications based on user preferences.
   - Logs notifications in the Notification DB.

![](../resources/problems/price_tracker/price_alert.png)
## Deep Dive
### Price Monitoring
- **Data Scrapers**: Scrapers fetch product prices from Amazon and update the Product Price DB.
- **Stale Price Sweeper**: Periodically checks for outdated prices and removes or updates them.
- **Price Drop Service**: Listens to changes in the Product Price DB via DynamoDB Streams and triggers notifications when price drops are detected.

### User Notification Preferences
- Users can set their preferred notification channels (email, SMS, push notifications) in their profile.
- The Notification Service respects these preferences when sending notifications.

### Scalability and Reliability
- Use DynamoDB for scalable and flexible storage of product and watchlist data.
- Use SQS for reliable and scalable queuing of notifications.
- Implement monitoring and alerting to ensure the system's health and performance.
## References
* https://systemdesignfightclub.com/price-drop-tracker/
* https://blog.crushingtecheducation.com/p/design-a-price-drop-tracker-like