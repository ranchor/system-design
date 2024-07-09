### Problem Statement

Design a system like Google Docs that allows multiple users to collaboratively create, edit, and view documents in real-time.

### Clarification Questions to Interviewer

1. What types of documents will the system support (text, spreadsheets, presentations)?
2. Should the system support offline editing and subsequent synchronization?
3. What level of user permissions and access control is required (view, edit, comment)?
4. Are there any specific security requirements for data storage and transmission?
5. What are the expectations for real-time collaboration? How responsive should the system be?
6. Are there requirements for version control and history tracking of document changes?
7. Should the system support integrations with other services or platforms?
8. What is the expected scale in terms of the number of users and documents?

### Requirements

#### Functional Requirements
- **Document Creation**: Users can create, edit, and delete documents.
- **Real-Time Collaboration**: Multiple users can collaborate on a document in real-time.
- **Visibility of Changes**: Users can see changes made by others in real-time.
- **Version Control**: Track document history and changes over time.
- **Access Control**: Manage user permissions (view, edit, comment).
- **Offline Support**: Allow offline editing and synchronize when back online.
- **Commenting**: Support comments and suggestions within documents.
- **Integration**: Integrate with third-party applications (optional).

#### Below the line (out of scope)
- **Advanced Formatting**: Advanced formatting features like those in desktop word processors.
- **Non-Essential Integrations**: Integration with non-essential services initially.
- **Custom Plugins**: Custom plugins or extensions.

#### Non-Functional Requirements
- **High Availability**: Ensure the system is highly available.
- **Low Latency**: Maintain low latency for real-time collaboration.
- **Scalability**: Scale to handle a large number of concurrent users and documents.
- **Data Consistency**: Ensure strong consistency of document data.
- **Security**: Ensure the security and privacy of documents.

#### Below the line (out of scope)
- **Geographic Redundancy**: Geographic redundancy (unless specified).
- **Multilingual Support**: Multilingual support.

### Back of Envelope Estimations/Capacity Estimation & Constraints

- Assume 1 million active users.
- Average document size: 100 KB.
- Peak concurrent users: 10% of active users (100,000 users).
- Each user makes an average of 10 edits per minute.
- Total documents: 10 million.
- Storage required: 10 million * 100 KB = ~1 TB (excluding version history and metadata).

### High-level API Design

#### Document Management
- `POST /documents`: Create a new document.
- `GET /documents/{docId}`: Retrieve a document.
- `PUT /documents/{docId}`: Update a document.
- `DELETE /documents/{docId}`: Delete a document.
- `GET /documents/{docId}/history`: Retrieve document history.

#### Collaboration
- `POST /documents/{docId}/collaborators`: Add a collaborator.
- `DELETE /documents/{docId}/collaborators/{userId}`: Remove a collaborator.
- `GET /documents/{docId}/collaborators`: List collaborators.

#### Real-time Editing
- `POST /documents/{docId}/changes`: Submit changes.
- `GET /documents/{docId}/changes`: Retrieve changes for synchronization.

#### Comments
- `POST /documents/{docId}/comments`: Add a comment.
- `GET /documents/{docId}/comments`: Retrieve comments.
- `DELETE /documents/{docId}/comments/{commentId}`: Delete a comment.

### Data Model

#### Document
- `docId`: String (UUID)
- `title`: String
- `content`: Text (or structured content format)
- `ownerId`: String (User ID)
- `createdAt`: Timestamp
- `updatedAt`: Timestamp

#### User
- `userId`: String (UUID)
- `username`: String
- `email`: String
- `permissions`: Map<docId, PermissionLevel>

#### Change
- `changeId`: String (UUID)
- `docId`: String (Document ID)
- `userId`: String (User ID)
- `timestamp`: Timestamp
- `changeData`: JSON (diff or patch)

#### Comment
- `commentId`: String (UUID)
- `docId`: String (Document ID)
- `userId`: String (User ID)
- `timestamp`: Timestamp
- `content`: Text

### High Level System Design

1. **Client-Side Application**: Web and mobile applications for document editing and collaboration.
2. **API Gateway**: Handles routing, authentication, and rate limiting.
3. **Document Service**: Manages CRUD operations for documents.
4. **Real-Time Collaboration Service**: Handles real-time editing and synchronization.
5. **Change Log Service**: Stores and manages changes for version control.
6. **User Service**: Manages user information and permissions.
7. **Notification Service**: Manages real-time updates and notifications to clients.
8. **Storage**: Document storage using databases and object storage.
9. **Authentication and Authorization**: Manages user access and permissions.

### Deep Dive

#### Conflict Resolution Techniques

##### Operational Transformation (OT)

Operational Transformation is a technique used in collaborative editing to ensure consistency among multiple users concurrently editing a document.

- **Concept**: Each user's edit is transformed relative to other users' edits to ensure the document remains consistent.
- **Mechanism**:
  1. When a user makes an edit, it is sent to the server.
  2. The server transforms the edit against any concurrent edits from other users.
  3. The transformed edit is then applied to the document and broadcast to all users.
  4. Users apply the transformed edit to their local copy of the document.
