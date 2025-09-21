



 Here’s a **complete setup document** for your local development environment for a Yelp-like location-based service using **Docker Compose**. This includes **Docker setup**, **service configurations**, and **README instructions for mysql, elastic search and redis set up **.

---

# **Yelp-Clone Local Development Setup**

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
