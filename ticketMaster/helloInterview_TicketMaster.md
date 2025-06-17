###### https://www.youtube.com/watch?v=fhdPyoO6aXI&t=1962s&ab_channel=HelloInterview-SWEInterviewPreparation

This transcript provides a detailed walkthrough of designing a **ticket booking service** like Ticketmaster, as presented by Evan, a former Meta staff engineer and co-founder of Hello Interview. The design follows a structured system design interview framework, covering functional and non-functional requirements, core entities, APIs, high-level architecture, and deep dives into scalability and low-latency challenges. Given the user’s prior request for a system architecture diagram for similar system design transcripts (e.g., news feed, YouTube), I’ll assume a **system architecture diagram** is desired but will confirm details before generating, as per guidelines. Below, I’ll summarize the design, address requirements, and propose a diagram.

---

### System Design: Ticket Booking Service Summary

#### 1. Requirements

**Functional Requirements:**
- **Search for Events**: Users can search for events by term, location, type, or date.
- **View Event Details**: Users can view event details, including venue, performer, and seat map (available tickets).
- **Book Tickets**: Users can reserve and confirm a ticket purchase (two-phase process).

**Non-Functional Requirements:**
- **Strong Consistency for Booking**: Prevent double-booking (no ticket assigned to multiple users).
- **High Availability for Search/View**: Event search and viewing should be highly available, tolerating eventual consistency for new events.
- **Scalability for Surges**: Handle traffic spikes for popular events (e.g., Taylor Swift concerts, Super Bowl).
- **Low Latency Search**: Fast search results, even for popular queries.
- **Read-Heavy Workload**: Reads (search/view) outnumber writes (booking) by ~100:1.
- **Out of Scope**: GDPR compliance, fault tolerance (noted but deprioritized).

**Assumptions**:
- Conversion rate: ~1% of searches/views lead to bookings.
- Event/venue/performer data changes infrequently; ticket status is dynamic.
- No explicit QPS/storage estimates (per Evan’s advice: math only when it informs design).

#### 2. Core Entities

- **Event**:
  - `id` (PK), `venueId` (FK), `performerId` (FK), `name`, `description`, ticket IDs (1-to-many).
- **Venue**:
  - `id` (PK), `location`, `seatMap`.
- **Performer**:
  - `id` (PK), other metadata (out of scope).
- **Ticket**:
  - `id` (PK), `eventId` (FK), `seat`, `price`, `status` (available/reserved/booked), `userId` (if booked), `reservedTimestamp` (optional).

#### 3. API Design

**RESTful APIs**:
1. **Search Events**:
   - `GET /search?term={string}&location={string}&type={string}&date={string}`
   - Response: `{ events: Array<PartialEvent> }` (e.g., `id`, `name`, `description`).
2. **View Event**:
   - `GET /events/:eventId`
   - Response: `{ event: { id, name, description, venue: {...}, performer: {...}, tickets: Array<{ id, seat, price, status }> } }`.
3. **Reserve Ticket**:
   - `POST /bookings/reserve`
   - Body: `{ ticketId: int }`, Header: JWT/session token (user authentication).
   - Response: `{ success: boolean }`.
   - Reserves ticket for 10 minutes.
4. **Confirm Booking**:
   - `PUT /bookings/confirm`
   - Body: `{ ticketId: int, paymentDetails: {...} }`, Header: JWT.
   - Response: `{ success: boolean }`.
   - Processes payment via Stripe, marks ticket as booked.

#### 4. High-Level Design

**Components**:
- **Client**: Browser/mobile app issuing API requests.
- **API Gateway**: Routes requests, handles authentication, rate limiting (e.g., AWS API Gateway).
- **Event CRUD Service**: Handles `GET /events/:eventId` and `GET /search`, reads event/venue/performer/ticket data.
- **Search Service**: Processes search queries, initially via SQL, optimized later.
- **Booking Service**: Manages `POST /bookings/reserve` and `PUT /bookings/confirm`, updates ticket status.
- **PostgreSQL Database**: Stores event, venue, performer, and ticket tables.
  - ACID transactions ensure consistency for bookings.
  - Sharded by `eventId` or `venueId` (if geographically distributed).
