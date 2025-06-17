##### https://www.youtube.com/watch?v=Zcv_899yqhI&t=1s&ab_channel=HelloInterview-SWEInterviewPreparation

This transcript, presented by Evan, a former Meta staff engineer and co-founder of Hello Interview, outlines the system design for an **ad click aggregator**, a common infrastructure design question in software engineering interviews. Unlike user-facing product designs (e.g., Ticketmaster, Uber), this is an infrastructure-focused system, requiring a modified approach. The transcript follows a structured roadmap: requirements, system interface and data flow, high-level design, and deep dives. Given your prior interest in system architecture diagrams for similar system design transcripts (e.g., Ticketmaster), I’ll assume you want a **system architecture diagram** but will confirm details before generating, as per guidelines. Below, I summarize the design, address requirements, and propose a diagram.

---

### System Design: Ad Click Aggregator Summary

#### 1. Requirements

**Functional Requirements:**
- **Redirect on Ad Click**: Users clicking an ad are redirected to the advertiser’s website (e.g., Nike ad → nike.com).
- **Query Click Metrics**: Advertisers can query aggregated click metrics over time windows (e.g., clicks per ad over a week with hourly granularity, or last day with 1-minute granularity).
  - Minimum granularity: 1 minute.

**Non-Functional Requirements:**
- **Scalability**: Handle peak of **10,000 ad clicks per second** (10K QPS) and **10 million active ads**.
- **Low Latency Analytics Queries**: Query response time < **1 second**.
- **Fault Tolerance and High Data Integrity**: No loss of click data (critical for accurate billing/payouts).
- **Near Real-Time Analytics**: Metrics available within **1-minute granularity**, as real-time as possible.
- **Idempotency of Ad Clicks**: Prevent abuse (e.g., bots/users spamming clicks); count only one click per user per ad impression.
- **Out of Scope**:
  - Ad targeting/serving (deciding which ad to show).
  - Cross-device tracking.
  - Integration with offline channels.
  - Spam detection beyond idempotency.
  - Demographic profiling, conversion tracking.

**Scale Assumptions**:
- **10 million ads** active at any time.
- **10,000 clicks/second** at peak.
- No explicit storage/QPS calculations (per Evan’s advice: math only when it informs design, deferred to deep dives).

#### 2. System Interface & Data Flow

**System Interface**:
- **Inputs**:
  - **Click Data**: From users clicking ads (includes `adId`, optional `userId`, timestamp).
  - **Advertiser Queries**: Requests for click metrics (e.g., clicks by `adId` over time).
- **Outputs**:
  - **User Redirection**: HTTP 302 redirect to advertiser’s website (e.g., nike.com).
  - **Aggregated Click Metrics**: Query results (e.g., total clicks per `adId` per minute/hour).

**Data Flow**:
1. **Click Data Received**: User clicks ad, sends click event.
2. **User Redirected**: System returns 302 redirect to advertiser’s website.
3. **Validate Click**: Ensure idempotency (prevent duplicate clicks).
4. **Log Click Data**: Store raw click event.
5. **Aggregate Click Data**: Process clicks into aggregated metrics (e.g., clicks per `adId` per minute).
6. **Query Aggregated Data**: Advertisers query metrics.

This flow maps to functional requirements:
- Steps 1–2: Redirect on click.
- Steps 3–6: Query click metrics.

#### 3. High-Level Design

**Components**:
- **Browser (Client)**: Sends click events (`POST /click`) with `adId`, `impressionId`.
- **Ad Placement Service**: Out of scope, but provides ads (`adId`, `redirectUrl`, metadata) from **Ads DB**. Generates `impressionId` and signed `impressionId`.
- **Click Processor Service**: Handles click events, validates idempotency, logs to stream, queries Ads DB for `redirectUrl`, returns 302 redirect.
- **Ads DB**: Stores ad data (`adId`, `redirectUrl`, metadata). Optimized for reads (e.g., DynamoDB/PostgreSQL).
- **Click DB (Cassandra, Initial Design)**: Write-optimized, stores raw click events (`eventId`, `adId`, `userId`, `timestamp`).
- **Kinesis Stream**: Buffers click events for real-time processing.
- **Flink Stream Aggregator**: Aggregates clicks in-memory (e.g., clicks per `adId` per minute), writes to OLAP DB.
- **OLAP Database**: Read-optimized, stores aggregated metrics (`adId`, `minute`, `totalClicks`). E.g., Snowflake, ClickHouse.
- **Query Service**: Handles advertiser queries (`GET /metrics`), reads from OLAP DB.
- **Spark (Batch Processing, Reconciliation)**: Periodically (e.g., daily) aggregates raw clicks from S3 for reconciliation.
- **S3**: Stores raw click events dumped from Kinesis for reconciliation.
- **Reconciliation Worker**: Compares Spark aggregations with OLAP DB, corrects inconsistencies.
- **Cron Scheduler**: Triggers Spark reconciliation job.
- **Redis (Idempotency Cache)**: Stores `impressionId` to deduplicate clicks.
- **API Gateway/Load Balancer**: Routes requests, scales services (e.g., AWS API Gateway).

