Let’s create a **concise Elasticsearch feature refresh ** that covers everything from **creating an index**, **settings & mappings**, to **analyzers and full-text search features** like stemming, stop words, fuzzy search, and wildcards. I’ll explain it step by step with examples you can try locally.

---

# 1. **Elasticsearch Primer: Full-Text & Location Search**

## **1. Elasticsearch Basics**

* **Index** → Like a database table
* **Document** → Like a row in a table
* **Field** → Like a column
* **Mapping** → Schema for the index, defining types and analyzers
* **Analyzer** → Text processor that converts raw text into searchable tokens

---

## **2. Creating an Index with Settings and Mappings**

### **Step 1: Create Index**

```http
PUT /places
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "rating": { "type": "float" }
    }
  }
}
```

* `text` → Full-text searchable
* `keyword` → Exact match / filters
* `geo_point` → Lat/Lon for location search

---

### **Step 2: Add Settings with Custom Analyzer**

```http
PUT /places
{
  "settings": {
    "analysis": {
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["cafe, coffee shop", "restaurant, diner"]
        }
      },
      "analyzer": {
        "custom_text_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stop", "english_stemmer", "synonym_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "custom_text_analyzer" },
      "category": { "type": "keyword" },
      "location": { "type": "geo_point" }
    }
  }
}
```

**Explanation:**

* **Lowercase** → Case-insensitive search
* **Stop words** → Removes common words like “the, a, an”
* **Stemming** → Reduces words to root (`restaurants` → `restaurant`)
* **Synonyms** → “cafe” matches “coffee shop”

---

## **3. Inserting Documents**

```http
POST /places/_doc
{
  "name": "Central Cafe",
  "category": "cafe",
  "location": { "lat": 37.7749, "lon": -122.4194 },
  "rating": 4.5
}

POST /places/_doc
{
  "name": "Golden Gate Pizza",
  "category": "restaurant",
  "location": { "lat": 37.7750, "lon": -122.4183 },
  "rating": 4.7
}
```

---

## **4. Full-Text Search Queries**

### **Match Query (Stemming & Stop Words Applied)**

```http
GET /places/_search
{
  "query": {
    "match": {
      "name": "central cafes"
    }
  }
}
```

* Matches `Central Cafe` due to **stemming** (“cafes” → “cafe”)

---

### **Fuzzy Search (Typo-Tolerant)**

```http
GET /places/_search
{
  "query": {
    "match": {
      "name": {
        "query": "centrall cafe",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

* Corrects small typos automatically

---

### **Wildcard Search**

```http
GET /places/_search
{
  "query": {
    "wildcard": {
      "name": {
        "value": "cent*"
      }
    }
  }
}
```

* Matches names starting with `cent` (`Central Cafe`)

---

### **Geo Search**

```http
GET /places/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": { "lat": 37.7749, "lon": -122.4194 }
        }
      }
    }
  }
}
```

* Finds places **within 5 km** of the given coordinates

---

### **Combining Text + Geo**

```http
GET /places/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { "name": { "query": "coffee shop", "fuzziness": "AUTO" } }
      },
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": { "lat": 37.7749, "lon": -122.4194 }
        }
      }
    }
  }
}
```

---

## **5. Key Concepts**

| Feature           | Where Configured     | Description           |
| ----------------- | -------------------- | --------------------- |
| Stemming          | Analyzer / Index     | Reduces words to root |
| Stop Words        | Analyzer / Index     | Removes common words  |
| Synonyms          | Analyzer / Index     | Map similar words     |
| Fuzzy Search      | Query-Time           | Handles typos         |
| Wildcard / Prefix | Query-Time           | Matches partial words |
| Geo Distance      | Query-Time           | Filter by location    |
| Autocomplete      | Mapping + Query-Time | `completion` type     |

---

## **6. Tips for Spring Boot Integration**

1. Use `spring-boot-starter-data-elasticsearch`
2. Define **ElasticsearchRepositories** for indexes
3. Use **ElasticsearchOperations** for complex queries
4. Keep **MySQL** for transactional data, **Elasticsearch** for search

---

If you want, I can make a **full ready-to-use Elasticsearch index JSON** for your Yelp-like service including:

* Stemming
* Stop words
* Synonyms
* Geo points
* Completion field for autocomplete

So you can just apply it locally and start testing search.

Do you want me to do that next?

Here’s a **ready-to-use Elasticsearch index setup** for your **Yelp-like location-based service**. It includes **geo-points, stemming, stop words, synonyms, fuzzy search readiness, and autocomplete**. You can just copy-paste this into Kibana Dev Tools or use `curl` locally.

---

# 2. **Yelp-Clone(yelp like app) Local Development Setup**

## **1. Project Folder Structure**

```
yelp-clone/
│
├─ docker-compose.yml
├─ README.md
└─ (future folders: backend, frontend, scripts)
```

---

## **2. Docker Compose Configuration**

Create `docker-compose.yml` in the project root:

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: yelp_clone
      MYSQL_USER: appuser
      MYSQL_PASSWORD: password123
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  redis:
    image: redis:7
    container_name: redis
    ports:
      - "6379:6379"

volumes:
  mysql-data:
  es-data:
```

