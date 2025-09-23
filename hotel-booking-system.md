
Hotel Booking System Architecture
<img width="1368" height="720" alt="image" src="https://github.com/user-attachments/assets/3f0154f2-a63a-4651-980b-1c148d2d9c51" />

[master requirement doc](https://claude.ai/public/artifacts/e6d8ebd9-3925-44a9-b924-4aa3736d9a23)

# Hotel Discovery & Booking Platform – System Requirements

## 1. Goal
Design a Hotel Discovery & Booking Platform where travelers can search for hotels in a city, view live room availability, make reservations with payments, and leave community reviews. Hotel owners can manage their listings and room inventory.

## 2. Functional Requirements (Use Cases)

### 2.1 Hotel Search & Discovery
- **UC-001**: Users can search for hotels by city name  
- **UC-002**: Search results include hotel details with total available room count and average rating  
- **UC-003**: Users can view detailed hotel information including amenities and photos (out of scope)
- **UC-004**: Users can read all reviews for a specific hotel  

### 2.2 Hotel Management (Owner)
- **UC-005**: Hotel owners can register and authenticate  
- **UC-006**: Hotel owners can add new hotels with name, description, and address  
- **UC-007**: Hotel owners can update hotel details and amenities (out of scope)
- **UC-008**: Hotel owners can manage room types and set pricing  

### 2.3 Room Booking & Availability
- **UC-009**: Users can view room types and current availability for a hotel  
- **UC-010**: Users can select check-in/check-out dates to see availability  
- **UC-011**: Users can make room reservations with payment processing  
- **UC-012**: Users can pay via credit card (Stripe) or bank transfer (Plaid)  
- **UC-013**: System updates room availability in real-time after bookings  

### 2.4 Review System
- **UC-014**: Registered users can write reviews with ratings (1–5 stars) and comments  
- **UC-015**: All users can view reviews sorted by most recent or highest rating  
- **UC-016**: System calculates and displays average ratings for hotels  

### 2.5 User Management
- **UC-017**: Users can register with email and password  
- **UC-018**: Users can login and manage their profiles  
- **UC-019**: System supports role-based access (guest, hotel owner, admin)  
- **UC-020**: Users can view their booking history  

## 3. Non-Functional Requirements
- **Scalability**: Handle high read traffic for search and room availability  
- **Low Latency**: Hotel search results should appear within ~200 ms  
- **Consistency**: Eventual consistency acceptable for review counts and availability  
- **Security**: Role-based access control and secure payment processing  
- **Availability**: 99.9 % uptime for booking and search services  

## 4. Microservices Architecture

| Service                 | Port | Responsibilities                                | Database         | Caching                       | Events                           |
|-------------------------|------|-------------------------------------------------|------------------|-------------------------------|-----------------------------------|
| **Hotel Service**       | 8001 | CRUD hotels, hotel management                   | Hotels DB (MySQL)| Redis (hotel profiles)        | HotelCreated, HotelUpdated        |
| **Room/Booking Service**| 8002 | Room types, availability, bookings, payments    | Rooms/Booking DB | Redis (availability counts)   | RoomAvailabilityChanged           |
| **Review Service**      | 8003 | Store reviews, calculate aggregates             | Reviews DB       | Redis (review aggregates)     | ReviewAdded                       |
| **Hotel Search Service**| 8004 | Search orchestration, result enrichment         | Elasticsearch    | Redis (fetch cached data)     | –                                 |
| **User Auth Service**   | 8005 | Authentication, authorization, user management  | Users DB         | Redis (sessions & tokens)     | –                                 |
| **Search Index Update** | 8006 | Kafka consumer, ES index management             | –                | –                             | Consumes all events               |

## 5. Database Schema & Sample Data

### 5.1 Hotels Database (Hotel Service)
```sql
-- Hotels table
CREATE TABLE hotels (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  owner_id BIGINT NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  address TEXT NOT NULL,
  city VARCHAR(100) NOT NULL,
  state VARCHAR(50),
  country VARCHAR(50) DEFAULT 'USA',
  postal_code VARCHAR(20),
  phone VARCHAR(20),
  email VARCHAR(100),
  amenities JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);


-- Sample Data
INSERT INTO hotels (owner_id, name, description, address, city, state, postal_code, phone, amenities) VALUES
(101, 'Grand Palace Hotel', 'Luxury hotel in downtown', '123 Market St', 'San Francisco', 'CA', '94102', '+1-415-555-0101', '["WiFi", "Pool", "Spa", "Restaurant"]'),
(102, 'Sea Breeze Inn', 'Coastal hotel with ocean views', '456 Ocean Ave', 'Los Angeles', 'CA', '90210', '+1-310-555-0102', '["WiFi", "Beach Access", "Restaurant"]'),
(103, 'Mountain View Lodge', 'Peaceful retreat in the mountains', '789 Alpine Rd', 'Denver', 'CO', '80202', '+1-303-555-0103', '["WiFi", "Hiking", "Fireplace"]');
```

### 5.2 Rooms/Booking Database (Room/Booking Service)
```sql
-- Room Types table
CREATE TABLE room_types (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  hotel_id BIGINT NOT NULL,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  capacity INT NOT NULL,
  price_per_night DECIMAL(10,2) NOT NULL,
  amenities JSON,
  FOREIGN KEY (hotel_id) REFERENCES hotels(id)
);

-- Room Inventory table
CREATE TABLE room_inventory (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  room_type_id BIGINT NOT NULL,
  date DATE NOT NULL,
  total_rooms INT NOT NULL,
  available_rooms INT NOT NULL,
  FOREIGN KEY (room_type_id) REFERENCES room_types(id),
  UNIQUE KEY unique_room_date (room_type_id, date)
);

-- Bookings table
CREATE TABLE bookings (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  hotel_id BIGINT NOT NULL,
  room_type_id BIGINT NOT NULL,
  check_in_date DATE NOT NULL,
  check_out_date DATE NOT NULL,
  num_rooms INT NOT NULL DEFAULT 1,
  total_amount DECIMAL(10,2) NOT NULL,
  payment_status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
  payment_method ENUM('stripe', 'plaid') NOT NULL,
  payment_transaction_id VARCHAR(255),
  booking_status ENUM('confirmed', 'cancelled', 'completed') DEFAULT 'confirmed',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (room_type_id) REFERENCES room_types(id)
);

-- Sample Data
INSERT INTO room_types (hotel_id, name, description, capacity, price_per_night, amenities) VALUES
(1, 'Standard Room', 'Comfortable room with city view', 2, 150.00, '["WiFi", "TV", "AC"]'),
(1, 'Deluxe Suite', 'Spacious suite with premium amenities', 4, 300.00, '["WiFi", "TV", "AC", "Minibar", "Balcony"]'),
(2, 'Ocean View Room', 'Room with stunning ocean views', 2, 200.00, '["WiFi", "TV", "AC", "Ocean View"]'),
(3, 'Mountain Cabin', 'Rustic cabin with mountain views', 4, 180.00, '["WiFi", "Fireplace", "Mountain View"]');

INSERT INTO room_inventory (room_type_id, date, total_rooms, available_rooms) VALUES
(1, '2025-02-01', 20, 15),
(1, '2025-02-02', 20, 18),
(2, '2025-02-01', 5, 3),
(2, '2025-02-02', 5, 4),
(3, '2025-02-01', 15, 12),
(4, '2025-02-01', 8, 6);

INSERT INTO bookings (user_id, hotel_id, room_type_id, check_in_date, check_out_date, num_rooms, total_amount, payment_status, payment_method) VALUES
(201, 1, 1, '2025-02-01', '2025-02-03', 1, 300.00, 'completed', 'stripe'),
(202, 2, 3, '2025-02-05', '2025-02-07', 2, 400.00, 'completed', 'plaid');
```

### 5.3 Reviews Database (Review Service)
``` sql
CREATE TABLE reviews (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  hotel_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL,
  booking_id BIGINT,
  rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  title VARCHAR(255),
  comment TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (hotel_id) REFERENCES hotels(id),
  FOREIGN KEY (booking_id) REFERENCES bookings(id)
);

-- Sample Data
INSERT INTO reviews (hotel_id, user_id, booking_id, rating, title, comment) VALUES
(1, 201, 1, 5, 'Excellent Stay!', 'Amazing service and beautiful rooms. Highly recommend!'),
(1, 203, NULL, 4, 'Great Location', 'Perfect location in downtown. Staff was very helpful.'),
(2, 202, 2, 5, 'Perfect Ocean Views', 'Woke up to stunning ocean views every morning. Loved it!'),
(3, 204, NULL, 4, 'Peaceful Retreat', 'Great for a weekend getaway. Very quiet and relaxing.');
```
### 5.4 Users Database (User Auth Service)
```
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(20),
  role ENUM('guest', 'hotel_owner', 'admin') DEFAULT 'guest',
  email_verified BOOLEAN DEFAULT FALSE,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Sample Data
INSERT INTO users (email, password_hash, first_name, last_name, phone, role, email_verified) VALUES
('john.doe@email.com', '$2b$12$hash1', 'John', 'Doe', '+1-555-0001', 'guest', TRUE),
('owner1@hotel.com', '$2b$12$hash2', 'Alice', 'Smith', '+1-555-0002', 'hotel_owner', TRUE),
('owner2@hotel.com', '$2b$12$hash3', 'Bob', 'Johnson', '+1-555-0003', 'hotel_owner', TRUE),
('admin@platform.com', '$2b$12$hash4', 'Admin', 'User', '+1-555-0004', 'admin', TRUE);

```

## 6. API Endpoints by Service
### 6.1 Hotel Service APIs
```
POST   /api/v1/hotels                    # Create new hotel
GET    /api/v1/hotels/{hotelId}          # Get hotel details
PUT    /api/v1/hotels/{hotelId}          # Update hotel
DELETE /api/v1/hotels/{hotelId}          # Delete hotel
GET    /api/v1/hotels/owner/{ownerId}    # Get hotels by owner

POST   /api/v1/hotels/{hotelId}/room-types     # Add room type
GET    /api/v1/hotels/{hotelId}/room-types     # Get room types
PUT    /api/v1/room-types/{roomTypeId}         # Update room type
DELETE /api/v1/room-types/{roomTypeId}         # Delete room type
```

### 6.2 Room/Booking Service APIs
```
GET    /api/v1/hotels/{hotelId}/availability    # Get room availability
POST   /api/v1/room-inventory                   # Update inventory (owner)
GET    /api/v1/room-types/{roomTypeId}/calendar # Get availability calendar

POST   /api/v1/bookings                         # Create booking
GET    /api/v1/bookings/{bookingId}             # Get booking details
PUT    /api/v1/bookings/{bookingId}/cancel      # Cancel booking
GET    /api/v1/users/{userId}/bookings          # Get user bookings
GET    /api/v1/hotels/{hotelId}/bookings        # Get hotel bookings (owner)

POST   /api/v1/payments/stripe                  # Process Stripe payment
POST   /api/v1/payments/plaid                   # Process Plaid payment
GET    /api/v1/payments/{transactionId}         # Get payment status
```

### 6.3 Review Service APIs
```
POST   /api/v1/reviews                          # Create review
GET    /api/v1/hotels/{hotelId}/reviews         # Get hotel reviews
GET    /api/v1/reviews/{reviewId}               # Get specific review
PUT    /api/v1/reviews/{reviewId}               # Update review
DELETE /api/v1/reviews/{reviewId}               # Delete review
GET    /api/v1/users/{userId}/reviews           # Get user reviews

GET    /api/v1/hotels/{hotelId}/rating-summary  # Get rating summary
```

### 6.4 Hotel Search Service APIs
```
GET    /api/v1/search/hotels                    # Search hotels by city
GET    /api/v1/search/hotels/{hotelId}/summary  # Get hotel summary with aggregates
POST   /api/v1/search/filters                   # Advanced search with filters
GET    /api/v1/search/suggestions               # Get search suggestions
```

###  6.5 User Auth Service APIs
```
POST   /api/v1/auth/register         # User registration
POST   /api/v1/auth/login            # User login
POST   /api/v1/auth/logout           # User logout
POST   /api/v1/auth/refresh          # Refresh JWT token
POST   /api/v1/auth/forgot-password  # Forgot password
POST   /api/v1/auth/reset-password   # Reset password

GET    /api/v1/users/profile         # Get user profile
PUT    /api/v1/users/profile         # Update user profile
GET    /api/v1/users/{userId}        # Get user details (admin)
PUT    /api/v1/users/{userId}/role   # Update user role (admin)
```

Below is a GitHub-flavored Markdown (GFM) document you can drop directly into a README.md.
It groups the APIs by service and includes **sample request bodies** and **sample JSON responses with example values** so it’s easy to copy/paste into docs or test tools like Postman.

````markdown
# Hotel Booking Platform – REST API Reference

All endpoints are prefixed with:  
`https://api.example.com/api/v1/`

---

## 6.1 Hotel Service APIs

### Create New Hotel
`POST /hotels`

**Request**
```json
{
  "name": "Sunset Resort",
  "ownerId": "own_12345",
  "city": "San Diego",
  "address": "123 Ocean Drive",
  "description": "Beachfront hotel with pool and spa"
}
````

**Response**

```json
{
  "hotelId": "hot_98765",
  "name": "Sunset Resort",
  "ownerId": "own_12345",
  "city": "San Diego",
  "address": "123 Ocean Drive",
  "description": "Beachfront hotel with pool and spa",
  "createdAt": "2025-09-22T17:00:00Z"
}
```

### Get Hotel Details

`GET /hotels/{hotelId}`

**Response**

```json
{
  "hotelId": "hot_98765",
  "name": "Sunset Resort",
  "ownerId": "own_12345",
  "city": "San Diego",
  "address": "123 Ocean Drive",
  "description": "Beachfront hotel with pool and spa",
  "rating": 4.6
}
```

### Update / Delete Hotel

`PUT /hotels/{hotelId}` and `DELETE /hotels/{hotelId}`
Request body for update is identical to **Create New Hotel**.

### Get Hotels by Owner

`GET /hotels/owner/{ownerId}` → Returns a JSON array of hotel objects.

---

### Room Types

**Add Room Type**
`POST /hotels/{hotelId}/room-types`

```json
{
  "name": "Deluxe Suite",
  "capacity": 4,
  "basePrice": 220.0
}
```

**Get Room Types**
`GET /hotels/{hotelId}/room-types`

**Update / Delete Room Type**
`PUT /room-types/{roomTypeId}` / `DELETE /room-types/{roomTypeId}`

---

## 6.2 Room & Booking Service APIs

### Check Availability

`GET /hotels/{hotelId}/availability?start=2025-10-01&end=2025-10-05`

**Response**

```json
{
  "hotelId": "hot_98765",
  "availableRooms": [
    {
      "roomTypeId": "rt_11",
      "type": "Deluxe Suite",
      "available": 3
    }
  ]
}
```

### Update Inventory (Owner)

`POST /room-inventory`

```json
{
  "roomTypeId": "rt_11",
  "date": "2025-10-01",
  "available": 5
}
```

### Get Room-Type Calendar

`GET /room-types/{roomTypeId}/calendar?month=2025-10`

---

### Bookings

**Create Booking**
`POST /bookings`

```json
{
  "hotelId": "hot_98765",
  "roomTypeId": "rt_11",
  "userId": "usr_555",
  "checkIn": "2025-10-02",
  "checkOut": "2025-10-05",
  "guests": 2
}
```

**Response**

```json
{
  "bookingId": "bk_20250922",
  "status": "CONFIRMED",
  "totalPrice": 660.00,
  "currency": "USD",
  "createdAt": "2025-09-22T17:10:00Z"
}
```

**Other Booking Endpoints**

* `GET /bookings/{bookingId}`
* `PUT /bookings/{bookingId}/cancel`
* `GET /users/{userId}/bookings`
* `GET /hotels/{hotelId}/bookings`

---

### Payments

* `POST /payments/stripe`
* `POST /payments/plaid`
* `GET /payments/{transactionId}`

**Sample Stripe Payment Request**

```json
{
  "bookingId": "bk_20250922",
  "token": "tok_visa"
}
```

**Response**

```json
{
  "transactionId": "txn_77777",
  "status": "SUCCESS",
  "amount": 660.00,
  "currency": "USD"
}
```

---

## 6.3 Review Service APIs

* `POST /reviews`
* `GET /hotels/{hotelId}/reviews`
* `GET /reviews/{reviewId}`
* `PUT /reviews/{reviewId}`
* `DELETE /reviews/{reviewId}`
* `GET /users/{userId}/reviews`
* `GET /hotels/{hotelId}/rating-summary`

**Create Review Example**

```json
{
  "hotelId": "hot_98765",
  "userId": "usr_555",
  "rating": 5,
  "comment": "Fantastic stay and friendly staff!"
}
```

---

## 6.4 Hotel Search Service APIs

* `GET /search/hotels?city=San%20Diego`
* `GET /search/hotels/{hotelId}/summary`
* `POST /search/filters`
* `GET /search/suggestions?query=Sun`

**Advanced Search Request**

```json
{
  "city": "San Diego",
  "priceRange": [100, 300],
  "minRating": 4
}
```

---

## 6.5 User Auth Service APIs

* `POST /auth/register`
* `POST /auth/login`
* `POST /auth/logout`
* `POST /auth/refresh`
* `POST /auth/forgot-password`
* `POST /auth/reset-password`

**Registration Request**

```json
{
  "email": "jane@example.com",
  "password": "StrongPass!23",
  "name": "Jane Doe"
}
```

**Response**

```json
{
  "userId": "usr_555",
  "email": "jane@example.com",
  "token": "jwt_token_here"
}
```

### User Profile

* `GET /users/profile`
* `PUT /users/profile`
* `GET /users/{userId}`  *(admin)*
* `PUT /users/{userId}/role`  *(admin)*

---

> **Note:** All timestamps are in ISO-8601 UTC format.
> Authentication for most endpoints is via Bearer JWT in the `Authorization` header.

```

Copy everything between the triple back-ticks into your `README.md` and GitHub will render it with syntax highlighting and code blocks ready to use.
```


## 7. Event & Messaging Design
```
Kafka Topics
# hotel-events
{
  "eventType": "HotelCreated|HotelUpdated",
  "hotelId": "12345",
  "ownerId": "67890",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": { "name": "Grand Palace", "city": "San Francisco" }
}

# room-availability
{
  "eventType": "RoomAvailabilityChanged",
  "hotelId": "12345",
  "roomTypeId": "67890",
  "date": "2025-02-01",
  "availableRooms": 15,
  "timestamp": "2025-01-15T10:30:00Z"
}

# review-events
{
  "eventType": "ReviewAdded",
  "reviewId": "98765",
  "hotelId": "12345",
  "userId": "54321",
  "rating": 5,
  "timestamp": "2025-01-15T10:30:00Z"
}
```

## 8. Caching Strategy
```
Redis Cache Keys
hotel:profile:{hotelId}            -> Hotel details JSON
hotel:owner:{ownerId}              -> List of hotel IDs
availability:{hotelId}:{date}      -> Available room counts by type
room:pricing:{roomTypeId}          -> Room pricing info
hotel:reviews:{hotelId}:summary    -> {avgRating, totalReviews}
hotel:reviews:{hotelId}:recent     -> Recent reviews list
session:{sessionId}                -> User session data
user:profile:{userId}              -> User profile info
```

## 9. External Integrations
```
Payment Processing: Stripe (credit card), Plaid (bank transfers)

Future Integrations:

Email: SendGrid for booking confirmations

SMS: Twilio for booking notifications

Maps: Google Maps for location services
```

## 10. Security Requirements
```
Authentication: JWT tokens with refresh mechanism

Authorization: Role-based access control (RBAC)

Data Encryption: TLS in transit, AES-256 at rest

Payment Security: PCI DSS compliance

Rate Limiting: API rate limits to prevent abuse

Input Validation: Sanitize all user inputs
```

