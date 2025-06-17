This transcript provides a detailed walkthrough of designing a **Yelp-like local business review system** for a senior-level system design interview, following the **Hello Interview delivery framework**. The speaker designs a system with 100 million daily active users and 10 million businesses, addressing functional and non-functional requirements, core entities, APIs, high-level design, and deep dives into scalability and consistency challenges. Below, I’ll summarize the key points and address the request to **generate an image** based on this transcript. Since the user previously clarified wanting an image, I’ll assume a **system architecture diagram** is appropriate, but I’ll confirm details before proceeding.

---

### System Design: Yelp-Like Business Review System Summary

#### 1. Requirements

**Functional Requirements:**
- **Search Businesses**: Users can search for businesses by name, location (latitude/longitude), and category.
- **View Business Details and Reviews**: Users can view a business’s details and its reviews.
- **Leave Reviews**: Users can leave one review per business, including a 1–5 star rating and optional text.

**Non-Functional Requirements:**
- **High Availability > Strong Consistency**: Eventual consistency is acceptable (e.g., new reviews or business updates can take minutes to propagate globally).
- **Scalability**: Support 100 million daily active users and 10 million businesses.
- **Low Latency Search**: Search queries should complete in <500ms.
- **Durability**: Not explicitly discussed but implied for review persistence.

**Constraints:**
- Each user can leave only one review per business.

#### 2. Core Entities

- **User**: Has an ID (other attributes like passwords are out of scope).
- **Business**: Has an ID, name, description, address, latitude/longitude, category, average rating, number of ratings, and an S3 link for images.
- **Review**: Has an ID, user ID, business ID, rating (1–5), optional text, and created timestamp.

#### 3. API Design

RESTful APIs to satisfy functional requirements:
1. **Search Businesses**:
   - `GET /businesses?category={string}&lat={float}&lon={float}&name={string}&page={int}&limit={int}`
   - Returns: `{ businesses: Array<Partial<Business>> }` (e.g., name, image, short description).
   - Pagination added after feedback to handle large result sets.
2. **View Business Details**:
   - `GET /businesses/:id`
   - Returns: `{ business: Business }` (full business details).
3. **View Reviews**:
   - `GET /businesses/:id/reviews?page={int}&limit={int}`
   - Returns: `{ reviews: Array<Review> }`
   - Split from business details for pagination and lazy loading.
4. **Create Review**:
   - `POST /businesses/:id/reviews`
   - Body: `{ rating: number, text?: string }`
   - Returns: `200 OK` or the created review.
   - Requires authentication (via API gateway) and server-side validation (rating 1–5).

**Feedback Applied:**
- Added pagination (`page`, `limit`) to search and reviews endpoints.
- Split business details and reviews into separate endpoints for performance.

#### 4. High-Level Design

**Components:**
- **Client**: Issues requests (e.g., mobile app, browser).
- **API Gateway**: Routes requests, handles authentication for `POST /reviews`.
- **Business Service**:
  - Handles `GET /businesses` (search) and `GET /businesses/:id` (view details).
  - Queries the database for businesses by name, category, and location.
- **Review Service**:
  - Handles `POST /businesses/:id/reviews` (create review) and `GET /businesses/:id/reviews` (view reviews).
  - Writes reviews to the database and validates data.
- **Primary Database (PostgreSQL)**:
  - **Business Table**: Stores ID, name, description, address, lat/lon, category, average_rating, num_ratings, image_s3_link.
  - **Review Table**: Stores ID, user_id, business_id, rating, text, created_at.
  - **User Table**: Stores ID (minimal for this design).
  - Single database for simplicity, as data volume (10M businesses) doesn’t justify splitting.

**Workflows:**
- **Search**: Client → API Gateway → Business Service → PostgreSQL (query with `SELECT * FROM businesses WHERE category = ? AND name LIKE ? AND lat/lon range`).
- **View Business/Reviews**: Client → API Gateway → Business Service → PostgreSQL (join `businesses` and `reviews` by business ID).
- **Create Review**: Client → API Gateway (auth) → Review Service → PostgreSQL (insert into `reviews`).