**Notes:**

* **Volumes** persist data between container restarts.
* **Port mappings** allow you to connect from your local machine.

---

## **3. README.md Instructions**

Create a `README.md` in the project root:

````markdown
# Yelp-Clone Local Development Setup

This project sets up a local development environment for a Yelp-like location-based service using Docker Compose. Services included:

- **MySQL** for transactional data
- **Elasticsearch** for geo & full-text search
- **Redis** for caching

---

## Prerequisites

- [Docker](https://www.docker.com/get-started) installed
- [Docker Compose](https://docs.docker.com/compose/install/)
- Optional: Postman or MySQL Workbench for testing APIs & DB

---

## 1. Start Services

From the project root where `docker-compose.yml` is located:

```bash
docker-compose up
````

* To run in background:

```bash
docker-compose up -d
```

---

## 2. Verify Services

```bash
docker ps
```

You should see containers:

* `mysql` on port `3306`
* `elasticsearch` on port `9200`
* `redis` on port `6379`

---

## 3. Connect to Services

### MySQL

```bash
mysql -h 127.0.0.1 -P 3306 -u appuser -p
```

Password: `password123`

### Elasticsearch

```bash
curl http://localhost:9200
```

Should return JSON cluster info.

### Redis

```bash
redis-cli ping
```

Should return: `PONG`

---

## 4. Stop Services

```bash
docker-compose down
```

* Use `-v` to remove volumes (data):

```bash
docker-compose down -v
```

---

## 5. Notes

* All services are configured for **local development only**.
* Elasticsearch is set to **single-node** for simplicity.
* Use the provided credentials when connecting from Spring Boot.

```

---

## ✅ **Next Steps After Setup**

1. Create **MySQL tables** for users, places, and reviews.  
2. Create **Elasticsearch index** with geo_point and analyzers.  
3. Connect **Spring Boot** backend to MySQL, Elasticsearch, and Redis.  
4. Implement **search endpoints** with geo + fuzzy + stemming.  

---

If you want, I can **write the MySQL + Elasticsearch schema setup** and provide **Spring Boot integration code** next, so your local environment is fully ready for development.  

Do you want me to do that next?
```

Here’s an **expanded MySQL schema** for a Yelp-like project, now with **two optional tables** plus **seed data (3 rows each)** so you can spin it up and test immediately.

---

# 3. **Expanded MySQL schema**
- 1️⃣ Core Tables

### `users`

```sql
CREATE TABLE users (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    username       VARCHAR(50)  NOT NULL UNIQUE,
    email          VARCHAR(100) NOT NULL UNIQUE,
    password_hash  VARCHAR(255) NOT NULL,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### `places`

```sql
CREATE TABLE places (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(150) NOT NULL,
    category     VARCHAR(50)  NOT NULL,
    address      VARCHAR(255),
    city         VARCHAR(100),
    state        VARCHAR(100),
    country      VARCHAR(100),
    latitude     DECIMAL(10,7) NOT NULL,
    longitude    DECIMAL(10,7) NOT NULL,
    avg_rating   DECIMAL(2,1)  DEFAULT 0.0,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### `reviews`

```sql
CREATE TABLE reviews (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    place_id   BIGINT NOT NULL,
    rating     TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment    TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id)  REFERENCES users(id)  ON DELETE CASCADE,
    FOREIGN KEY (place_id) REFERENCES places(id) ON DELETE CASCADE
);
```

---

## 2️⃣ Optional Supporting Tables

### `categories`

Normalized list of categories to avoid duplicates in `places.category`.

```sql
CREATE TABLE categories (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    name      VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### `place_photos`

Holds URLs of images for each place.

```sql
CREATE TABLE place_photos (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    place_id  BIGINT NOT NULL,
    photo_url VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (place_id) REFERENCES places(id) ON DELETE CASCADE
);
```

---

## 3️⃣ Seed Data (Insert 3 Rows per Table)

### Users

```sql
INSERT INTO users (username, email, password_hash) VALUES
('alice', 'alice@example.com', 'hash_alice'),
('bob',   'bob@example.com',   'hash_bob'),
('carol', 'carol@example.com', 'hash_carol');
```

### Categories

```sql
INSERT INTO categories (name) VALUES
('Cafe'),
('Restaurant'),
('Bakery');
```

### Places

```sql
INSERT INTO places (name, category, address, city, state, country, latitude, longitude, avg_rating)
VALUES
('Central Cafe',  'Cafe',  '123 Market St',   'San Francisco', 'CA', 'USA', 37.7749, -122.4194, 4.5),
('Golden Gate Pizza','Restaurant','456 Bay St','San Francisco', 'CA', 'USA', 37.8050, -122.4200, 4.7),
('Sweet Tooth Bakery','Bakery','789 Mission St','San Francisco','CA','USA',37.7830, -122.4090, 4.3);
```

### Reviews

```sql
INSERT INTO reviews (user_id, place_id, rating, comment) VALUES
(1, 1, 5, 'Amazing coffee and atmosphere!'),
(2, 2, 4, 'Great pizza, slightly long wait times.'),
(3, 3, 5, 'Delicious pastries and friendly staff.');
```

### Place Photos

```sql
INSERT INTO place_photos (place_id, photo_url) VALUES
(1, 'https://example.com/photos/central-cafe-1.jpg'),
(2, 'https://example.com/photos/golden-gate-pizza-1.jpg'),
(3, 'https://example.com/photos/sweet-tooth-bakery-1.jpg');
```

---

### ✅ How to Use

1. Save all the `CREATE TABLE` statements into a file (e.g., `schema.sql`) and run:

   ```bash
   mysql -u root -p yelp_clone < schema.sql
   ```
2. Save the `INSERT` statements into `seed.sql` and run:

   ```bash
   mysql -u root -p yelp_clone < seed.sql
   ```

This gives you a **fully populated local MySQL database** with:

* **Users** ready for authentication
* **Places** linked to **categories**, **photos**, and **reviews**
* Clean relationships for syncing to Elasticsearch.

# 4. **Elasticsearch Index: `places`**

```json
PUT /places
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "cafe, coffee shop",
            "restaurant, diner",
            "pizza, pizzeria"
          ]
        }
      },
      "analyzer": {
        "custom_text_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "english_stop",
            "english_stemmer",
            "synonym_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "name": { "type": "text", "analyzer": "custom_text_analyzer" },
      "category": { "type": "keyword" },
      "description": { "type": "text", "analyzer": "custom_text_analyzer" },
      "location": { "type": "geo_point" },
      "rating": { "type": "float" },
      "suggest": { "type": "completion" }
    }
  }
}
```

---

## **Explanation of Fields**

| Field         | Type       | Purpose                                                    |
| ------------- | ---------- | ---------------------------------------------------------- |
| `id`          | long       | Unique place ID                                            |
| `name`        | text       | Full-text searchable (with stemming, stop words, synonyms) |
| `category`    | keyword    | Exact match filtering (e.g., cafe, restaurant)             |
| `description` | text       | Searchable description field                               |
| `location`    | geo\_point | Latitude/Longitude for geo queries                         |
| `rating`      | float      | Can be used for sorting or filtering                       |
| `suggest`     | completion | Autocomplete suggestions for frontend                      |

---

## **Example: Adding a Place**

```json
POST /places/_doc/1
{
  "id": 1,
  "name": "Central Cafe",
  "category": "cafe",
  "description": "A cozy coffee shop with great espresso and pastries",
  "location": { "lat": 37.7749, "lon": -122.4194 },
  "rating": 4.5,
  "suggest": { "input": ["Central Cafe", "Cafe Central"] }
}
```

---

## **Example: Fuzzy Search**

```json
GET /places/_search
{
  "query": {
    "match": {
      "name": {
        "query": "centrall cafe",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

* Will match `"Central Cafe"` even with typos.

---

## **Example: Geo + Text Search**

```json
GET /places/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { "description": { "query": "coffee shop", "fuzziness": "AUTO" } }
      },
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": { "lat": 37.7749, "lon": -122.4194 }
        }
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 37.7749, "lon": -122.4194 },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