- **Redis (Ticket Lock)**: Distributed lock for ticket reservations with 10-minute TTL.
- **Stripe**: Third-party payment processor, handles payment via webhooks.
- **Cron Job (Optional)**: Cleans up expired reservations (suboptimal, replaced by Redis lock).

**Workflows**:
- **Search**:
  1. Client sends `GET /search` to API Gateway.
  2. Routes to Search Service, queries PostgreSQL (initially slow).
  3. Returns partial event list.
- **View Event**:
  1. Client sends `GET /events/:eventId` to API Gateway.
  2. Routes to Event CRUD Service, queries PostgreSQL for event, venue, performer, and tickets.
  3. Checks Redis for reserved tickets, filters out from available list.
  4. Returns event details and seat map.
- **Reserve Ticket**:
  1. Client sends `POST /bookings/reserve` with `ticketId` and JWT.
  2. API Gateway routes to Booking Service, authenticates user.
  3. Sets `ticketId` in Redis with 10-minute TTL (no DB update).
  4. Returns success.
- **Confirm Booking**:
  1. Client sends `PUT /bookings/confirm` with `ticketId`, payment details, JWT.
  2. Booking Service calls Stripe, awaits webhook confirmation.
  3. On success, updates PostgreSQL ticket status to `booked`, sets `userId`, removes Redis lock.
  4. Returns success.

**Reservation Expiry**:
- **Initial Approach (Suboptimal)**: Store `reservedTimestamp` in ticket table, use Cron job to reset `status` to `available` after 10 minutes. Introduces ~9-minute delay (job runs every 10 minutes).
- **Optimized Approach**: Use Redis distributed lock with 10-minute TTL. Tickets not in Redis are available, ensuring real-time expiry.

#### 5. Deep Dives

**Deep Dive 1: Low Latency Search**
- **Problem**: SQL query (`SELECT * FROM events WHERE name LIKE '%term%'`) requires full table scan, slow for large datasets.
- **Solution**: Use **Elasticsearch** for inverted index and geospatial queries.
  - Tokenizes event `name`, `description` into terms (e.g., “Philadelphia Eagles” → “Philadelphia”, “Eagles”).
  - Maps terms to event IDs for fast lookup.
  - Supports location (`lat/lng`) and date range queries.
- **Data Sync**:
  - **Option 1**: Application code writes to PostgreSQL and Elasticsearch on event creation/update (risk of inconsistency if one fails).
  - **Option 2 (Preferred)**: Change Data Capture (CDC) streams PostgreSQL changes to Elasticsearch.
    - Low update frequency (events rarely change) avoids write bottlenecks.
- **Caching**:
  - **Node Query Caching**: OpenSearch caches top 10K queries per shard (LRU).
  - **Redis/Memcached**: Cache normalized search terms and results.
  - **CDN**: Cache `GET /search` responses for 30–60 seconds, effective for popular queries (e.g., “Taylor Swift”).
    - Less effective with high query permutations (e.g., precise `lat/lng`).
    - Invalid for personalized results (not required here).

**Deep Dive 2: Scalability for Popular Event Surges**
- **Problem**: Popular events (e.g., Super Bowl) cause traffic spikes, overwhelming backend and creating stale seat maps.
- **Solution 1: Real-Time Seat Updates**:
  - **Long Polling**: Client sends HTTP request, server holds for 30–60 seconds, responds with seat updates. Cheap, suitable for short sessions (1–5 minutes).
  - **Server-Sent Events (SSE)**: Unidirectional persistent connection from server to client, pushes seat status changes (reserved/booked). Preferred for longer sessions.
    - Avoids WebSockets (bidirectional, overkill).
  - **Implementation**: Event CRUD Service pushes updates via SSE when Redis lock or PostgreSQL ticket status changes.
  - **Drawback**: For ultra-popular events, seat map goes “black” as tickets sell out instantly, poor UX.
- **Solution 2: Virtual Waiting Queue**:
  - **Mechanism**: For popular events, users enter a Redis-based priority queue (sorted set by arrival time or randomized for fairness).
    - Admin-enabled via config.
  - **Workflow**:
    1. Users join queue instead of accessing event page.
    2. Queue releases users in batches (e.g., 100 users per 100 booked seats).
    3. Notified via SSE when allowed to book.
  - **Benefits**: Protects backend, improves UX by managing expectations.
  - **Implementation**: Redis sorted set for queue, event-driven logic for releases.