- **Example**: Imagine two users, Alice and Bob, editing the same document. Initially, the document contains the text "Hello".
  - Alice's Action: Alice deletes "H".
  - Bob's Action: Bob inserts "Y" at the beginning.
  - Without OT, applying Alice's and Bob's changes in any order might lead to inconsistency. With OT, the changes are transformed to maintain consistency.
  - Initial State: "Hello"
  - Alice's Edit: "ello" (delete "H")
  - Bob's Edit: "YHello" (insert "Y" at index 0)
  - Using OT: Apply Bob's insertion after Alice's deletion. Transformed State: "Yello" (Bob's insertion is applied to Alice's result)
- **Advantages**: Maintains consistency across all users. Allows for real-time collaboration with minimal latency.
- **Challenges**: Complex to implement, especially with multiple types of operations. Requires careful handling of edge cases and conflict scenarios.

##### Conflict-free Replicated Data Types (CRDT)

CRDTs are data structures that allow for conflict-free merging of concurrent updates, making them ideal for distributed systems.

- **Concept**: CRDTs ensure that all replicas of a document converge to the same state, regardless of the order of operations.
- **Types**:
  - G-Counter: A grow-only counter.
  - PN-Counter: A counter that can both increment and decrement.
  - G-Set: A grow-only set.
  - OR-Set: An observed-remove set.
- **Mechanism**:
  1. Each replica independently applies updates to its local state.
  2. Updates are periodically exchanged between replicas.
  3. Each replica merges updates in a conflict-free manner.
- **Advantages**: Simplifies reasoning about concurrency. Naturally supports offline editing and synchronization.
- **Challenges**: Can be more complex to design for certain data types. Typically results in higher storage overhead.

##### Differential Synchronization (DS)

Differential Synchronization is an algorithm designed to keep multiple copies of a document in sync, even in the presence of conflicting edits.

- **Concept**: Periodically synchronize differences between document copies to ensure consistency.
- **Mechanism**:
  1. Each client maintains a shadow copy of the document.
  2. Edits are applied to the shadow copy.
  3. Periodically, the client computes the difference between the local copy and the shadow copy and sends the diff to the server.
  4. The server applies the diff to the master copy and sends back the diff between the master copy and the shadow copy.
  5. The client applies the server's diff to its local copy and shadow copy.
- **Advantages**: Reduces the amount of data sent over the network by only sending diffs. Handles conflicts by merging diffs.
- **Challenges**: Requires efficient diff computation. Potentially complex to handle merge conflicts correctly.

### Example of Operational Transformation

Imagine two users, Alice and Bob, editing the same document. Initially, the document contains the text "Hello".

1. **Alice's Action**: Alice deletes "H".
2. **Bob's Action**: Bob inserts "Y" at the beginning.

Without OT, applying Alice's and Bob's changes in any order might lead to inconsistency. With OT, the changes are transformed to maintain consistency.

- **Initial State**: "Hello"
- **Alice's Edit**: "ello" (delete "H")
- **Bob's Edit**: "YHello" (insert "Y" at index 0)

Using OT:
- Apply Bob's insertion after Alice's deletion.
- Transformed State: "Yello" (Bob's insertion is applied to Alice's result)

### Sample Client and Service Code for WebSocket

#### Client-Side Code (JavaScript)

```javascript
const socket = new WebSocket('

ws://yourserver.com/collaborate');

socket.onopen = () => {
  console.log('Connected to the server');
};

// Sending an operation
function sendOperation(operation) {
  socket.send(JSON.stringify(operation));
}

socket.onmessage = (event) => {
  const operation = JSON.parse(event.data);
  applyOperation(operation);
};

function applyOperation(operation) {
  // Apply the received operation to the document
}

function createOperation(type, content) {
  return {
    type: type,
    content: content,
    timestamp: Date.now(),
    userId: currentUserId,
  };
}
```

#### Server-Side Code (Java)

```java
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.util.HashSet;
import java.util.Set;

public class CollaborationHandler extends TextWebSocketHandler {

    private final Set<WebSocketSession> sessions = new HashSet<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        sessions.add(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        for (WebSocketSession s : sessions) {
            if (s.isOpen() && !s.getId().equals(session.getId())) {
                s.sendMessage(new TextMessage(payload));
            }
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        sessions.remove(session);
    }
}
```

### High Level Diagram

```
+-------------------+
|                   |
|   Client-Side     |
|                   |
| Web/Mobile Apps   |
|                   |
+---------+---------+
          |
          v
+---------+---------+       +---------------------+
|                   |       |                     |
|   API Gateway     | <---- |  Authentication &   |
|                   |       |  Authorization      |
+---------+---------+       +---------------------+
          |
          v
+---------+---------+       +---------------------+
|                   |       |                     |
| Document Service  | <---- |  User Service       |
|                   |       |                     |
+---------+---------+       +---------------------+
          |
          v
+---------+---------+
|                   |
| Real-Time Service |
|                   |
+---------+---------+
          |
          v
+---------+---------+
|                   |
|  Change Log       |
|  Service          |
|                   |
+---------+---------+
          |
          v
+---------+---------+
|                   |
|  Storage          |
|                   |
+-------------------+
```


## References
* https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/system-design-google-docs
* https://medium.com/@sureshpodeti/system-design-google-docs-93e12133a979
* https://www.youtube.com/watch?v=2auwirNBvGg&t=2115s
* https://medium.com/@sureshpodeti/system-design-google-docs-93e12133a979