---

## **Example: Autocomplete / Suggest**

```json
POST /places/_search
{
  "suggest": {
    "place-suggest": {
      "prefix": "cen",
      "completion": {
        "field": "suggest",
        "skip_duplicates": true,
        "size": 5
      }
    }
  }
}
```

* Typing `"cen"` will suggest `"Central Cafe"`
* Works well for frontend search bars

---

✅ **This index covers everything you need for a Yelp-like service:**

* Full-text search with **stemming & stop words**
* **Synonym handling**
* **Fuzzy search** for typos
* **Wildcard/prefix search** capability
* **Geo-point search** for nearby places
* **Autocomplete / suggest** for search bars

---


 Here’s a **complete setup document** for your local development environment for a Yelp-like location-based service using **Docker Compose**. This includes **Docker setup**, **service configurations**, and **README instructions for mysql, elastic search and redis set up **.

--- 

# 4. Java, Spring boot Location Search Project

# Yelp Clone - Spring Boot Project

This repository contains a minimal, opinionated Spring Boot project for a Yelp-like location search service using:
- Java 17
- Spring Boot 3.x
- Spring Data JPA (Hibernate) + MySQL
- Spring Data Elasticsearch (documents for places)
- Spring Data Redis (caching)
- Flyway (DB migrations)

