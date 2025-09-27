 **RecoMind system design** for a YouTube-style short video recommendation system 

---

````markdown
# 📘 RecoMind: YouTube-Style Short Video Recommendation System

---

## 🎯 1. Purpose of the System

We are building a **recommendation system** for a **short video platform**, similar to YouTube Shorts or TikTok.

RecoMind will:

- Recommend short videos to users based on their activity
- Track user behavior (views, likes, skips)
- Show personalized recommendations
- Show trending videos to new or logged-out users

---

## ✅ 2. Functional Requirements

### 🎥 Core User Features:

- Watch short videos
- Like or skip a video
- View personalized video recommendations
- View trending videos when not logged in

### 🧠 System Behavior:

- Collect user interaction data (likes, views, skips)
- Build user profiles based on behavior
- Recommend Top N short videos per user

---

## 💡 3. How the Recommendation Engine Works

### 🧱 Strategies Used

| Strategy              | Description                                                                 | Used in RecoMind |
|----------------------|-----------------------------------------------------------------------------|------------------|
| Content-based        | Recommends videos similar to what the user already liked                    | ✅ Yes           |
| Collaborative        | Recommends videos liked by users with similar tastes                        | ✅ Yes           |
| Hybrid               | Combines both content-based and collaborative approaches                   | ✅ Core approach |
| Popularity-based     | Recommends trending content for new users                                   | ✅ Yes           |

### 👥 Example

- User A liked videos tagged with `#funny`, `#meme`.
- Content-based: suggests other `#funny`, `#meme` videos.
- Collaborative: finds that other users who liked the same also liked `#dance` videos.
- Final recommendation is a mix of all.

---

## 🗃️ 4. Database Design

We will use **PostgreSQL** for core storage and **Redis** for caching.

### 📄 Tables

---

#### 1. `users`

| Column        | Type     |
|---------------|----------|
| `user_id`     | UUID     |
| `name`        | TEXT     |
| `created_at`  | TIMESTAMP |

---

#### 2. `videos`

| Column        | Type     |
|---------------|----------|
| `video_id`    | UUID     |
| `title`       | TEXT     |
| `url`         | TEXT     |
| `tags`        | TEXT[]   |
| `creator_id`  | UUID     |
| `created_at`  | TIMESTAMP |

---

#### 3. `video_actions`

| Column        | Type                         |
|---------------|------------------------------|
| `id`          | UUID                         |
| `user_id`     | UUID                         |
| `video_id`    | UUID                         |
| `action_type` | TEXT (`view`, `like`, `skip`)|
| `timestamp`   | TIMESTAMP                    |

---

#### 4. `user_profiles`

| Column        | Type     |
|---------------|----------|
| `user_id`     | UUID     |
| `top_tags`    | TEXT[]   |
| `updated_at`  | TIMESTAMP |

---

#### 5. `video_popularity`

| Column         | Type     |
|----------------|----------|
| `video_id`     | UUID     |
| `score`        | FLOAT    |
| `last_updated` | TIMESTAMP |

---

## 🌐 5. API Design

---

### 🔹 GET `/recommendations/{user_id}`

Returns top N personalized videos.