**Deep Dive 3: Strong Consistency for Booking**
- **Covered in High-Level Design**: Redis lock ensures no double-booking during reservation; PostgreSQL ACID transactions ensure single write for `booked` status.

**Deep Dive 4: Read-Heavy Workload**
- **Solution**: Cache static data (event, venue, performer) in Redis.
  - Key: `eventId`, Value: `{ event, venue, performer }`.
  - Infrequent updates (events rarely change) simplify cache invalidation.
  - Tickets remain uncached (dynamic status).
- **Impact**: Reduces PostgreSQL read load, speeds up `GET /events/:eventId`.

**Deep Dive 5: Sharding (Optional)**
- **When Needed**: If PostgreSQL storage exceeds single instance capacity.
- **Strategy**:
  - Shard by `eventId`: Most queries filter by event.
  - Shard by `venueId`: If users search events near their location (geographic correlation).
- **Trade-Offs**: Discussed with interviewer (e.g., query patterns, data locality).

#### 6. Non-Functional Requirements Check

- **Strong Consistency for Booking**: Redis lock + PostgreSQL ACID transactions prevent double-booking.
- **High Availability for Search/View**: Elasticsearch and Redis caching ensure fast, available reads; eventual consistency for new events acceptable.
- **Scalability for Surges**: Virtual waiting queue, auto-scaling services, and load balancers handle spikes.
- **Low Latency Search**: Elasticsearch, CDN, and Redis caching achieve fast search.
- **Read-Heavy Workload**: Redis caching for static data reduces DB load.
- **Out of Scope**: GDPR, fault tolerance addressed implicitly via redundancy.

---

### Image Generation Request

Given the user’s consistent requests for system architecture diagrams, I’ll propose a **system architecture diagram** visualizing components (Client, API Gateway, Event CRUD Service, Search Service, Booking Service, PostgreSQL, Redis, Elasticsearch, Stripe, Virtual Waiting Queue) and their interactions. I’ll provide a Chart.js placeholder and describe the intended diagram, awaiting confirmation.

#### Proposed Image: System Architecture Diagram

**Description**:
- A diagram showing:
  - **Client**: Issues `GET /search`, `GET /events/:eventId`, `POST /bookings/reserve`, `PUT /bookings/confirm`.
  - **API Gateway**: Routes requests, authenticates via JWT.
  - **Event CRUD Service**: Queries PostgreSQL/Redis for event details, checks Redis lock for ticket status.
  - **Search Service**: Queries Elasticsearch for search results, cached in CDN/Redis.
  - **Booking Service**: Manages reservations (Redis lock), confirms bookings (Stripe, PostgreSQL).
  - **PostgreSQL**: Stores event, venue, performer, ticket tables (sharded if needed).
  - **Redis**: Distributed lock for reservations, cache for static data and search results.
  - **Elasticsearch**: Inverted index for search, synced via CDC.
  - **Stripe**: Processes payments via webhooks.
  - **Virtual Waiting Queue (Redis)**: Manages surges for popular events.
  - Arrows showing flows:
    - Search: Client → API Gateway → Search Service → Elasticsearch/CDN.
    - View: Client → API Gateway → Event CRUD Service → PostgreSQL/Redis.
    - Booking: Client → API Gateway → Booking Service → Redis/Stripe/PostgreSQL.
    - Queue: Client → API Gateway → Virtual Waiting Queue → Redis.
  - Labels for key processes (e.g., “SSE Updates,” “CDC Sync,” “TTL Lock”).

**Chart.js Placeholder**:
A `scatter` chart with nodes for components and lines for connections.