---

## Project structure (single-file snapshot for reference)

```
yelp-clone/
├── pom.xml
├── src/main/java/com/example/yelp
│   ├── YelpCloneApplication.java
│   ├── config
│   │   ├── ElasticsearchConfig.java
│   │   └── RedisConfig.java
│   ├── controller
│   │   └── PlaceController.java
│   ├── dto
│   │   ├── PlaceRequest.java
│   │   └── PlaceResponse.java
│   ├── entity
│   │   ├── User.java
│   │   ├── Place.java
│   │   ├── Review.java
│   │   ├── Category.java
│   │   └── PlacePhoto.java
│   ├── repository
│   │   ├── UserRepository.java
│   │   ├── PlaceRepository.java
│   │   ├── ReviewRepository.java
│   │   ├── CategoryRepository.java
│   │   └── PlacePhotoRepository.java
│   ├── search
│   │   ├── PlaceDocument.java
│   │   ├── PlaceSearchService.java
│   │   └── ElasticsearchIndexInitializer.java
│   └── service
│       └── PlaceService.java
├── src/main/resources
│   ├── application.yml
│   └── db/migration/V1__init_schema_and_seed.sql
└── README.md
```

---

## pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>yelp-clone</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <properties>
        <java.version>17</java.version>
        <spring.boot.version>3.2.0</spring.boot.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- JPA / MySQL -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!-- Elasticsearch -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>

        <!-- Redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- Flyway -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- Validation & Lombok (optional but convenient) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## application.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/yelp_clone?useSSL=false&allowPublicKeyRetrieval=true
    username: appuser
    password: password123
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
  redis:
    host: localhost
    port: 6379
  elasticsearch:
    uris: http://localhost:9200

