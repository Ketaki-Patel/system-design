# Web Crawler — System Design Document

## 1. Overview

A simple web crawler that fetches web pages, extracts links, and stores the data for further analysis (e.g., indexing or data mining). 

---

## 2. High-Level Architecture

```mermaid
graph TD
    A[URL Frontier / Queue] --> B[Fetcher Service]
    B --> C[HTML Parser]
    C --> D[Data Store]
    C --> E[Link Extractor]
    E --> A
```

**Components**

* **URL Frontier / Queue**: Holds the list of URLs to crawl (e.g., Kafka, RabbitMQ, or simple in-memory queue).
* **Fetcher Service**: Sends HTTP requests and retrieves page content.
* **HTML Parser**: Extracts useful data (text, metadata).
* **Link Extractor**: Finds new links on each page and pushes them back to the URL Frontier.
* **Data Store**: Saves fetched pages or extracted metadata (could be a database or object storage).

---

## 3. Functional Requirements

* **Seed Input**: Accept a starting list of URLs.
* **Fetch Content**: Download HTML for each URL.
* **Extract Links**: Identify and enqueue new URLs from fetched pages.
* **Deduplication**: Avoid crawling the same URL repeatedly.
* **Politeness / Rate Limiting**: Respect robots.txt and add configurable delays between requests.
* **Data Storage**: Store raw HTML or key metadata for later use (e.g., indexing).
* **Monitoring**: Provide simple metrics (pages crawled, queue size).

---

## 4. Non-Functional Requirements

* **Scalability**: Must handle increasing URLs by adding more crawler workers.
* **Reliability**: Should resume from the queue after a crash without losing seed URLs.
* **Performance**: Crawl thousands of pages per hour with low latency per fetch.
* **Fault Tolerance**: Failures (network timeouts, invalid pages) are retried or skipped without halting the system.
* **Extensibility**: Easy to plug in new parsers or data sinks later (e.g., search index).
* **Resource Control**: Configurable concurrency limits to avoid overloading target sites.

---

## 5. Key Use Cases

* **Search Engine Indexing**: Gather and store page content to build a searchable index of websites.
* **Price Monitoring**: Regularly crawl e‑commerce sites to track product prices and availability.
* **Content Aggregation**: Collect blog posts or news articles for a recommendation or news‑feed app.
* **SEO Analysis**: Crawl a specific domain to check for broken links, page titles, or metadata quality.
* **Academic or Market Research**: Gather publicly available data sets such as job postings or product reviews.

---

## 6. Technology Choices (Simple Options)

* **Language**: Java with Spring Boot (to match team skillset).
* **Queue**: Kafka or RabbitMQ for distributed crawling.
* **Data Store**: PostgreSQL or Amazon S3 for raw data.
* **Deployment**: Docker containers or Kubernetes for horizontal scaling.

# **high-level walk-through** of the flow  step by step along with possible technology usage:

---

<img width="1041" height="624" alt="image" src="https://github.com/user-attachments/assets/89befb2f-9c33-4c88-9506-937f964205fc" />


### 1️⃣ URL Frontier / Queue

* Start with a **seed list of URLs** you want to crawl.
* These URLs are placed in a queue (for example, Kafka, RabbitMQ, or an in-memory queue).
* This queue is the *source of truth* for what needs to be fetched next.

### 2️⃣ Fetcher Service

* Worker processes (or crawler nodes) pull URLs from the queue.
* Each worker sends an **HTTP request** to the target website and downloads the raw HTML content.
* If a page is unreachable, the worker logs the error and either retries or skips it.

### 3️⃣ HTML Parser

* The downloaded HTML is passed to a parser that cleans and structures the content.
* It extracts any information we care about (title, body text, metadata).

### 4️⃣ Link Extractor

* While parsing, the crawler also **collects all hyperlinks** on the page.
* These new links are checked for duplicates and robots.txt rules.
* Valid new URLs are **pushed back to the URL Frontier**, so the queue never runs dry.

### 5️⃣ Data Store

* The parsed content and key metadata are saved in a **Data Store** (database, S3, or other storage).
* This provides a permanent record for search indexing, analytics, or other downstream uses.

---

# **End-to-End Summary:**
The crawler begins with a small list of seed URLs, then repeatedly:

1. **Takes a URL from the queue → fetches the page → parses it → extracts new links → stores data → re-enqueues new URLs.**

