WhatsApp System Design Interview Script: Handling Multiple Clients for a Single User
This script provides a conversational, step-by-step response for a system design interview question about handling multiple clients for a single user in WhatsApp. It’s designed to be delivered naturally, covering requirements, calculations, solutions, challenges, trade-offs, and responses to interviewer probes, with a focus on depth and clarity.

Interviewer: In WhatsApp, a user might be logged in on multiple devices—like their phone, tablet, and web browser. How would you design the system to handle multiple clients for a single user, ensuring messages are delivered consistently and in real-time?
You: Great question! Let’s design a system to handle multiple clients for a single user in WhatsApp, ensuring all devices receive messages in real-time with consistent state. I’ll start by clarifying requirements, do some quick calculations to understand the scale, propose a few solutions, dive deep into trade-offs, and address any follow-ups. Feel free to jump in with clarifications or questions!
Step 1: Clarify Requirements
When a user is logged in on multiple devices—say, their phone, tablet, and web browser—we need to ensure all clients receive messages simultaneously, maintain consistent chat history, and handle state like read receipts across devices. For example, if User A sends a message to User B, who’s on three devices, all three should get the message instantly, and marking it as read on one device should sync across all.
Key requirements:

Real-Time Delivery: Messages must reach all active clients in near real-time (<100ms latency).
Consistency: Chat history, read receipts, and message states (e.g., sent, delivered) must be identical across devices.
Scalability: Support ~1B users, with maybe 10% (100M) having multiple clients (e.g., 2-3 devices each).
Reliability: No message loss, even if a device is temporarily offline.
User Experience: Seamless sync (e.g., typing indicators, message deletions) across devices.
Features: Support 1:1 and group chats, with durable storage for offline devices.

I’ll assume we’re using WebSocket connections for real-time messaging, with a database (e.g., Inbox table) for persistent storage. Does this scope work, or should I include specifics like media sync or end-to-end encryption?
Interviewer: That’s fine, focus on messaging and state sync. How do you handle the scale of multiple clients?
Step 2: Back-of-the-Envelope Calculations
Let’s estimate the scale. Assume 1B total users, with 10% (100M) using multiple devices. If each has 2-3 clients on average (e.g., phone + tablet or phone + web), that’s ~200-300M active client connections. WhatsApp historically handled 1-2M connections per chat server, so let’s use 2M:

300M connections ÷ 2M/server ≈ 150 chat servers.

For message volume, assume 10B messages/day (1B users × 10 messages/day). That’s:

10B ÷ 86,400 seconds ≈ 115,000 messages/second (average).
With multiple clients, each message might need 2-3 deliveries, so peak throughput could hit ~300-350K deliveries/second.

Each message is ~1KB, so bandwidth is manageable, but syncing state (e.g., read receipts) across devices adds overhead. We’ll use these numbers to identify bottlenecks.
Step 3: High-Level Architecture
Here’s the setup: Users connect via WebSocket to chat servers, which route messages and sync state. Messages are stored in an Inbox table (e.g., sharded MySQL or Cassandra) for durability. The challenge is delivering messages to all of User B’s clients (e.g., phone, tablet, web) and keeping their state (chat history, read receipts) consistent, even if they’re on different servers.
A naive approach might broadcast messages to all servers, but that’s inefficient. We need a way to track a user’s active clients, route messages to them, and sync state efficiently. Let’s explore solutions.
Step 4: Exploring Solutions
Approach 1: Broadcast to All Chat Servers
Approach: When User A sends a message to User B, the sending server broadcasts it to all chat servers. Each server checks if it has any of User B’s clients connected and delivers the message via WebSocket. State updates (e.g., read receipts) are written to the Inbox table and broadcast similarly.
Flow:

User A (Server 1) sends a message to User B.
Server 1 broadcasts to all 150 servers.
Servers with User B’s clients (e.g., Server 2: phone, Server 3: tablet) deliver the message.
Read receipts are written to the database and broadcast to update all clients.

Challenges:

Inefficiency: Broadcasting to 150 servers for every message is wasteful, especially if User B has only 2-3 clients.
Scalability: At 350K deliveries/second, broadcasting multiplies network traffic (150 servers × 350K = 52.5M messages/second).
Consistency: Concurrent state updates (e.g., read receipts from multiple devices) could cause race conditions without careful locking.

Why It’s Weak: The broadcast overhead kills scalability, and consistency is hard to maintain without complex coordination.
Interviewer: Broadcasting sounds inefficient. What’s another approach?
Approach 2: Client Registry with Database
Approach: Maintain a central database (e.g., Redis or DynamoDB) to track which chat servers host each user’s clients. When User B connects on their phone to Server 2, we store {userB: [Server2]}; adding a tablet on Server 3 updates it to {userB: [Server2, Server3]}. Messages are sent only to the listed servers.
Flow:

