## Problem Statement

## Requirements
### Functional Requirements
* **Post photos and videos**: The users can post photos and videos on Instagram.
* **Follow and unfollow users**: The users can follow and unfollow other users on Instagram.
* **Like or dislike posts**: The users can like or dislike posts of the accounts they follow.
* **Search photos and videos**: The users can search photos and videos based on captions and location.
* **Generate news feed**: The users can view the news feed consisting of the photos and videos (in chronological order) from all the users they follow. Users can also view suggested and promoted photos in their news feed.
### Non-Functional Requirements
* **Scalability**: The system should be scalable to handle millions of users in terms of computational resources and storage.
* **Latency**: The latency to generate a news feed should be low.
* **Availability**: The system should be highly available.
* **Durability** Any uploaded content (photos and videos) should never get lost.
* **Consistency**: We can compromise a little on consistency. It is acceptable if the content (photos or videos) takes time to show in followers’ feeds located in a distant region.
* **Reliability**: The system must be able to tolerate hardware and software failures.

## Back of Envelope Estimations/Capacity Estimation & Constraints
## High-level API design 
* Post photos or videos
``postMedia(userID, media_type, list_of_hashtags, caption)``
* Follow and unfollow users
``followUser(userID, target_userID)``
* Like or dislike posts
``likePost(userID, post_id)``
* Search photos or videos
``searchPhotos(userID, keyword)``
* Generate news feed
``viewNewsfeed(userID, generate_timeline)``
## Database Design
## High Level System Design and Algorithm
## References
* https://leetcode.com/discuss/interview-question/1490932/Instagram-System-Design, https://drive.google.com/file/d/1VRwvbijCV_oM4fKljdcmLb0AJv9S38y-/view
* https://read.learnyard.com/hld-instagram-system-design/
* https://github.com/codekarle/system-design/blob/master/system-design-prep-material/architecture-diagrams/Facebook%20System%20Design.png
* https://nikhilgupta1.medium.com/instagram-system-design-f62772649f90
* https://www.linkedin.com/posts/rocky-bhatia-a4801010_instagram-system-design-designing-a-system-activity-7132334932504289280-U5hQ/
* https://www.youtube.com/watch?v=YoS5cp0cirM