# Flyway
flyway:
  enabled: true
  locations: classpath:db/migration

server:
  port: 8080

logging:
  level:
    org.springframework: INFO
```

---

## Flyway Migration: src/main/resources/db/migration/V1__init_schema_and_seed.sql

```sql
-- Users
CREATE TABLE IF NOT EXISTS users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL UNIQUE,
  email VARCHAR(100) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Categories
CREATE TABLE IF NOT EXISTS categories (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Places
CREATE TABLE IF NOT EXISTS places (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(150) NOT NULL,
  category VARCHAR(50) NOT NULL,
  address VARCHAR(255),
  city VARCHAR(100),
  state VARCHAR(100),
  country VARCHAR(100),
  latitude DECIMAL(10,7) NOT NULL,
  longitude DECIMAL(10,7) NOT NULL,
  avg_rating DECIMAL(2,1) DEFAULT 0.0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Reviews
CREATE TABLE IF NOT EXISTS reviews (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  place_id BIGINT NOT NULL,
  rating TINYINT NOT NULL,
  comment TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_review_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT fk_review_place FOREIGN KEY (place_id) REFERENCES places(id) ON DELETE CASCADE
);

-- Place photos
CREATE TABLE IF NOT EXISTS place_photos (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  place_id BIGINT NOT NULL,
  photo_url VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_photo_place FOREIGN KEY (place_id) REFERENCES places(id) ON DELETE CASCADE
);

-- Seed users
INSERT INTO users (username, email, password_hash) VALUES
('alice', 'alice@example.com', 'hash_alice'),
('bob', 'bob@example.com', 'hash_bob'),
('carol', 'carol@example.com', 'hash_carol');

-- Seed categories
INSERT INTO categories (name) VALUES
('Cafe'), ('Restaurant'), ('Bakery');

-- Seed places
INSERT INTO places (name, category, address, city, state, country, latitude, longitude, avg_rating) VALUES
('Central Cafe', 'Cafe', '123 Market St', 'San Francisco', 'CA', 'USA', 37.7749000, -122.4194000, 4.5),
('Golden Gate Pizza', 'Restaurant','456 Bay St','San Francisco','CA','USA',37.8050000,-122.4200000,4.7),
('Sweet Tooth Bakery','Bakery','789 Mission St','San Francisco','CA','USA',37.7830000,-122.4090000,4.3);

-- Seed reviews
INSERT INTO reviews (user_id, place_id, rating, comment) VALUES
(1,1,5,'Amazing coffee and atmosphere!'),
(2,2,4,'Great pizza, slightly long wait times.'),
(3,3,5,'Delicious pastries and friendly staff.');

-- Seed photos
INSERT INTO place_photos (place_id, photo_url) VALUES
(1,'https://example.com/photos/central-cafe-1.jpg'),
(2,'https://example.com/photos/golden-gate-pizza-1.jpg'),
(3,'https://example.com/photos/sweet-tooth-bakery-1.jpg');
```

---

## Entities (JPA)

### Place.java
```java
package com.example.yelp.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "places")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Place {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String category;
    private String address;
    private String city;
    private String state;
    private String country;

    private Double latitude;
    private Double longitude;

    private Double avgRating;

    private Instant createdAt;
    private Instant updatedAt;
}
```

### User.java
```java
package com.example.yelp.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;
    private String passwordHash;
}
```

### Review.java
```java
package com.example.yelp.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "reviews")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Review {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;
    private Long placeId;
    private Integer rating;
    private String comment;
    private Instant createdAt;
}
```

### PlacePhoto.java
```java
package com.example.yelp.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "place_photos")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PlacePhoto {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long placeId;
    private String photoUrl;
}
```

### Category.java
```java
package com.example.yelp.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "categories")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
}
```

---

## Repositories

### PlaceRepository.java
```java
package com.example.yelp.repository;