**Initial (Suboptimal) Design**:
- Click Processor writes to Cassandra (write-optimized).
- Spark runs every 5 minutes (Cron-triggered), aggregates clicks, writes to OLAP DB.
- Query Service reads from OLAP DB.
- **Issues**:
  - 5-minute delay violates near real-time requirement.
  - Cassandra range queries/aggregations slow for analytics.
- **Why Separate DBs**:
  - **Contention Reduction**: Write-heavy (Cassandra) and read-heavy (OLAP) workloads separated.
  - **Fault Isolation**: Issues in write path don’t affect read path, and vice versa.

**Optimized Design**:
- Replaces Spark/Cassandra with Kinesis/Flink for real-time aggregation.
- Adds reconciliation via Spark/S3 for data integrity.

**Workflows**:
- **Click and Redirect**:
  1. Browser sends `POST /click` with `adId`, `impressionId`, `signedImpressionId`.
  2. API Gateway routes to Click Processor.
  3. Click Processor verifies signature, checks Redis for `impressionId` (idempotency).
  4. If valid, stores `impressionId` in Redis, puts click event on Kinesis.
  5. Queries Ads DB for `redirectUrl` using `adId`.
  6. Returns 302 redirect to browser.
- **Aggregation**:
  1. Flink reads click events from Kinesis.
  2. Aggregates in-memory (e.g., clicks per `adId` per 1-minute window).
  3. Flushes partial results every 10 seconds, final results at minute end to OLAP DB.
- **Query Metrics**:
  1. Advertiser sends `GET /metrics?adId={id}&startTime={time}&endTime={time}`.
  2. Query Service reads aggregated metrics from OLAP DB, returns results.
- **Reconciliation**:
  1. Kinesis dumps raw clicks to S3.
  2. Cron triggers Spark daily.
  3. Spark aggregates clicks, Reconciliation Worker compares with OLAP DB, corrects errors.

#### 4. Deep Dives

**Deep Dive 1: Near Real-Time Analytics**
- **Problem**: Spark’s 5-minute batch processing delays metrics, violating 1-minute granularity.
- **Solution**: Replace Spark with **Kinesis** and **Flink**.
  - **Kinesis**: Streams click events (sharded by `adId`).
  - **Flink**: Aggregates in-memory with **1-minute aggregation window**, flushes partial results every **10 seconds** (flush interval).
    - E.g., at minute 45, Flink counts clicks for `adId=1`, writes partial counts at 10s, 20s, etc., final count at 60s.
    - UX: Partial data shown as dotted lines in graphs, updated every 10 seconds.
  - **Benefits**: Metrics available ~10 seconds after clicks, fully accurate after 1 minute.
- **Trade-Offs**:
  - Flink overhead higher than Spark for small datasets, but justified for real-time needs.
  - Small latency (10s) acceptable for UX.