User B’s phone connects to Server 2; Server 2 registers in Redis: {userB: [Server2]}.
User B’s tablet connects to Server 3; Redis updates to {userB: [Server2, Server3]}.
User A (Server 1) sends a message to User B; Server 1 queries Redis, gets [Server2, Server3], and sends the message to both.
State updates (e.g., read receipt from phone) are written to the Inbox table and pushed to Server 3 for the tablet.

Challenges:

Database Latency: Querying Redis for every message adds 1-2ms latency, critical at 350K messages/second.
Consistency: Rapid client connections/disconnections (e.g., flaky networks) could lead to stale registry entries.
Scalability: The registry database becomes a hotspot, handling millions of updates/second for connections and messages.

Pros: Targeted delivery reduces network overhead compared to broadcasting.
Interviewer: The database could be a bottleneck. Any way to avoid it?
Approach 3: Consistent Hashing with Client Tracking
Approach: Use consistent hashing to assign each user to a “home” chat server based on their user ID. Each client (phone, tablet, web) connects to this home server. The home server tracks all of User B’s clients, even if they’re on other servers due to load balancing, and routes messages internally.
Flow:

User B’s ID hashes to Server 2 (home server). Phone connects to Server 2, tablet to Server 3 (due to load).
Server 3 notifies Server 2: “I have User B’s tablet.”
User A (Server 1) sends a message to User B; Server 1 hashes User B’s ID to Server 2 and sends the message.
Server 2 forwards the message to User B’s phone (local) and Server 3 (tablet).
State updates are coordinated by Server 2 and written to the Inbox table.

Challenges:

Home Server Bottleneck: The home server must handle all of User B’s clients, which could overload if a user has many devices.
Scaling Complexity: Adding servers requires rebalancing, and client tracking adds overhead for cross-server coordination.
Failure Handling: If the home server fails, we need a failover mechanism (e.g., ZooKeeper to reassign home servers).

Pros: Predictable routing with no central database query per message.
Interviewer: That’s interesting, but what about failures? Any other ideas?
Approach 4: Redis Pub/Sub with Client Subscriptions
Approach: Use Redis Pub/Sub, where each user ID is a topic. When User B’s phone connects to Server 2, it subscribes to pubsub:userB; the tablet on Server 3 does the same. Messages are published to pubsub:userB, and all subscribed servers deliver to their clients. State is synced via the Inbox table.
Flow:

User B’s phone (Server 2) and tablet (Server 3) subscribe to pubsub:userB.
User A (Server 1) sends a message to User B, publishing to pubsub:userB.
Server 2 and Server 3 receive the message and push it to User B’s phone and tablet via WebSocket.
Read receipt from phone is written to the Inbox table and published to pubsub:userB to sync the tablet.
Offline clients (e.g., web) retrieve messages from the Inbox table on reconnect.

Challenges:

Latency: Pub/Sub adds 5-10ms latency due to Redis routing.
Connection Overhead: Each server connects to all Redis nodes, creating an all-to-all relationship (e.g., 150 servers × 10 Redis nodes).
At-Most-Once Delivery: Pub/Sub may drop messages if no subscribers are active, but the Inbox table ensures durability.

Pros: Scalable, lightweight, and aligns with WhatsApp’s Erlang-based pub/sub architecture. Handles multiple clients elegantly.
Step 5: Deep Dive into Trade-Offs
Let’s compare these approaches to pick the best one, focusing on scalability, latency, consistency, and complexity.
Broadcast to All Chat Servers:

Pros: Simple to implement—no need to track client locations. Works for small-scale systems.
Cons: Scales poorly; broadcasting 350K messages/second to 150 servers creates ~52.5M messages/second, overwhelming the network. Race conditions for state updates (e.g., read receipts) require complex locking.
Trade-Offs:
Simplicity vs. Scalability: Easy setup but unscalable due to broadcast overhead.
Performance vs. Efficiency: Fast for small clusters but wasteful for WhatsApp’s scale.
Consistency vs. Complexity: Hard to ensure consistent state without heavy database coordination.


When to Use: Suitable for tiny systems (e.g., 10 servers), but not for 100M multi-client users.

Client Registry with Database:

Pros: Targeted delivery reduces network traffic compared to broadcasting. Scales better by querying only relevant servers.
Cons: Redis/DynamoDB becomes a bottleneck at 350K queries/second. Stale entries from flaky connections cause delivery failures.
Trade-Offs:
Scalability vs. Latency: Scales to millions of users but adds 1-2ms per message, impacting real-time delivery.
Reliability vs. Complexity: Reliable if registry is accurate, but maintaining consistency is complex with frequent updates.
*When to Use: Viable for systems with fewer concurrent clients or slower message rates, but risky for WhatsApp’s scale.



