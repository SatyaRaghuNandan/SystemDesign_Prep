###### https://www.youtube.com/watch?v=xfX1fMQaNk4&t=1985s&ab_channel=jackniu


This transcript outlines the design of a **news feed system** similar to Twitter, Instagram, or Facebook, focusing on personalized and recommended content with pagination, scalability, and low-latency requirements. The speaker follows a system design framework, covering functional and non-functional requirements, traffic and storage estimation, API design, data schema, high-level architecture, and feed generation with ranking considerations. Since the user previously requested an image for similar system design transcripts (e.g., Yelp, YouTube), I’ll assume a **system architecture diagram** is desired but will confirm details before generating, as per guidelines. Below, I’ll summarize the design, address requirements, and propose a diagram.

---

### System Design: News Feed System Summary

#### 1. Requirements

**Functional Requirements:**
- **Personalized Feed**: Display posts from friends/followers (10 posts per page, with pagination).
- **Upload Posts**: Users can upload text, images, or videos.
- **Configurable Feeds**: Support topic-based feeds (e.g., sports, popular) and user-configurable tags.
- **Low Latency (Not Real-Time)**: Feed updates can tolerate a 10-second delay for new posts.

**Non-Functional Requirements:**
- **Scalability**: Handle user and post growth (500M daily active users, DAU).
- **Availability**: >99.9% availability (three nines).
- **Low Latency**: P99 latency <1 second for feed retrieval and scrolling.
- **Fault Tolerance/Data Redundancy**: Use primary-secondary database models for data backup.
- **Ranking**: Incorporate ranking algorithms (possibly ML-based) for feed personalization.

**Traffic Estimation:**
- **DAU**: 500M users.
- **QPS**: ~50K queries per second (500M / 86,400 seconds ≈ 5,787 QPS, rounded up with burst factor).
- **Post Volume**: 500M users × 10 posts/day = 5B posts/day.
- **Storage**:
  - **User Data**: 500M users × 10KB/user = 5TB.
  - **Text Posts**: 5B posts/day × 10KB/post = 50TB/day.
  - **Images**: 500M users × 10 posts × 1% image × 10KB = 500GB/day.
  - **Videos**: 500M users × 10 posts × 1% video × 10MB = 500TB/day.
  - **Total Content**: ~1PB/day (500TB video + 500GB image + 50TB text).
  - **Yearly Storage**: ~365PB/year (assuming no deletion).

**Assumptions**:
- 1% of posts are images/videos (conservative estimate).
- Feed retrieval serves top 10–20 posts, cached for performance.
- Auto-scaling for app servers, databases, and caches.

#### 2. Data Schema

**Relational Database (User DB, Post DB)**:
- **User Table**:
  - `userId` (int, PK): Unique user identifier.
  - `email` (string): User email.
  - Other fields (e.g., name, settings): Out of scope.
- **Post Table**:
  - `postId` (int, PK): Unique post identifier.
  - `creatorId` (int, FK): User who created the post.
  - `content` (string/URL): Text or S3 URL for image/video.
  - `timestamp` (datetime): Creation time.
  - Other fields (e.g., tags): Optional.

**Graph Database (Relationships)**:
- **Edge Table**:
  - `userA` (int): Follower user ID.
  - `userB` (int): Followed user ID.
  - `relationship` (string): “following”.
  - `timestamp` (datetime): When followed.
- Used to query followers for feed generation.

**Feed Cache (Redis)**:
- **Key**: `userId:tagId` (e.g., `123:sports` for user 123’s sports feed).
- **Value**: List of `{ postId: int, creatorId: int }` (top 10–20 posts).
- Stores pre-generated, ranked feeds per user and tag.

**Object Storage (S3)**:
- Stores images/videos, referenced by URLs in `Post` table.

#### 3. API Design

**Client-Facing APIs**:
1. **Upload Post**:
   - `POST /posts`
   - Parameters: `{ userId: int, content: string | Blob, tags?: string[] }`.
   - Response: `{ success: boolean, postId?: int }`.
   - Handles text, image, or video uploads.
2. **Get Feed**:
   - `GET /feeds?userId={int}&tagId={string}&count={int}&offset={int}`
   - Response: `{ posts: Array<{ postId: int, creatorId: int, content: string | URL, timestamp: string }> }`.
   - Returns 10 posts (default) with pagination (`offset` for scrolling).

