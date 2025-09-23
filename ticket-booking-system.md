
# Online Ticketing Platform (like ticketmaster.com)
<img width="1668" height="854" alt="image" src="https://github.com/user-attachments/assets/ba9180ab-2da6-4000-847f-085d5abed089" />
---

## 1. Project Overview

Design a **highly available, horizontally scalable online ticketing platform** where users can browse events and purchase tickets.

* **Goal**: Showcase solid distributed-system thinking (load handling, consistency, caching, queues) in a manageable prototype.
* **Scope**: Focus on event creation, ticket inventory management, and a simple purchase workflow. Exclude advanced features like dynamic pricing or lottery pre-sales.

---

## 2. Core Functional Requirements

### A. Event Management

* **Create Event**: Promoter/organizer can create events with

  * Event name, date/time, venue (with seating capacity or general admission).
  * Ticket price and total ticket count.
* **Update/Delete Event**: Organizer can modify or cancel events before ticket sales start.
* **View Events**: Public API for users to browse/search events.

### B. Ticket Sales

* **Purchase Tickets**: Logged-in user can buy up to a limited number of tickets per event.
* **Inventory Check**: Ensure no overselling (atomic decrement).
* **Order Confirmation**: On success, generate a digital ticket (QR code or unique ID).

### C. User Management (Basic)

* User registration/login (email + password or OAuth).
* View past purchases.

---

## 3. Non-Functional Requirements

| Category         | Requirement                                                                                   |
| ---------------- | --------------------------------------------------------------------------------------------- |
| **Availability** | Target 99.9% uptime. Design for fault tolerance and multi-AZ deployment.                      |
| **Scalability**  | Must handle sudden spikes (e.g., popular concert release) by auto-scaling.                    |
| **Consistency**  | Strong consistency for ticket inventory. Eventual consistency acceptable for browsing events. |
| **Latency**      | API responses under 200 ms for browsing; purchase flow under 2 s.                             |
| **Security**     | HTTPS, JWT-based authentication, basic rate limiting to prevent abuse.                        |

---

## 4. Simplified Architecture (High Level)

**Clients**

* Web or mobile app.

**API Gateway / Load Balancer**

* Routes traffic to microservices and supports horizontal scaling.

**Services**

1. **Event Service**

   * CRUD for events.
   * Stores event details in a relational DB (e.g., PostgreSQL/MySQL).
   * Publishes event-created/updated messages to a queue (for cache updates, analytics).

2. **Ticketing Service**

   * Manages ticket inventory and purchase flow.
   * Uses Redis for:

     * **Inventory cache** to handle high read/write load.
     * **Distributed lock or atomic decrement** to prevent overselling.
   * Persists successful orders to the primary database.

3. **User Service**

   * Handles authentication/authorization and user profiles.

**Databases**

* **Primary DB**: Relational (MySQL/PostgreSQL) with strong ACID guarantees for orders & events.
* **Cache Layer**: Redis or Memcached for hot event listings and available tickets.

**Async Components**

* **Message Queue (Kafka/RabbitMQ/SQS)** for:

  * Cache invalidation.
  * Email confirmation jobs.

**CDN**

* For static assets and ticket PDFs/QR codes.

---

## 5. Key Design Considerations

* **High Availability**

  * Deploy services in multiple Availability Zones.
  * Use load balancers with health checks and auto-scaling groups.

* **Scalable Inventory Locking**

  * Option 1: Redis `SETNX` with short TTL to “soft hold” tickets during checkout.
  * Option 2: Database row-level locking (e.g., `SELECT … FOR UPDATE`) for final commit.

* **Failure Handling**

  * Idempotent order API to safely retry failed requests.
  * Dead-letter queues for message failures.

* **Monitoring & Alerts**

  * Metrics: request latency, error rates, stock depletion.
  * Tools: Prometheus + Grafana or AWS CloudWatch.

---

## 6. Nice-to-Have (Optional for Prototype)

* Simple seating chart (general admission is enough for MVP).
* Email/SMS notifications.
* Payment gateway simulation (mock service, not real payments).

---

## 7. Deliverables for Interview

* **Architecture Diagram**: Show clients, API gateway, services, DB, cache, queue, and CDN.
* **ER Diagram**: Tables for `Users`, `Events`, `Tickets`, `Orders`.
* **Sequence Diagram**: Ticket purchase flow with locking.
* **Brief Scaling Strategy**: How to add more nodes, handle spikes, and recover from failure.