**Notes:**
- Initial search query is inefficient (no indexing); optimized in deep dives.
- Services are split for independent scaling (search/view vs. review writes).

#### 5. Deep Dives

**Deep Dive 1: Calculate and Update Average Rating**
- **Problem**: Ensure `average_rating` in the `businesses` table is accurate and available for search results, updated “up to the minute.”
- **Solution**: Use a SQL transaction in the Review Service:
  - Insert new review into `reviews`.
  - Update `businesses`:
    - Increment `num_ratings`.
    - Recalculate `average_rating`: `(current_average * num_ratings + new_rating) / (num_ratings + 1)`.
  - Transaction ensures atomicity.
- **Justification**: With ~5M reviews/day (~50 writes/sec), PostgreSQL can handle this without a message queue or cron job.
- **Feedback**: Should mention race conditions (addressed in next question).

**Deep Dive 2: Handle Concurrent Review Submissions**
- **Problem**: Multiple users submitting reviews for the same business simultaneously could cause race conditions (e.g., Transaction B overwrites Transaction A’s `average_rating` update).
- **Solution**: Use either:
  - **Pessimistic Locking**: Lock the business row during the transaction to prevent concurrent updates.
  - **Optimistic Concurrency Control (OCC)**: Check `average_rating` and `num_ratings` at transaction start; retry if changed at commit.
- **Choice**: OCC preferred due to low review throughput, minimizing lock contention.
- **Feedback**: Solution is solid, showing awareness of concurrency.

**Deep Dive 3: Ensure One Review Per User Per Business**
- **Problem**: Enforce the constraint that a user can only leave one review per business.
- **Solution**: Add a **composite unique constraint** on `reviews(user_id, business_id)` in PostgreSQL.
  - Database rejects duplicate reviews, returning an error.
  - Supplement with client-side and Review Service logic to prevent users from attempting duplicates.
- **Justification**: Database constraint is robust against bypasses (e.g., API calls, backfills) compared to application logic alone.
- **Feedback**: Clarify composite constraint (not separate constraints on `user_id` and `business_id`); handle duplicate errors gracefully.

**Deep Dive 4: Improve Search for Complex Queries**
- **Problem**: Initial search query (`SELECT * WHERE category = ? AND name LIKE ? AND lat/lon range`) is slow without proper indexing, especially for 10M businesses.
- **Solution**: Use PostgreSQL extensions instead of Elasticsearch:
  - **PostGIS Extension**: Enables geospatial indexing on `lat/lon` for efficient location-based queries.
  - **Full-Text Search**: Uses GIN indexes for inverted indexes on `name` and `category`.
  - Queries use PostGIS for location and full-text search for name/category, achieving <500ms latency.
- **Justification**: Avoids Elasticsearch to reduce complexity (no CDC or dual-store consistency needed); PostgreSQL extensions are sufficient for scale.
- **Feedback**: Could elaborate on indexing details if probed by interviewer.

---

### Image Generation Request

The user requested an **image** based on the transcript. Given the context, a **system architecture diagram** is the most relevant, as it visualizes the components (Client, API Gateway, Business Service, Review Service, PostgreSQL) and their interactions. However, since I must confirm before generating images, I’ll propose a Chart.js-based diagram as a placeholder (since Chart.js is used for visualizations in this context) and describe what it would represent. If the user confirms or prefers a different image (e.g., a flowchart or UI mockup), I’ll adjust accordingly.

#### Proposed Image: System Architecture Diagram