Consistent Hashing with Client Tracking:

Pros: No per-message database queries, as the home server routes messages. Predictable routing via user ID hashing.
Cons: Home server becomes a hotspot if a user has many clients. Scaling requires rebalancing, and failures need failover (e.g., via ZooKeeper). Cross-server client tracking adds complexity.
Trade-Offs:
Performance vs. Scalability: Low latency for routing but limited by home server capacity.
Reliability vs. Complexity: Reliable delivery if home server is up, but failover and rebalancing are complex.
Scalability vs. Load Balancing: Scales with more servers, but load imbalances (e.g., popular users) cause bottlenecks.


When to Use: Good for systems with predictable client counts per user, but less ideal for WhatsApp’s variable client load.

Redis Pub/Sub with Client Subscriptions:

Pros: Highly scalable—Redis handles millions of subscriptions. Lightweight and aligns with WhatsApp’s Erlang pub/sub model. Inbox table ensures durability for offline clients.
Cons: 5-10ms latency from Pub/Sub routing. All-to-all connections (servers to Redis nodes) increase network complexity. At-most-once delivery risks transient drops, mitigated by the Inbox table.
Trade-Offs:
Latency vs. Scalability: Small latency hit for massive scalability with Redis clusters.
Simplicity vs. Reliability: Simple pub/sub setup, but needs Inbox table for guaranteed delivery.
Cost vs. Performance: Low memory footprint vs. database registry, but network overhead from connections.


When to Use: Ideal for WhatsApp’s real-time, multi-client needs, balancing scalability and performance.

Interviewer: Why Redis Pub/Sub over consistent hashing?
Step 6: Recommendation and Mitigations
I recommend Redis Pub/Sub for handling multiple clients. It’s scalable, handling 300M client connections across 150 servers, and aligns with WhatsApp’s real-world architecture, which uses similar pub/sub patterns. The Inbox table ensures consistency and durability, and the 5-10ms latency is acceptable for real-time messaging. Consistent hashing is efficient but risks home server bottlenecks and complex failover, making Pub/Sub a better fit.
Mitigations:

Latency: Co-locate Redis nodes with chat servers to minimize network hops, targeting <5ms Pub/Sub latency.
Connection Overhead: Use Redis Cluster with sharding (e.g., 10 shards) to reduce all-to-all connections, balancing load across nodes.
Delivery Gaps: Poll the Inbox table every 1-2 seconds for transient Pub/Sub drops, ensuring no messages are missed.
Scalability: Monitor Redis load and scale clusters horizontally, adding nodes without disrupting chat servers.
Consistency: Use a versioned Inbox table (e.g., message_id, user_id, state, timestamp) to resolve conflicts, with the latest timestamp winning.

Interviewer: How do you handle group chats with multiple clients?
Step 7: Handling Interviewer Probes
For group chats, we store group metadata (member IDs) in a database. When User A sends a message to a group, the server queries the member list, then publishes to each member’s Pub/Sub topic (e.g., pubsub:userB, pubsub:userC). Each user’s clients receive the message via their subscribed servers. State (e.g., read receipts) is synced similarly, with the Inbox table ensuring consistency. To optimize, we cache group memberships in Redis to reduce database queries.
Interviewer: What if a client disconnects frequently?Frequent disconnections (e.g., flaky networks) are handled by the Inbox table. When a client reconnects, it queries /inbox for undelivered messages. Pub/Sub subscriptions are re-established on reconnect, and periodic polling (every 1-2s) catches any missed Pub/Sub messages. We could also use a heartbeat mechanism to detect stale subscriptions and clean them up.
Interviewer: How do you ensure read receipt consistency?Read receipts are written to the Inbox table with a message_id, user_id, and read_timestamp. When User B marks a message as read on their phone, Server 2 writes to the database and publishes to pubsub:userB. Other clients (e.g., tablet on Server 3) receive the update and sync their state. Conflicts are resolved by taking the latest read_timestamp. A distributed lock in Redis ensures atomic updates for high-concurrency cases.
Interviewer: What about storage for the Inbox table?The Inbox table is sharded by user_id in a scalable database like Cassandra, with fields: user_id, message_id, sender_id, content, state (sent/delivered/read), timestamp. For 10B messages/day, with each ~1KB, we need ~10TB/day. Sharding across 100 nodes gives 100GB/node/day, manageable with compression and TTLs (e.g., delete after 30 days).
Step 8: Closing
This design handles multiple clients for 100M users with Redis Pub/Sub, ensuring real-time delivery, consistent state, and scalability. We’ve addressed bottlenecks like latency and connection overhead with mitigations like sharding and polling. Does this cover the scope, or should we dive deeper into, say, database design or handling media sync across clients?