```chartjs
{
  "type": "scatter",
  "data": {
    "datasets": [
      {
        "label": "Components",
        "data": [
          { "x": 0, "y": 6, "label": "Client" },
          { "x": 2, "y": 6, "label": "API Gateway" },
          { "x": 4, "y": 7, "label": "Event CRUD Service" },
          { "x": 4, "y": 5, "label": "Search Service" },
          { "x": 4, "y": 3, "label": "Booking Service" },
          { "x": 6, "y": 7, "label": "PostgreSQL" },
          { "x": 6, "y": 5, "label": "Redis" },
          { "x": 6, "y": 3, "label": "Elasticsearch" },
          { "x": 6, "y": 1, "label": "Stripe" },
          { "x": 8, "y": 5, "label": "Virtual Waiting Queue" }
        ],
        "backgroundColor": ["#4CAF50", "#2196F3", "#FF9800", "#F44336", "#9C27B0", "#673AB7", "#FF5722", "#009688", "#3F51B5", "#E91E63"],
        "borderColor": ["#388E3C", "#1976D2", "#F57C00", "#D32F2F", "#7B1FA2", "#512DA8", "#E64A19", "#00796B", "#303F9F", "#C2185B"],
        "pointRadius": 10,
        "pointHoverRadius": 12
      },
      {
        "label": "Connections",
        "data": [
          { "x": 0, "y": 6 }, { "x": 2, "y": 6 },
          { "x": 2, "y": 6 }, { "x": 4, "y": 7 },
          { "x": 2, "y": 6 }, { "x": 4, "y": 5 },
          { "x": 2, "y": 6 }, { "x": 4, "y": 3 },
          { "x": 2, "y": 6 }, { "x": 8, "y": 5 },
          { "x": 4, "y": 7 }, { "x": 6, "y": 7 },
          { "x": 4, "y": 7 }, { "x": 6, "y": 5 },
          { "x": 4, "y": 5 }, { "x": 6, "y": 3 },
          { "x": 4, "y": 3 }, { "x": 6, "y": 5 },
          { "x": 4, "y": 3 }, { "x": 6, "y": 7 },
          { "x": 4, "y": 3 }, { "x": 6, "y": 1 },
          { "x": 6, "y": 7 }, { "x": 6, "y": 3 },
          { "x": 8, "y": 5 }, { "x": 6, "y": 5 }
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
          { "type": "label", "xValue": 2, "yValue": 6, "content": ["API Gateway"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#2196F3" },
          { "type": "label", "xValue": 4, "yValue": 7, "content": ["Event CRUD"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF9800" },
          { "type": "label", "xValue": 4, "yValue": 5, "content": ["Search Service"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#F44336" },
          { "type": "label", "xValue": 4, "yValue": 3, "content": ["Booking Service"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#9C27B0" },
          { "type": "label", "xValue": 6, "yValue": 7, "content": ["PostgreSQL"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#673AB7" },
          { "type": "label", "xValue": 6, "yValue": 5, "content": ["Redis"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF5722" },
          { "type": "label", "xValue": 6, "yValue": 3, "content": ["Elasticsearch"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#009688" },
          { "type": "label", "xValue": 6, "yValue": 1, "content": ["Stripe"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#3F51B5" },
          { "type": "label", "xValue": 8, "yValue": 5, "content": ["Waiting Queue"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#E91E63" }
        ]
      }
    },
    "scales": {
      "x": { "display": false, "min": -1, "max": 9 },
      "y": { "display": false, "min": 0, "max": 8 }
    }
  }
}
```

**Explanation**:
- **Nodes**: Represent Client, API Gateway, Event CRUD Service, Search Service, Booking Service, PostgreSQL, Redis, Elasticsearch, Stripe, Virtual Waiting Queue.
- **Lines**: Show flows (e.g., Client to services, services to DBs, Booking Service to Stripe).
- **Colors**: Distinctive for dark/light themes.
- **Labels**: Annotate components.
- **Limitations**: Chart.js is suboptimal; a UML diagram would be more precise.

---

### Confirmation Needed

Please confirm or clarify:
1. Is a **system architecture diagram** the desired image, or do you prefer another type (e.g., flowchart of booking process, seat map UI mockup)?
2. If a diagram, should it emphasize specific elements (e.g., Redis lock, virtual waiting queue, SSE updates)?
3. If a chart, what data should it visualize (e.g., QPS surges, ticket reservation times)?
4. Any style preferences (e.g., minimalist, detailed)?

If you confirm the architecture diagram, I can describe a detailed UML-style image or refine the Chart.js output. If no response, I’ll assume the above chart suffices. Let me know how to proceed!