**Deep Dive 2: Scalability for 10K Clicks/Second**
- **Solutions**:
  - **Horizontal Scaling**:
    - Click Processor, Ad Placement, Query Services scale dynamically (AWS auto-scaling, CPU/memory thresholds).
    - Each service has a load balancer.
    - API Gateway (AWS) handles routing, additional load balancing.
  - **Kinesis Sharding**:
    - Default limit: **1,000 records/s** or **1 MB/s** per shard.
    - At 10K clicks/s, shard by `adId`, requiring ~10 shards (assuming small event size).
    - Flink tasks read from each shard, aggregate independently (no contention).
  - **OLAP DB Sharding**:
    - Shard by `advertiserId` (common query pattern: advertisers query all their ads).
  - **Hot Shard Problem** (e.g., viral Nike ad with LeBron James):
    - **Issue**: Single `adId` shard overwhelmed, causing latency or data loss.
    - **Solution**: **Celebrity Problem** approach.
      - For hot `adId`, append random number `[0, N]` to shard key (e.g., `adId:123_0`, `adId:123_1`, ..., `adId:123_10`).
      - Click Processor detects hot `adId` (via monitoring), applies `0-N` sharding.
      - Flink task aggregates over multiple shards for hot `adId`, maintaining consistency.
    - **Benefits**: Distributes load across N shards, reduces bottleneck.
- **Math (Deferred)**:
  - Storage for OLAP DB: Estimate based on 10M ads, 1-minute granularity, retention period (e.g., 30 days).
  - Example: 10M ads × 60 min × 24 hr × 30 days × 8 bytes (`adId`, `minute`, `totalClicks`) ≈ 5 TB, sharded across nodes.

**Deep Dive 3: Fault Tolerance and Data Integrity**
- **Problem**: Click loss impacts billing accuracy; Flink failures or transient errors cause inaccuracies.
- **Solutions**:
  - **Kinesis Retention Policy**:
    - Enable **7-day retention** on Kinesis.
    - If Flink task fails, it resumes from last cursor, reprocesses events.
  - **Flink Checkpointing**:
    - Considered but rejected: **1-minute aggregation window** means max 60s of data to reprocess (10K × 60 = 600K events).
    - Checkpointing (e.g., every 15min to S3) unnecessary; reprocessing from stream faster.
    - **Senior/Staff Nuance**: Recognize small window makes checkpointing overkill, avoiding S3 costs.
  - **Periodic Reconciliation**:
    - **Mechanism**:
      - Kinesis dumps raw clicks to S3 via connectors.
      - Cron triggers Spark daily, aggregates clicks.
      - Reconciliation Worker compares Spark results with OLAP DB, overwrites errors.
      - Alerts team on inconsistencies (logging/monitoring).
    - **Benefits**: Catches transient Flink errors, out-of-order events, bad code pushes.
    - **Frequency**: Daily sufficient; hourly if stricter accuracy needed.
  - **Hybrid Architecture**:
    - Combines **Kappa** (Flink real-time) and **Lambda** (Spark batch) elements.
    - **Kappa**: Stream-only, real-time (initial Flink design).
    - **Lambda**: Batch for accuracy, real-time for speed (overwrites after 5min).
    - **Hybrid**: Real-time accuracy via Flink, batch reconciliation for integrity.
    - **Senior/Staff Nuance**: Practicality over architectural purity; hybrid adapts to business needs (real-time + accuracy).

**Deep Dive 4: Idempotency of Ad Clicks**
- **Problem**: Bots/users spamming clicks inflate metrics; logged-out users complicate `userId`-based deduplication.
- **Solution**:
  - **Ad Impression ID**:
    - Ad Placement Service generates unique `impressionId` per ad instance (e.g., Nike ad on Monday vs. Thursday).
    - Browser sends `adId`, `impressionId`, `signedImpressionId` on click.
  - **Signature Verification**:
    - Click Processor verifies `signedImpressionId` using server-side private key.
    - Prevents malicious users from forging `impressionId` (e.g., random IDs via bot).
  - **Redis Cache**:
    - Stores `impressionId` (key) to deduplicate clicks.
    - If `impressionId` exists, drop click; else, store and process.
    - TTL set to match ad campaign duration (e.g., days/weeks).
  - **Alternative (Suboptimal)**:
    - Store `impressionId` in Ads DB, validate legitimacy (adds DB call, slower).
  - **Benefits**:
    - Handles logged-out users (no `userId` needed).
    - Prevents spam without ML-based detection (out of scope).
    - Fast verification via Redis (in-memory).

**Deep Dive 5: Low Latency Analytics Queries**
- **Covered in High-Level Design**: OLAP DB optimized for reads, pre-aggregated metrics (`adId`, `minute`, `totalClicks`).
- **Additional Optimization**:
  - Index OLAP DB on `advertiserId`, `minute` for common queries.
  - Cache frequent queries in Redis (key: `adId_startTime_endTime`, value: results), invalidated on Flink writes.