---

## 8. Database Schema (with `user_type`)

### Tables

```sql
-- Users table with user_type
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    user_type     ENUM('guest','organizer','admin') DEFAULT 'guest' NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE events (
    id            BIGSERIAL PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    description   TEXT,
    venue         VARCHAR(255) NOT NULL,
    event_date    TIMESTAMP NOT NULL,
    ticket_price  DECIMAL(10,2) NOT NULL,
    total_tickets INT NOT NULL,
    created_by    BIGINT REFERENCES users(id),
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT REFERENCES users(id),
    event_id      BIGINT REFERENCES events(id),
    quantity      INT NOT NULL,
    total_price   DECIMAL(10,2) NOT NULL,
    status        ENUM('PENDING','CONFIRMED','CANCELLED') DEFAULT 'PENDING',
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tickets (
    id            BIGSERIAL PRIMARY KEY,
    order_id      BIGINT REFERENCES orders(id),
    ticket_code   VARCHAR(64) UNIQUE NOT NULL,
    issued_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Sample Data Inserts

```sql
INSERT INTO users (name, email, password_hash, user_type)
VALUES
  ('Alice Buyer', 'alice@example.com', 'hashed_pw1', 'guest'),
  ('Bob Organizer', 'bob@events.com', 'hashed_pw2', 'organizer'),
  ('Carol Admin', 'carol@admin.com', 'hashed_pw3', 'admin');

INSERT INTO events (name, description, venue, event_date, ticket_price, total_tickets, created_by)
VALUES
  ('Rock Night', 'Live rock concert', 'City Arena', '2025-12-10 20:00:00', 75.00, 500, 2),
  ('Tech Conference', 'Annual tech meetup', 'Convention Center', '2026-01-15 09:00:00', 120.00, 300, 2);

INSERT INTO orders (user_id, event_id, quantity, total_price, status)
VALUES
  (1, 1, 2, 150.00, 'CONFIRMED');

INSERT INTO tickets (order_id, ticket_code)
VALUES
  (1, 'TCK-ABC123'),
  (1, 'TCK-ABC124');
```

---

## 9. REST API Endpoints (with Sample Request/Response)

### User APIs

**POST /api/v1/users/register**
Request:

```json
{
  "name": "Alice Buyer",
  "email": "alice@example.com",
  "password": "plaintext_password",
  "user_type": "guest"
}
```

Response:

```json
{
  "id": 1,
  "name": "Alice Buyer",
  "email": "alice@example.com",
  "user_type": "guest",
  "created_at": "2025-09-22T10:00:00Z"
}
```

**POST /api/v1/users/login**
Request:

```json
{
  "email": "alice@example.com",
  "password": "plaintext_password"
}
```

Response:

```json
{
  "token": "jwt-token-string",
  "user": {
    "id": 1,
    "name": "Alice Buyer",
    "user_type": "guest"
  }
}
```

---

### Event APIs

**POST /api/v1/events**  *(organizer only)*
Request:

```json
{
  "name": "Rock Night",
  "description": "Live rock concert",
  "venue": "City Arena",
  "event_date": "2025-12-10T20:00:00Z",
  "ticket_price": 75.00,
  "total_tickets": 500
}
```

Response:

```json
{
  "id": 1,
  "name": "Rock Night",
  "ticket_price": 75.00,
  "total_tickets": 500,
  "created_by": 2
}
```

**GET /api/v1/events**
Response:

```json
[
  {
    "id": 1,
    "name": "Rock Night",
    "venue": "City Arena",
    "event_date": "2025-12-10T20:00:00Z",
    "ticket_price": 75.00,
    "remaining_tickets": 498
  }
]
```

---

### Ticket Purchase API

**POST /api/v1/orders**
Request:

```json
{
  "event_id": 1,
  "quantity": 2
}
```

Response:

```json
{
  "order_id": 10,
  "status": "CONFIRMED",
  "tickets": [
    {"ticket_code": "TCK-XYZ111"},
    {"ticket_code": "TCK-XYZ112"}
  ]
}
```

---

### Admin API Example

**GET /api/v1/admin/users** *(admin only)*
Response:

```json
[
  {
    "id": 1,
    "name": "Alice Buyer",
    "email": "alice@example.com",
    "user_type": "guest"
  },
  {
    "id": 2,
    "name": "Bob Organizer",
    "email": "bob@events.com",
    "user_type": "organizer"
  }
]
```

---