**Sample Response**
```json
[
  {
    "video_id": "v1",
    "title": "Epic Funny Fail",
    "url": "https://...",
    "tags": ["funny", "meme"]
  },
  ...
]
````

Uses **Redis cache**. Fallback to DB if cache is cold.

---

### 🔹 POST `/video_action`

Tracks feedback on videos.

**Sample Request**

```json
{
  "user_id": "u1",
  "video_id": "v1",
  "action_type": "like"
}
```

Event is sent to **Kafka** for processing.

---

### 🔹 GET `/trending`

Returns globally popular videos.

---

## ⚡ 6. Caching Strategy

Using **Redis** for fast access.

| Cache Key                   | Description             | TTL     |
| --------------------------- | ----------------------- | ------- |
| `recommendations:{user_id}` | Top videos for the user | 10 mins |
| `popular_videos`            | Top trending videos     | 15 mins |
| `user_profile:{user_id}`    | Precomputed top tags    | 30 mins |

---

## 🔄 7. Streaming with Kafka

All user interactions (view, like, skip) are published to a Kafka topic.

### Kafka Topic: `video_actions_stream`

**Sample Kafka Message**

```json
{
  "user_id": "u1",
  "video_id": "v1",
  "action_type": "view",
  "timestamp": "2025-09-23T12:00:00Z"
}
```

**Kafka Consumers:**

* Insert into `video_actions`
* Update `video_popularity`
* Update `user_profiles`
* Trigger offline model retraining

---

## 🔟 8. Top-N Recommendation Generation

### Step-by-Step Flow:

1. User watches or likes a video → tracked via API
2. Action is sent to Kafka → stored in DB
3. Batch job (hourly):

   * Updates popularity score
   * Updates user’s top tags
4. Match user tags with video tags (content-based)
5. Find similar users (collaborative)
6. Blend recommendations
7. Store top results in `recommendations:{user_id}` Redis key

---

## 📊 9. Sample Data

### User

```json
{
  "user_id": "u1",
  "name": "Alice"
}
```

### Video

```json
{
  "video_id": "v1",
  "title": "Epic Dog Fails",
  "tags": ["funny", "animals"],
  "url": "https://..."
}
```

### Video Action

```json
{
  "user_id": "u1",
  "video_id": "v1",
  "action_type": "like"
}
```

---

## 🧠 10. How It All Works (Human-Friendly Summary)

1. User watches or likes a video → system stores it
2. It tracks what kind of videos (tags) the user likes
3. Every hour, it matches new videos with those tags
4. Also checks what similar users liked
5. Combines the results and stores them in Redis
6. When the user opens the app, recommendations load instantly

---

## ✅ Recap

| Area                    | Summary                                          |
| ----------------------- | ------------------------------------------------ |
| Functional Requirements | Watch, like, recommend videos                    |
| Recommendation Types    | Content-based + collaborative + hybrid           |
| DB Tables               | Users, Videos, Video Actions, Profiles           |
| APIs                    | `/recommendations`, `/video_action`, `/trending` |
| Caching                 | Redis for fast lookup                            |
| Streaming               | Kafka for real-time ingestion                    |
| Top-N Logic             | Hourly batch job + Redis + API serving           |

---

## 🔧 Future Improvements

* Add Machine Learning model for smarter predictions
* Personalize feed by time of day, location
* Add A/B testing for ranking models
* Real-time updates (event-driven scores)

---

```

---

✅ **Now you can paste this directly into GitHub** as a `README.md`, wiki doc, or system spec.

Would you like me to also:
- Create a diagram (architecture or data flow)?
- Help you turn this into a small working project?
- Export this as a PDF or Markdown file?