import com.example.yelp.entity.Place;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface PlaceRepository extends JpaRepository<Place, Long> {
}
```

### UserRepository.java
```java
package com.example.yelp.repository;

import com.example.yelp.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

### ReviewRepository.java
```java
package com.example.yelp.repository;

import com.example.yelp.entity.Review;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ReviewRepository extends JpaRepository<Review, Long> {
}
```

---

## Elasticsearch - Document & Search Service

### PlaceDocument.java
```java
package com.example.yelp.search;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.GeoPointField;
import lombok.*;

@Document(indexName = "places")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PlaceDocument {
    @Id
    private Long id;
    private String name;
    private String category;
    private String description;
    @GeoPointField
    private String location; // "lat,lon" or object can be handled at conversion
    private Double rating;
}
```

### ElasticsearchIndexInitializer.java
```java
package com.example.yelp.search;

import jakarta.annotation.PostConstruct;
import org.springframework.stereotype.Component;
import org.springframework.data.elasticsearch.client.elc.ElasticsearchTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.index.PutIndexTemplateRequest;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.data.elasticsearch.core.IndexOperations;
import org.springframework.data.elasticsearch.core.document.Document;

@Component
public class ElasticsearchIndexInitializer {
    private final ElasticsearchOperations operations;

    public ElasticsearchIndexInitializer(ElasticsearchOperations operations) {
        this.operations = operations;
    }

    @PostConstruct
    public void init() {
        IndexOperations indexOps = operations.indexOps(IndexCoordinates.of("places"));
        if (!indexOps.exists()) {
            String mapping = "{\n" +
                    "  \"settings\": {\n" +
                    "    \"number_of_shards\": 1,\n" +
                    "    \"number_of_replicas\": 0,\n" +
                    "    \"analysis\": {\n" +
                    "      \"filter\": {\n" +
                    "        \"english_stop\": { \"type\": \"stop\", \"stopwords\": \"_english_\" },\n" +
                    "        \"english_stemmer\": { \"type\": \"stemmer\", \"language\": \"english\" },\n" +
                    "        \"synonym_filter\": { \"type\": \"synonym\", \"synonyms\": [\"cafe, coffee shop\", \"restaurant, diner\"] }\n" +
                    "      },\n" +
                    "      \"analyzer\": {\n" +
                    "        \"custom_text_analyzer\": {\n" +
                    "          \"type\": \"custom\",\n" +
                    "          \"tokenizer\": \"standard\",\n" +
                    "          \"filter\": [\"lowercase\", \"english_stop\", \"english_stemmer\", \"synonym_filter\"]\n" +
                    "        }\n" +
                    "      }\n" +
                    "    }\n" +
                    "  },\n" +
                    "  \"mappings\": {\n" +
                    "    \"properties\": {\n" +
                    "      \"id\": { \"type\": \"long\" },\n" +
                    "      \"name\": { \"type\": \"text\", \"analyzer\": \"custom_text_analyzer\" },\n" +
                    "      \"category\": { \"type\": \"keyword\" },\n" +
                    "      \"description\": { \"type\": \"text\", \"analyzer\": \"custom_text_analyzer\" },\n" +
                    "      \"location\": { \"type\": \"geo_point\" },\n" +
                    "      \"rating\": { \"type\": \"float\" }\n" +
                    "    }\n" +
                    "  }\n" +
                    "}";
            Document d = Document.parse(mapping);
            indexOps.create(d);
        }
    }
}
```

