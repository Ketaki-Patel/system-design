
# Location Based Service


## Location-Based Service: High-Level System Design

This system powers a Yelp-like platform where users can discover nearby businesses, business owners can manage their listings, and everyone can share reviews.
The architecture is built with microservices, ensuring each responsibility—users, places, reviews, and search—scales independently while remaining consistent.

---

<img width="818" height="782" alt="image" src="https://github.com/user-attachments/assets/ab1b5bc7-8f95-488e-beb6-a033873eac51" />

---


### 1. User Management & Authentication

**Service:** **User Service**
**Data Store:** MySQL (Users table)

*Workflow*

1. **Registration / Update** – A client (mobile or web) calls the User Service through the load balancer to create or update a profile.
2. **Authentication** – On login, the User Service validates credentials and issues an access token (e.g., JWT).
3. Other services use that token to authorize actions like posting reviews or claiming a business.

*Purpose*

* Central place to manage identities and permissions.
* Supplies user data (IDs, profiles) to other services without exposing passwords or sensitive details.

---

### 2. Business Listings (Places)

**Service:** **Place Service**
**Stores:** MySQL (source of truth), Elasticsearch (search index)
**Messaging:** Kafka

*Workflow*

1. **Add or Update Place** – A business owner submits details (name, address, geo-coordinates, categories) to the Place Service.
2. **Persist** – Place Service writes the canonical record to MySQL.
3. **Publish Event** – After a successful commit, it sends a `place.created` or `place.updated` event to Kafka.
4. **Index Sync** – The Indexer Service consumes the event and updates or creates the corresponding document in Elasticsearch.

*Outcome*

* MySQL remains the authoritative database.
* Elasticsearch provides fast, geo-aware searches using the latest updates.

---

### 3. Reviews

**Service:** **Review Service**
**Stores:** MySQL (reviews table), Elasticsearch (denormalized review data)
**Messaging:** Kafka

*Workflow*

1. **Add Review** – Authenticated users post ratings and comments via the Review Service.
2. **Persist** – Data is saved in MySQL as the source of truth.
3. **Publish Event** – A `review.created` event goes to Kafka.
4. **Search Index Update** – The Indexer consumes the event and updates the associated place document in Elasticsearch with aggregates (e.g., average rating, review count).
5. **Retrieve Reviews** – When a client requests reviews, the Review Service reads directly from MySQL for full detail or can serve cached aggregates from Elasticsearch.

---

### 4. Location Search

**Service:** **Search Service**
**Store:** Elasticsearch

*Workflow*

1. **Nearby Search** – A user queries for “coffee near me.”
2. **Elasticsearch Query** – The Search Service runs a geo-distance and text query on the Places index.
3. **Result Delivery** – Matches (including aggregated ratings and categories) are returned to the client.

*Index Maintenance*

* **Incremental Updates:** Real-time events from Place and Review Services keep Elasticsearch nearly in sync.
* **Full Rebuilds:** Periodic full indexing from MySQL ensures long-term consistency in case of missed events.

---

### 5. Architectural Overview

**Key Components**

* **Client (Web/Mobile):** Entry point for all actions—registration, adding places, posting reviews, and searching.
* **Load Balancer:** Routes requests to the appropriate microservice.
* **Microservices:**

  * *User Service* – user CRUD and authentication.
  * *Place Service* – business listing management.
  * *Review Service* – add/get reviews.
  * *Search Service* – location and keyword search.
  * *Indexer Service* – listens to Kafka and updates Elasticsearch.
* **Databases:**

  * *MySQL* – single source of truth for users, places, and reviews.
  * *Elasticsearch* – optimized for full-text and geo search.
* **Messaging:** *Kafka* – reliable event bus to propagate updates from MySQL to Elasticsearch.

---

### End-to-End User Journeys

| Use Case                                | Path Through the System                                                           |
| --------------------------------------- | --------------------------------------------------------------------------------- |
| **User registers or updates profile**   | Client → Load Balancer → User Service → MySQL                                     |
| **Business owner adds/edits a place**   | Client → Load Balancer → Place Service → MySQL → Kafka → Indexer → Elasticsearch  |
| **User posts a review**                 | Client → Load Balancer → Review Service → MySQL → Kafka → Indexer → Elasticsearch |
| **User searches for businesses nearby** | Client → Load Balancer → Search Service → Elasticsearch                           |

---

### Why This Design Works

* **Separation of Concerns:** Each service focuses on a single domain (users, places, reviews, search).
* **Scalability:** Search traffic scales independently of writes.
* **Consistency:** MySQL is the authoritative source; Kafka and the Indexer keep Elasticsearch closely synchronized.
* **Performance:** Elasticsearch delivers low-latency geo and text queries suitable for real-time location discovery.

---

This architecture provides the foundation for a robust, Yelp-like location platform—supporting millions of users, high search volumes, and frequent updates while remaining maintainable and easy to evolve.