**Description**:
- A diagram showing:
  - **Client** (browser/mobile) sending requests.
  - **API Gateway** routing to services.
  - **Business Service** querying PostgreSQL for search and business details.
  - **Review Service** handling review creation and retrieval.
  - **PostgreSQL** with `businesses`, `reviews`, and `users` tables.
  - Arrows indicating request flows (e.g., `GET /businesses` → Business Service → PostgreSQL).
  - Optional: Highlight PostGIS and full-text search extensions in PostgreSQL.

**Chart.js Placeholder** (since image generation requires confirmation):
I’ll create a simple **flowchart-like visualization** using Chart.js’s `scatter` chart to represent components as nodes and connections as lines, simulating the architecture.

```chartjs
{
  "type": "scatter",
  "data": {
    "datasets": [
      {
        "label": "Components",
        "data": [
          { "x": 0, "y": 4, "label": "Client" },
          { "x": 2, "y": 4, "label": "API Gateway" },
          { "x": 4, "y": 5, "label": "Business Service" },
          { "x": 4, "y": 3, "label": "Review Service" },
          { "x": 6, "y": 4, "label": "PostgreSQL" }
        ],
        "backgroundColor": ["#4CAF50", "#2196F3", "#FF9800", "#F44336", "#3F51B5"],
        "borderColor": ["#388E3C", "#1976D2", "#F57C00", "#D32F2F", "#303F9F"],
        "pointRadius": 10,
        "pointHoverRadius": 12
      },
      {
        "label": "Connections",
        "data": [
          { "x": 0, "y": 4 }, { "x": 2, "y": 4 },
          { "x": 2, "y": 4 }, { "x": 4, "y": 5 },
          { "x": 2, "y": 4 }, { "x": 4, "y": 3 },
          { "x": 4, "y": 5 }, { "x": 6, "y": 4 },
          { "x": 4, "y": 3 }, { "x": 6, "y": 4 }
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
          { "type": "label", "xValue": 0, "yValue": 4, "content": ["Client"], "font": { "size": 14 }, "color": "#FFF", "backgroundColor": "#4CAF50" },
          { "type": "label", "xValue": 2, "yValue": 4, "content": ["API Gateway"], "font": { "size": 14 }, "color": "#FFF", "backgroundColor": "#2196F3" },
          { "type": "label", "xValue": 4, "yValue": 5, "content": ["Business Service"], "font": { "size": 14 }, "color": "#FFF", "backgroundColor": "#FF9800" },
          { "type": "label", "xValue": 4, "yValue": 3, "content": ["Review Service"], "font": { "size": 14 }, "color": "#FFF", "backgroundColor": "#F44336" },
          { "type": "label", "xValue": 6, "yValue": 4, "content": ["PostgreSQL"], "font": { "size": 14 }, "color": "#FFF", "backgroundColor": "#3F51B5" }
        ]
      }
    },
    "scales": {
      "x": { "display": false, "min": -1, "max": 7 },
      "y": { "display": false, "min": 2, "max": 6 }
    }
  }
}
```

**Explanation**:
- **Nodes**: Represent Client, API Gateway, Business Service, Review Service, and PostgreSQL.
- **Lines**: Show request flows (e.g., Client to API Gateway, API Gateway to services, services to database).
- **Colors**: Distinctive for visibility in dark/light themes.
- **Labels**: Annotate each node for clarity.

**Limitations**: Chart.js isn’t ideal for architecture diagrams (better for data plots). If you confirm, I can describe a more detailed image (e.g., a UML-like diagram) or adjust the chart.

---

### Confirmation Needed

Please confirm or clarify:
1. Is a **system architecture diagram** what you want, or do you prefer another image (e.g., flowchart, UI mockup)?
2. If a diagram, should it include specific details (e.g., PostGIS, authentication flow)?
3. If you meant a **chart** (e.g., latency or throughput), what data should it visualize?
4. Any style preferences (e.g., minimalist, detailed)?

Once you confirm, I’ll generate the appropriate image or chart. If you don’t respond, I’ll assume the above Chart.js diagram suffices. Let me know how to proceed!