### PlaceSearchService.java
```java
package com.example.yelp.search;

import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.sort.SortBuilders;
import org.elasticsearch.search.sort.SortOrder;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.NativeSearchQueryBuilder;
import org.springframework.data.elasticsearch.core.query.NativeSearchQuery;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class PlaceSearchService {
    private final ElasticsearchOperations operations;

    public PlaceSearchService(ElasticsearchOperations operations) {
        this.operations = operations;
    }

    public List<PlaceDocument> findNearbyAndMatch(String text, double lat, double lon, double distanceKm) {
        var qb = QueryBuilders.boolQuery()
                .must(QueryBuilders.matchQuery("name", text).fuzziness("AUTO"))
                .filter(QueryBuilders.geoDistanceQuery("location").point(lat, lon).distance(distanceKm + "km"));

        NativeSearchQuery q = new NativeSearchQueryBuilder()
                .withQuery(qb)
                .withSort(SortBuilders.geoDistanceSort("location", lat, lon).order(SortOrder.ASC))
                .build();

        SearchHits<PlaceDocument> hits = operations.search(q, PlaceDocument.class, IndexCoordinates.of("places"));
        return hits.stream().map(h -> h.getContent()).toList();
    }
}
```

---

## Service to sync MySQL -> Elasticsearch (on-demand simplified)

### PlaceService.java
```java
package com.example.yelp.service;

import com.example.yelp.entity.Place;
import com.example.yelp.repository.PlaceRepository;
import com.example.yelp.search.PlaceDocument;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.stream.Collectors;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.IndexOperations;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;

@Service
public class PlaceService {
    private final PlaceRepository placeRepository;
    private final ElasticsearchOperations operations;

    public PlaceService(PlaceRepository placeRepository, ElasticsearchOperations operations) {
        this.placeRepository = placeRepository;
        this.operations = operations;
    }

    public List<Place> listAll() {
        return placeRepository.findAll();
    }

    @Transactional
    public Place create(Place p) {
        Place saved = placeRepository.save(p);
        // Index into ES
        PlaceDocument doc = PlaceDocument.builder()
                .id(saved.getId())
                .name(saved.getName())
                .category(saved.getCategory())
                .description(saved.getAddress())
                .location(saved.getLatitude() + "," + saved.getLongitude())
                .rating(saved.getAvgRating())
                .build();
        operations.save(doc, IndexCoordinates.of("places"));
        return saved;
    }

    public void reindexAll() {
        List<PlaceDocument> docs = placeRepository.findAll().stream().map(p -> PlaceDocument.builder()
                .id(p.getId())
                .name(p.getName())
                .category(p.getCategory())
                .description(p.getAddress())
                .location(p.getLatitude() + "," + p.getLongitude())
                .rating(p.getAvgRating())
                .build()).collect(Collectors.toList());

        docs.forEach(d -> operations.save(d, IndexCoordinates.of("places")));
    }
}
```

---

## Controller & DTOs

### PlaceRequest.java
```java
package com.example.yelp.dto;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class PlaceRequest {
    private String name;
    private String category;
    private String address;
    private String city;
    private String state;
    private String country;
    private Double latitude;
    private Double longitude;
}
```

### PlaceResponse.java
```java
package com.example.yelp.dto;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PlaceResponse {
    private Long id;
    private String name;
    private String category;
    private String address;
    private Double latitude;
    private Double longitude;
    private Double avgRating;
}
```