#### 5. Non-Functional Requirements Check
- **Scalability (10K clicks/s)**: Horizontal scaling, Kinesis sharding, hot shard mitigation.
- **Low Latency Queries (<1s)**: OLAP DB, Redis caching, indexed queries.
- **Fault Tolerance/Data Integrity**: Kinesis retention, reconciliation, no checkpointing.
- **Near Real-Time**: Flink 10s flush intervals, 1-minute window.
- **Idempotency**: `impressionId`, signed verification, Redis deduplication.
- **Out of Scope**: Spam detection, demographic filtering addressed implicitly.

---

### Image Generation Request

Given your prior requests for system architecture diagrams, I propose a **system architecture diagram** visualizing components (Browser, API Gateway, Ad Placement Service, Click Processor, Query Service, Ads DB, Kinesis, Flink, OLAP DB, Redis, S3, Spark, Reconciliation Worker, Cron) and their interactions. I’ll provide a Chart.js placeholder and describe the intended diagram, awaiting confirmation.

#### Proposed Image: System Architecture Diagram

**Description**:
- A diagram showing:
  - **Browser**: Sends `POST /click` and `GET /metrics`.
  - **API Gateway**: Routes requests, load balances.
  - **Ad Placement Service**: Generates `adId`, `impressionId`, `signedImpressionId`, queries Ads DB.
  - **Click Processor Service**: Verifies signatures, checks Redis, logs to Kinesis, returns 302 redirect.
  - **Query Service**: Reads OLAP DB for metrics.
  - **Ads DB**: Stores `adId`, `redirectUrl`.
  - **Kinesis Stream**: Buffers clicks, sharded by `adId` (hot shards: `adId_0-N`).
  - **Flink**: Aggregates clicks (1-min window, 10s flush), writes to OLAP DB.
  - **OLAP DB**: Stores aggregated metrics (`adId`, `minute`, `totalClicks`).
  - **Redis**: Caches `impressionId` for idempotency, query results.
  - **S3**: Stores raw clicks for reconciliation.
  - **Spark**: Aggregates clicks for reconciliation.
  - **Reconciliation Worker**: Corrects OLAP DB errors.
  - **Cron Scheduler**: Triggers Spark.
  - Arrows showing flows:
    - Click: Browser → API Gateway → Click Processor → Redis/Kinesis/Ads DB.
    - Redirect: Click Processor → Browser (302).
    - Aggregation: Kinesis → Flink → OLAP DB.
    - Query: Browser → API Gateway → Query Service → OLAP DB.
    - Reconciliation: Kinesis → S3 → Spark → Reconciliation Worker → OLAP DB.
  - Labels for key processes (e.g., “Signed Impression,” “10s Flush,” “7-Day Retention”).

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
          { "x": 0, "y": 8, "label": "Browser" },
          { "x": 2, "y": 8, "label": "API Gateway" },
          { "x": 4, "y": 9, "label": "Ad Placement" },
          { "x": 4, "y": 7, "label": "Click Processor" },
          { "x": 4, "y": 5, "label": "Query Service" },
          { "x": 6, "y": 9, "label": "Ads DB" },
          { "x": 6, "y": 7, "label": "Kinesis" },
          { "x": 6, "y": 5, "label": "Flink" },
          { "x": 6, "y": 3, "label": "OLAP DB" },
          { "x": 6, "y": 1, "label": "Redis" },
          { "x": 8, "y": 7, "label": "S3" },
          { "x": 8, "y": 5, "label": "Spark" },
          { "x": 8, "y": 3, "label": "Reconciliation" },
          { "x": 8, "y": 1, "label": "Cron" }
        ],
        "backgroundColor": ["#4CAF50", "#2196F3", "#FF9800", "#F44336", "#9C27B0", "#673AB7", "#FF5722", "#009688", "#3F51B5", "#E91E63", "#FFC107", "#795548", "#607D8B", "#00BCD4"],
        "borderColor": ["#388E3C", "#1976D2", "#F57C00", "#D32F2F", "#7B1FA2", "#512DA8", "#E64A19", "#00796B", "#303F9F", "#C2185B", "#FFA000", "#6D4C41", "#546E7A", "#0097A7"],
        "pointRadius": 10,
        "pointHoverRadius": 12
      },
      {
        "label": "Connections",
        "data": [
          { "x": 0, "y": 8 }, { "x": 2, "y": 8 },
          { "x": 2, "y": 8 }, { "x": 4, "y": 9 },
          { "x": 2, "y": 8 }, { "x": 4, "y": 7 },
          { "x": 2, "y": 8 }, { "x": 4, "y": 5 },
          { "x": 4, "y": 9 }, { "x": 6, "y": 9 },
          { "x": 4, "y": 7 }, { "x": 6, "y": 9 },
          { "x": 4, "y": 7 }, { "x": 6, "y": 7 },
          { "x": 4, "y": 7 }, { "x": 6, "y": 1 },
          { "x": 6, "y": 7 }, { "x": 6, "y": 5 },
          { "x": 6, "y": 5 }, { "x": 6, "y": 3 },
          { "x": 4, "y": 5 }, { "x": 6, "y": 3 },
          { "x": 6, "y": 7 }, { "x": 8, "y": 7 },
          { "x": 8, "y": 7 }, { "x": 8, "y": 5 },
          { "x": 8, "y": 5 }, { "x": 8, "y": 3 },
          { "x": 8, "y": 1 }, { "x": 8, "y": 5 },
          { "x": 8, "y": 3 }, { "x": 6, "y": 3 }
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
          { "type": "label", "xValue": 0, "yValue": 8, "content": ["Browser"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#4CAF50" },
          { "type": "label", "xValue": 2, "yValue": 8, "content": ["API Gateway"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#2196F3" },
          { "type": "label", "xValue": 4, "yValue": 9, "content": ["Ad Placement"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF9800" },
          { "type": "label", "xValue": 4, "yValue": 7, "content": ["Click Processor"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#F44336" },
          { "type": "label", "xValue": 4, "yValue": 5, "content": ["Query Service"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#9C27B0" },
          { "type": "label", "xValue": 6, "yValue": 9, "content": ["Ads DB"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#673AB7" },
          { "type": "label", "xValue": 6, "yValue": 7, "content": ["Kinesis"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FF5722" },
          { "type": "label", "xValue": 6, "yValue": 5, "content": ["Flink"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#009688" },
          { "type": "label", "xValue": 6, "yValue": 3, "content": ["OLAP DB"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#3F51B5" },
          { "type": "label", "xValue": 6, "yValue": 1, "content": ["Redis"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#E91E63" },
          { "type": "label", "xValue": 8, "yValue": 7, "content": ["S3"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#FFC107" },
          { "type": "label", "xValue": 8, "yValue": 5, "content": ["Spark"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#795548" },
          { "type": "label", "xValue": 8, "yValue": 3, "content": ["Reconciliation"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#607D8B" },
          { "type": "label", "xValue": 8, "yValue": 1, "content": ["Cron"], "font": { "size": 10 }, "color": "#FFF", "backgroundColor": "#00BCD4" }
        ]
      }
    },
    "scales": {
      "x": { "display": false, "min": -1, "max": 9 },
      "y": { "display": false, "min": 0, "max": 10 }
    }
  }
}
```

**Explanation**:
- **Nodes**: Represent Browser, API Gateway, Ad Placement Service, Click Processor, Query Service, Ads DB, Kinesis, Flink, OLAP DB, Redis, S3, Spark, Reconciliation Worker, Cron.
- **Lines**: Show flows (e.g., Browser to services, Click Processor to Kinesis, Flink to OLAP DB).
- **Colors**: Distinctive for dark/light themes.
- **Labels**: Annotate components.
- **Limitations**: Chart.js is suboptimal; a UML diagram would be more precise.

---

### Confirmation Needed

Please confirm or clarify:
1. Is a **system architecture diagram** the desired image, or do you prefer another type (e.g., flowchart of click processing, data flow diagram)?
2. If a diagram, should it emphasize specific elements (e.g., Kinesis sharding, Flink aggregation, idempotency)?
3. If a chart, what data should it visualize (e.g., click volume over time, shard distribution)?
4. Any style preferences (e.g., minimalist, detailed)?

If you confirm the architecture diagram, I can describe a detailed UML-style image or refine the Chart.js output. If no response, I’ll assume the above chart suffices. Let me know how to proceed!