This loop continues until there are no new URLs or until a configured limit is reached.
Because each stage (queue, fetcher, parser, storage) is separate, you can scale horizontally by adding more fetcher or parser workers without changing the overall design.

Absolutely! Let me explain the main data flow in simple terms with the numbered references:

## Web Crawler Data Flow - Simple Explanation

**1. Seed URLs → URL Frontier (Kafka)**
- You start by giving the system a list of websites you want to crawl (like "crawl google.com, facebook.com")
- These starting URLs get put into a waiting line (queue) managed by Kafka

**2. URL Frontier → Fetcher Service** 
- The Fetcher Service is like a worker that says "give me the next website to visit"
- It picks up one URL from the waiting line

**3. Fetcher Service → External Websites**
- The Fetcher Service visits that website (sends HTTP request)
- Like opening a web browser and going to the website

**4. External Websites → Fetcher Service**
- The website sends back its HTML page content
- Like the webpage loading in your browser

**5. Fetcher Service → HTML Parser Service**
- The raw HTML code gets sent to the HTML Parser
- The parser's job is to understand what's on the page

**6. HTML Parser Service → Data Store**
- The useful information (text, titles, images) gets saved in the database
- This is like bookmarking and saving the important parts of the webpage

**7. HTML Parser Service → Link Extractor Service**
- The HTML Parser also sends the page content to find all the links on that page
- Like finding all the clickable links on a webpage

**8. Link Extractor Service → URL Frontier (Kafka)**
- All the new links found get added back to the waiting line
- Now the system has more websites to visit, and the cycle repeats

**The Loop:** Steps 2-8 keep repeating until there are no more URLs to crawl or you stop the system.

## Details of other compoment

Let me explain each component in detail:

## **Fetcher Service Components:**

### **1. HTTP Client**
- **Purpose**: Makes actual HTTP requests to websites
- **Function**: Handles GET/POST requests, manages timeouts, retries failed requests
- **Why needed**: Core component that physically downloads web pages

### **2. Rate Limiter** 
- **Purpose**: Controls how fast we crawl websites
- **Function**: Ensures we don't send too many requests per second to same domain
- **Why needed**: Prevents overwhelming target servers (politeness), avoids getting blocked/banned

### **3. robots.txt Parser**
- **Purpose**: Respects website crawling rules
- **Function**: Reads robots.txt files to check what URLs we're allowed to crawl
- **Why needed**: Legal/ethical compliance - respects website owner's wishes

### **4. URL Deduplication (Bloom Filter)**
- **Purpose**: Prevents crawling same URL multiple times
- **Function**: Keeps track of already-visited URLs using memory-efficient Bloom Filter
- **Why needed**: Saves bandwidth, storage, and prevents infinite loops

---

## **Data Store Components:**

### **1. Raw HTML Storage (S3)**
- **What it stores**: Complete HTML source code of web pages
- **Why S3**: 
  - **Cheap storage** for large amounts of data
  - **Scalable** - can store petabytes
  - **Durable** - won't lose data
  - **Cold storage** - rarely accessed but kept for backup/reprocessing

### **2. Metadata DB (PostgreSQL)**
- **What metadata we store**:
  - **URL, title, description** of each page
  - **Crawl timestamp** (when was it crawled)
  - **Content length, response code** (200, 404, etc.)
  - **Domain information** (which website it came from)
  - **Link count** (how many links found on page)
  - **Content type** (HTML, PDF, etc.)
- **Why PostgreSQL**: Fast queries, relationships between data, ACID compliance

### **3. Cache Layer (Redis)**
- **What it caches**:
  - **Recently crawled URLs** (fast duplicate checking)
  - **robots.txt files** (avoid re-downloading same rules)
  - **DNS lookups** (website IP addresses)
  - **Popular page metadata** (frequently accessed data)
- **Why needed**: 
  - **Speed** - RAM is 1000x faster than disk
  - **Reduces database load** - fewer queries to PostgreSQL
  - **Quick duplicate detection** - instant URL checking

## **Why This Architecture?**

- **S3** = Long-term, cheap storage for raw HTML (like a warehouse)
- **PostgreSQL** = Fast, structured queries for metadata (like a filing system)  
- **Redis** = Lightning-fast temporary storage (like your desk drawer for frequently used items)