Let me know!
```

## How the recommendation engine works at high level (based on the diagram we created)
---

### 1. **User Action**

When someone watches a movie or taps “like,” the **video app sends an event** to **Apache Kafka**.
Kafka is a fast message pipeline that delivers every user action to the rest of the system.

**🛠️ Technologies**: React/Browser/Mobile App, Apache Kafka

---

### 2. **Immediate Storage in MySQL**

A **Spring Boot service** saves key facts from those events in **MySQL** so we always know the *current* state of each user—
their profile, their recent watch history, and the latest recommendation list.

**🛠️ Technologies**: Spring Boot, MySQL, Kafka Consumer

---

### 3. **Long-Term Storage in S3**

At the same time, **Kafka Connect** writes a full copy of every event into **Amazon S3**.
S3 is a huge, low-cost “data lake,” perfect for keeping the *entire* click-and-watch history, even years old,
which will later feed the machine-learning model.

**🛠️ Technologies**: Kafka Connect, Amazon S3, S3 Data Lake

---

### 4. **Analytics & Feature Engineering**

An **Analytics Service** reads the live Kafka streams, cleans the raw events, and creates useful features such as
“User 123 watched 10 sci-fi movies this week” or
“User 456 stops comedies after 5 minutes.”
These derived features are also stored in S3.

**🛠️ Technologies**: Spring Boot (Analytics Service), Apache Kafka Streams or Kafka Consumer, Amazon S3

---

### 5. **Machine-Learning Training and Hosting**

**Amazon SageMaker** takes the event history and feature files from S3 to train a **recommendation model**.
The model learns patterns—like which users share tastes or which tags tend to go together.
After training, SageMaker hosts the model behind a simple HTTPS endpoint so other services can call it.

**🛠️ Technologies**: Amazon SageMaker, Amazon S3, HTTPS API

---

#### 📊 How SageMaker Decides What to Recommend

When SageMaker receives user features (e.g., watch history, genres, engagement patterns), it uses a machine learning model to rank videos based on predicted relevance.
Depending on the model design, it may use:

---

##### 🧠 1. Content-Based Filtering

* Recommends items similar to those the user already liked/watched
* Based on attributes like:

  * Genre, tags, actors, directors, runtime, etc.
* Great for cold-start scenarios when collaborative data is sparse

✅ Personalized based on user's preferences
❌ Can get repetitive

---

##### 👥 2. Collaborative Filtering

* Recommends what **similar users** liked
* Two types:

  * **User-based**: Find users like you
  * **Item-based**: Find items similar to what you liked

✅ Taps into crowd behavior
❌ Doesn’t work well for new users or new items (cold-start problem)

---

##### 🔀 3. Hybrid Models

* Combine content-based + collaborative filtering
* Use rule-based blending, weighted scoring, or deep learning

✅ Best of both worlds
✅ Robust against cold-start
✅ Most common in production

---

##### 🧬 4. Other Real-World Approaches

* **Contextual Bandits**: Adapt in real-time using rewards (clicks, watch time)
* **Sequence Models (RNNs, Transformers)**: Understand user watch order
* **Graph Neural Networks**: Model user-video-tag relationships
* **Trend Boosting & Editorial Injection**: Inject new or promoted content into the list

---

### 6. **Generating Recommendations**

The **Recommendation Service** (a **Spring Boot microservice**) is the bridge between the app and the model.

* When a user opens the app, this service:

  * Pulls the **latest user state** from **MySQL** (watch history, profile)
  * Sends it to the **SageMaker endpoint**
  * Receives a **ranked list of recommended videos**

**🛠️ Technologies**: Spring Boot (Recommendation Service), REST API (to SageMaker), JSON over HTTPS
**🗂️ Data Source**: MySQL is the **source of truth** for current user state (not S3 or Kafka)

---

### 7. **Fast Delivery with Redis**

The Recommendation Service **stores that ranked list in Redis**, an in-memory cache.
Redis keeps the list ready so the system doesn’t have to recalculate it for every small action.

**🛠️ Technologies**: Redis, Spring Data Redis or Jedis (Java Redis client)

---

### 8. **Instant Display**

Later—maybe seconds or minutes afterward—when the user **refreshes their home screen**,
the app simply reads the already-prepared list straight from Redis.
Because Redis lives in memory, it responds in a few milliseconds, and the personalized recommendations appear **instantly**.

**🛠️ Technologies**: React/Browser/Mobile App, Redis, REST API (to fetch recommendations)

---

## 🧩 Recommendation Caching Strategy

* Redis recommendations can be updated:

  * **Immediately after user watches something**
  * Or on a **time-based TTL** (e.g., every 15 min)
  * Or after **X user interactions** (e.g., after 3 new videos)
* This reduces load on SageMaker and keeps responses fast

---

### 🧵 One-Sentence Summary

User actions flow **from the app → Kafka → MySQL (current state) and S3 (history) → Analytics for feature creation → SageMaker for training/hosting → Recommendation Service → Redis cache → back to the app**,
so every click is captured, the model keeps learning, and the viewer sees fresh video suggestions with almost no delay.

**🛠️ Core Tech Stack Summary**:
React/Mobile App, Apache Kafka, Spring Boot (Microservices), MySQL, Redis, Amazon S3, Amazon SageMaker, Kafka Connect

---

Great — let’s analyze **both use cases** using your existing recommendation flow and architecture.

We’ll walk through:

* What data is available
* What the system does
* What the user sees

This will help you clearly document and explain how recommendations evolve between a user’s first interaction and returning sessions.

---

## ✅ Use Case 1: **User comes online for the first time and watches a video**

### 🔍 What’s happening:

* The user has **no prior profile**, no watch history, and no cached recommendations.
* They start by watching a video (e.g., a sci-fi movie).

### 🔁 System Behavior:

1. The watch event is sent to **Kafka**
2. **MySQL** is updated with the user’s first interaction
3. **Kafka Connect** streams the event into **S3** (for long-term storage)
4. **Analytics Service** starts creating user features like:

   * “Watched 1 sci-fi video”
   * “Genre: sci-fi”
5. The **Recommendation Service** pulls the latest user state (now non-empty) and sends it to **SageMaker**
6. **SageMaker** returns a ranked list of similar or popular videos in sci-fi or related genres
7. The list is **stored in Redis** for fast delivery
8. The user opens the homepage → recommendations are served instantly from Redis

### 👀 What videos will the user see?

* A **personalized list based on their very first action**:

  * Videos similar to the one they just watched
  * Popular sci-fi titles
  * New releases in the same category
  * "People who watched this also watched…" logic
* Generated by **SageMaker** using collaborative filtering or content-based models

---

## ✅ Use Case 2: **User returns the next day and watches another video**

### 🔍 What’s different now:

* The user **already has a profile**
* A previous watch history is stored in **MySQL**
* Recommendations from the last session may still be in **Redis**

### 🔁 System Behavior:

1. The user opens the app — the homepage may show **cached recommendations from Redis**

   * If still valid (based on TTL), Redis serves them instantly
2. The user watches another video
3. Another event flows through Kafka → MySQL + S3 + Analytics
4. Depending on your **cache invalidation strategy**, one of two things happens:

   * **If cache TTL is expired** OR a trigger threshold is hit (e.g., 3 new events):

     * Recommendation Service recalculates
     * SageMaker is called again using updated user profile
     * Redis is updated with the new list
   * **If cache is still valid**, Redis is not updated yet — user keeps seeing old recommendations

### 👀 What videos will the user see?

* If cache is **fresh**:

  * Slightly outdated but still relevant list from previous session
* If cache is **refreshed**:

  * A **more accurate recommendation list** that includes:

    * Combined insights from both videos watched
    * New genre patterns (e.g., sci-fi and comedy)
    * More personalized ranking

---

## Summary Table

| Feature                 | Use Case 1 – First-Time User        | Use Case 2 – Returning User               |
| ----------------------- | ----------------------------------- | ----------------------------------------- |
| Profile Exists?         | ❌ No                                | ✅ Yes                                     |
| Watch History           | 🚫 None → 1 video watched           | ✅ History grows with each session         |
| Redis Cached?           | ❌ No → Created after first video    | ✅ Maybe (depends on TTL or trigger logic) |
| Recommendation Accuracy | 🟡 Low (1 video basis)              | 🟢 Higher (multiple interactions)         |
| Recommendation Strategy | Content-based or cold-start logic   | Personalized collaborative filtering      |
| Redis Update?           | ✅ Yes (first recommendation stored) | ✅ Conditionally (TTL or thresholds)       |

---

## Final Developer Insight

> The **first-time experience** relies heavily on content similarity and cold-start fallback logic, while the **returning experience** leverages a progressively richer user profile to provide deeper personalization — all backed by the **Spring Boot → Kafka → MySQL/S3 → Analytics → SageMaker → Redis** flow.

---