#### 4. High-Level Design

**Components**:
- **Client**: Mobile/web app issuing upload and feed requests.
- **Load Balancer**: Distributes traffic to app servers.
- **App Server**: Auto-scalable, handles API requests (`POST /posts`, `GET /feeds`).
  - Routes uploads to message queue and media service.
  - Queries feed cache (Redis) for feed retrieval.
- **Message Queue (Kafka)**:
  - Buffers post upload events for asynchronous processing.
  - Triggers feed generation for followers.
- **Consumer**: Processes Kafka events, saves posts to cache/database, and updates feeds.
- **Media Service**: Handles image/video uploads to S3, returns URLs.
- **Object Storage (S3)**: Stores images/videos.
- **Feed Cache (Redis Cluster)**: Stores top 10–20 ranked posts per user/tag.
  - Sharded for scalability.
- **Post Database (Relational, Sharded)**: Stores post data (e.g., MySQL, sharded by `postId`).
  - Primary-secondary model for redundancy.
- **User Database (Relational)**: Stores user data (e.g., MySQL).
- **Graph Database (e.g., Neo4j)**: Stores follower relationships.
  - Cached in Redis for performance.
- **Feed Service**: Retrieves posts from cache/database, handles pagination.
- **Discovery Service**: Maps `userId` to database shards using ZooKeeper.
- **Feed Generation Service**: Pre-generates feeds for users based on new posts.
  - Queries graph DB for followers.
  - Ranks posts (ML-based, details below).
  - Updates feed cache.
- **ZooKeeper**: Manages shard mappings for sharded databases.

**Upload Workflow**:
1. Client sends `POST /posts` to App Server with `userId`, `content`, `tags`.
2. For text posts:
   - App Server sends event to Kafka.
   - Consumer saves post to Post Cache (Redis) and Post DB (MySQL).
   - Consumer sends event to Feed Generation Service via Kafka.
3. For image/video posts:
   - App Server calls Media Service.
   - Media Service uploads content to S3, returns URL.
   - On success, App Server sends post event (with S3 URL) to Kafka.
   - Consumer processes as above.
4. Feed Generation Service:
   - Queries Graph DB for user’s followers.
   - Ranks post with existing posts (ML-based).
   - Updates Feed Cache for each follower (top 10–20 posts).

**Retrieval Workflow**:
1. Client sends `GET /feeds?userId={int}&tagId={string}&count=10&offset={int}`.
2. App Server queries Feed Service.
3. Feed Service:
   - Checks Feed Cache (Redis) for top 10 posts (`userId:tagId`).
   - If cache miss or pagination (`offset > 10`), queries Discovery Service.
4. Discovery Service uses ZooKeeper to map `userId` to Post DB shard.
5. Feed Service queries Post DB for additional posts, caches results in Redis (for active users).
6. Returns ranked posts to client.

**Feed Generation**:
- Triggered by new post via Kafka.
- Queries Graph DB for followers.
- Ranks post with existing posts using ML algorithm (e.g., engagement score).
- Updates Feed Cache for each follower, maintaining top 10–20 posts.
- Offline process, ensuring eventual consistency (10s delay).

#### 5. Ranking Algorithm

- **Location**: Feed Generation Service, updating Feed Cache.
- **Approach**: ML-based ranking (simplified for design):
  - **Features**: Post age, creator’s follower count, user engagement (likes, comments), tag relevance.
  - **Model**: Logistic regression or neural network predicting engagement probability.
  - **Output**: Score per post, sorted to select top 10–20.
- **Implementation**:
  - Fetch follower posts from Post DB.
  - Score posts using ML model.
  - Store ranked `postId`s in Feed Cache.
- **Scalability**: Batch processing in Feed Generation Service, parallelized across followers.

#### 6. Non-Functional Requirements Check

- **Scalability**:
  - App Server, Media Service, Feed Service: Auto-scalable.
  - Kafka: Handles 50K QPS with partitioning.
  - Redis: Sharded cluster for feed cache.
  - MySQL: Sharded by `postId`, scaled with replicas.
  - S3: “Infinitely scalable” for media.
- **Availability (>99.9%)**:
  - Load Balancer and auto-scaling ensure no single point of failure.
  - Kafka’s replication and Redis clustering provide fault tolerance.
  - MySQL primary-secondary model ensures failover.