### PlaceController.java
```java
package com.example.yelp.controller;

import com.example.yelp.dto.PlaceRequest;
import com.example.yelp.dto.PlaceResponse;
import com.example.yelp.entity.Place;
import com.example.yelp.service.PlaceService;
import com.example.yelp.search.PlaceSearchService;
import com.example.yelp.search.PlaceDocument;
import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/places")
public class PlaceController {
    private final PlaceService placeService;
    private final PlaceSearchService searchService;

    public PlaceController(PlaceService placeService, PlaceSearchService searchService) {
        this.placeService = placeService;
        this.searchService = searchService;
    }

    @PostMapping
    public PlaceResponse createPlace(@RequestBody PlaceRequest req) {
        Place p = Place.builder()
                .name(req.getName())
                .category(req.getCategory())
                .address(req.getAddress())
                .city(req.getCity())
                .state(req.getState())
                .country(req.getCountry())
                .latitude(req.getLatitude())
                .longitude(req.getLongitude())
                .avgRating(0.0)
                .build();
        Place saved = placeService.create(p);
        return PlaceResponse.builder()
                .id(saved.getId())
                .name(saved.getName())
                .category(saved.getCategory())
                .address(saved.getAddress())
                .latitude(saved.getLatitude())
                .longitude(saved.getLongitude())
                .avgRating(saved.getAvgRating())
                .build();
    }

    @GetMapping("/nearby")
    public List<PlaceDocument> searchNearby(@RequestParam String q,
                                            @RequestParam double lat,
                                            @RequestParam double lon,
                                            @RequestParam(defaultValue = "5") double km) {
        return searchService.findNearbyAndMatch(q, lat, lon, km);
    }

    @GetMapping
    public List<PlaceResponse> listAll() {
        return placeService.listAll().stream().map(p -> PlaceResponse.builder()
                .id(p.getId())
                .name(p.getName())
                .category(p.getCategory())
                .address(p.getAddress())
                .latitude(p.getLatitude())
                .longitude(p.getLongitude())
                .avgRating(p.getAvgRating()).build()).collect(Collectors.toList());
    }
}
```

---

## Elasticsearch & Redis config

### ElasticsearchConfig.java
```java
package com.example.yelp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.repository.config.EnableElasticsearchRepositories;

@Configuration
@EnableElasticsearchRepositories(basePackages = "com.example.yelp.search")
public class ElasticsearchConfig {
    // Spring Boot auto-configuration is typically enough when spring.elasticsearch.uris is set
}
```

### RedisConfig.java
```java
package com.example.yelp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.context.annotation.Bean;

@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        return template;
    }
}
```

---

## Application bootstrap

### YelpCloneApplication.java
```java
package com.example.yelp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class YelpCloneApplication {
    public static void main(String[] args) {
        SpringApplication.run(YelpCloneApplication.class, args);
    }
}
```

---

## API Examples

### 1) Create Place (HTTP)
**Request**
```
POST /api/places
Content-Type: application/json
{
  "name": "New Cafe",
  "category": "Cafe",
  "address": "100 New St",
  "city": "San Francisco",
  "state": "CA",
  "country": "USA",
  "latitude": 37.7749,
  "longitude": -122.4194
}
```

**Response**
```
200 OK
{
  "id": 10,
  "name": "New Cafe",
  "category": "Cafe",
  "address": "100 New St",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "avgRating": 0.0
}
```

### 2) Search Nearby (HTTP)
**Request**
```
GET /api/places/nearby?q=coffee&lat=37.7749&lon=-122.4194&km=5
```

**Response** (example)
```
200 OK
[
  { "id":1, "name":"Central Cafe", "category":"Cafe", "description":"...", "location":"37.7749,-122.4194", "rating":4.5 }
]
```

---

## Notes & Next Steps
- This snapshot is intentionally concise. In a real project you'd add:
  - DTO validation, error handling (ControllerAdvice)
  - Authentication & Authorization (Spring Security / JWT)
  - More robust mapping between JPA entities and ES documents
  - Async sync (Kafka) for scale
  - Unit & integration tests
  - Generate the full multi-file Maven project as a downloadable zip.

---