- **Low Latency (P99 < 1s)**:
  - Feed Cache (Redis) serves top 10 posts in <100ms.
  - Pagination queries hit sharded Post DB, cached for active users.
- **Data Redundancy**:
  - MySQL primary-secondary model for User/Post DB.
  - S3 replication for media.
  - Kafka’s persistent logs for events.
- **Ranking**: ML-based ranking in Feed Generation Service meets personalization needs.

#### 7. Edge Cases and Considerations

- **Cold Start**: New users with no followers get recommended posts (e.g., popular or tag-based feeds).
- **Active Users**: Cache more posts for active users to reduce DB queries.
- **Media Upload Failures**: Retry uploads to S3; only save post if successful.
- **Feed Staleness**: Eventual consistency (10s delay) acceptable per requirements.
- **Sharding**: Post DB sharded by `postId`; user-to-shard mapping in ZooKeeper.

---

### Image Generation Request

Given the user’s prior requests for images with system design transcripts, I’ll propose a **system architecture diagram** visualizing the components (Client, Load Balancer, App Server, Kafka, Feed Service, Redis, MySQL, S3, etc.) and their interactions. Since image generation requires confirmation, I’ll provide a Chart.js-based placeholder (per guidelines) and describe the intended diagram.

#### Proposed Image: System Architecture Diagram

**Description**:
- A diagram showing:
  - **Client**: Issues `POST /posts` and `GET /feeds`.
  - **Load Balancer**: Distributes traffic.
  - **App Server**: Handles API requests, routes to Kafka/Media Service.
  - **Kafka**: Two queues (post uploads, feed generation).
  - **Consumer**: Saves posts to Redis/MySQL, triggers feed generation.
  - **Media Service**: Uploads images/videos to S3.
  - **S3**: Stores media content.
  - **Feed Service**: Retrieves feeds from Redis/MySQL.
  - **Redis Cluster**: Caches top 10–20 posts per user/tag.
  - **MySQL (Sharded)**: Stores user/post data with primary-secondary.
  - **Graph DB (Neo4j)**: Stores follower relationships.
  - **Feed Generation Service**: Ranks posts, updates Redis.
  - **Discovery Service/ZooKeeper**: Maps `userId` to shards.
  - Arrows showing flows:
    - Upload: Client → App Server → Kafka → S3/MySQL/Redis → Feed Generation.
    - Retrieval: Client → App Server → Feed Service → Redis/MySQL.
  - Labels for key processes (e.g., “ML Ranking,” “Pagination”).

**Chart.js Placeholder**:
A `scatter` chart with nodes for components and lines for connections, simulating the architecture.

```chartjs
{
  "type": "scatter",
  "data": {
    "datasets": [
      {
        "label": "Components",
        "data": [
          { "x": 0, "y": 6, "label": "Client" },
          { "x": 2, "y": 6, "label": "Load Balancer" },
          { "x": 4, "y": 6, "label": "App Server" },
          { "x": 6, "y": 7, "label": "Kafka" },
          { "x": 6, "y": 5, "label": "Consumer" },
          { "x": 4, "y": 4, "label": "Media Service" },
          { "x": 6, "y": 3, "label": "S3" },
          { "x": 8, "y": 7, "label": "Feed Service" },
          { "x": 8, "y": 5, "label": "Redis" },
          { "x": 8, "y": 3, "label": "MySQL" },
          { "x": 10, "y": 7, "label": "Graph DB" },
          { "x": 10, "y": 5, "label": "Feed Generation" },
          { "x": 10, "y": 3, "label": "Discovery/ZooKeeper" }
        ],
        "backgroundColor": ["#4CAF50", "#2196F3", "#FF9800", "#F44336", "#9C27B0", "#673AB7", "#FF5722", "#009688", "#3F51B5", "#E91E63", "#795548", "#607D8B", "#0288D1"],
        "borderColor": ["#388E3C", "#1976D2", "#F57C00", "#D32F2F", "#7B1FA2", "#512DA8", "#E64A19", "#00796B", "#303F9F", "#C2185B", "#5D4037", "#455A64", "#0277BD"],
        "pointRadius": 10,
        "pointHoverRadius": 12
      },
      {
        "label": "Connections",
        "data": [
          { "x": 0, "y": 6 }, { "x": 2, "y": 6 },
          { "x": 2, "y": 6 }, { "x": 4, "y": 6 },
          { "x": 4, "y": 6 }, { "x": 6, "y": 7 },
          { "x": 4, "y": 6 }, { "x": 4, "y": 4 },
          { "x": 4, "y": 4 }, { "x": 6, "y": 3 },
          { "x": 6, "y": 7 }, { "x": 6, "y": 5 },
          { "x": 6, "y": 5 }, { "x": 8, "y": 5 },
          { "x": 6, "y": 5 }, { "x": 8, "y": 3 },
          { "x": 6, "y": 5 }, { "x": 6, "y": 7 },
          { "x": 4, "y": 6 }, { "x": 8, "y": 7 },
          { "x": 8, "y": 7 }, { "x": 8, "y": 5 },
          { "x": 8, "y": 7 }, { "x": 8, "y": 3 },
          { "x": 8, "y": 7 }, { "x": 10, "y": 3 },
          { "x": 6, "y": 7 }, { "x": 10, "y": 5 },
          { "x": 10, "y": 5 }, { "x": 10, "y": 7 },
          { "x": 10, "y": 5 }, { "x": 8, "y": 5 }
        ],
        "showLine": true,
        "borderColor": "#757575",
        "borderWidth": 2,
        "pointRadius": 0
      }
    ]
  },
  "options": {
    "plugins": {
      "legend": { "display": false },
      "annotation": {
        "annotations": [
          { "type": "label", "xValue": 0, "yValue": 6, "content": ["Client"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#4CAF50" },
          { "type": "label", "xValue": 2, "yValue": 6, "content": ["Load Balancer"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#2196F3" },
          { "type": "label", "xValue": 4, "yValue": 6, "content": ["App Server"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF9800" },
          { "type": "label", "xValue": 6, "yValue": 7, "content": ["Kafka"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#F44336" },
          { "type": "label", "xValue": 6, "yValue": 5, "content": ["Consumer"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#9C27B0" },
          { "type": "label", "xValue": 4, "yValue": 4, "content": ["Media Service"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#673AB7" },
          { "type": "label", "xValue": 6, "yValue": 3, "content": ["S3"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF5722" },
          { "type": "label", "xValue": 8, "yValue": 7, "content": ["Feed Service"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#009688" },
          { "type": "label", "xValue": 8, "yValue": 5, "content": ["Redis"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#3F51B5" },
          { "type": "label", "xValue": 8, "yValue": 3, "content": ["MySQL"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#E91E63" },
          { "type": "label", "xValue": 10, "yValue": 7, "content": ["Graph DB"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#795548" },
          { "type": "label", "xValue": 10, "yValue": 5, "content": ["Feed Generation"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#607D8B" },
          { "type": "label", "xValue": 10, "yValue": 3, "content": ["Discovery/ZooKeeper"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#0288D1" }
        ]
      }
    },
    "scales": {
      "x": { "display": false, "min": -1, "max": 11 },
      "y": { "display": false, "min": 2, "max": 8 }
    }
  }
}
```

**Explanation**:
- **Nodes**: Represent Client, Load Balancer, App Server, Kafka, Consumer, Media Service, S3, Feed Service, Redis, MySQL, Graph DB, Feed Generation, Discovery/ZooKeeper.
- **Lines**: Show key flows (e.g., Client to App Server, App Server to Kafka/S3, Feed Service to Redis/MySQL).
- **Colors**: Distinctive for dark/light themes.
- **Labels**: Annotate components for clarity.
- **Limitations**: Chart.js is suboptimal for architecture diagrams; a UML-like image would be more detailed.

---

### Confirmation Needed

Please confirm or clarify:
1. Is a **system architecture diagram** the desired image, or do you prefer another type (e.g., flowchart of upload/retrieval, UI mockup of news feed)?
2. If a diagram, should it highlight specific elements (e.g., ML ranking, Kafka queues, sharding)?
3. If a chart, what data should it visualize (e.g., QPS distribution, storage growth)?
4. Any style preferences (e.g., minimalist, detailed)?

If you confirm the architecture diagram, I can describe a more detailed UML-style image or refine the Chart.js output. If no response, I’ll assume the above chart suffices. Let me know how to proceed!
