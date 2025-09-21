
## 🧭 Overview

This design defines a **scalable, distributed, fault-tolerant job scheduler** with support for:

* High-concurrency job submission
* Background distributed execution
* Retry and failure recovery
* Safe coordination across multiple instances
* Observability and status tracking
* Persistent state using MySQL and queue systems

[Distributed Scheduler Ref doc](https://www.linkedin.com/pulse/system-design-distributed-job-scheduler-keep-simple-stupid-ismail)
 
my db design and apis are bit different from this doc, so it is just for reference
---

<img width="559" height="316" alt="image" src="https://github.com/user-attachments/assets/baf25bfd-8063-48ff-9936-fb6fb559842e"/>

<img width="661" height="564" alt="image" src="https://github.com/user-attachments/assets/1a77585a-f1f0-4aaf-a0fd-0a85cf84ba54" />

Support for:

* Job retries
* Executor/node failure handling
* A new `job_executions` table to track execution attempts, retries, and timestamps

---

## 🏗️ High-Level Architecture

(Aligned with uploaded diagram)

### Components:

* **Load Balancer**
* **submitJob API**
* **viewJob API**
* **MySQL Database**
* **Job Scheduler Service**
* **Distributed Message Queue** (e.g., RabbitMQ, Kafka)
* **Job Executor Service**
* **File System / Blob Storage**

---

## 🔄 System Workflow (Expanded with Retry & Resilience)

### 💡 Summary


### 🔁 Enhanced Execution Flow (with Retry + Failure Recovery)

1. **Client submits a job** via API.

2. **submitJob** saves the job with `PENDING` status in the `jobs` table.

3. **Job Scheduler** (multiple instances) polls DB using locking and selects a batch of jobs.

4. It marks them as `QUEUED` and pushes them to a **queue**.

5. **Job Executors** consume the queue, mark jobs as `RUNNING`, and log an entry in the `job_executions` table.

6. If execution:

   * **Succeeds**: mark job `SUCCESS`, store result.
   * **Fails**: increment retry count, log failure in `job_executions`, and:

     * Retry (if attempts < max)
     * Else move to `FAILED` status.

7. **viewJob** API allows polling job status and results.

---

## 🧬 MySQL Schema

### Table: `jobs`

```sql
CREATE TABLE jobs (
    id CHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    payload JSON NOT NULL,
    status ENUM('PENDING', 'QUEUED', 'RUNNING', 'SUCCESS', 'FAILED') NOT NULL DEFAULT 'PENDING',
    retry_count INT DEFAULT 0,
    max_retries INT DEFAULT 3,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    result JSON DEFAULT NULL
);
```

### Table: `job_executions`

```sql
CREATE TABLE job_executions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_id CHAR(36),
    attempt_number INT NOT NULL,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME DEFAULT NULL,
    status ENUM('RUNNING', 'SUCCESS', 'FAILED') NOT NULL,
    error TEXT DEFAULT NULL,
    executor_id VARCHAR(255),
    FOREIGN KEY (job_id) REFERENCES jobs(id)
);
```

> Each job execution attempt logs its lifecycle independently.

---

## 🧪 Sample Data

```sql
INSERT INTO jobs (id, name, payload, status, max_retries)
VALUES 
(UUID(), 'ImageProcessing', JSON_OBJECT('imageUrl', 'http://img.com/1.png'), 'PENDING', 3),
(UUID(), 'PDFConversion', JSON_OBJECT('fileId', 'doc123'), 'QUEUED', 2),
(UUID(), 'DataAggregation', JSON_OBJECT('dataset', 'sales_2023'), 'SUCCESS', 1);
```

---

## 🔁 Polling with Safe Locking (Multi-Instance) (part of Scheduler Service)


To avoid job contention across schedulers:

```sql
START TRANSACTION;

UPDATE jobs
SET status = 'QUEUED',
    updated_at = NOW()
WHERE id IN (
    SELECT id
    FROM jobs
    WHERE status = 'PENDING'
    ORDER BY created_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
RETURNING *;

COMMIT;
```
✅ Notes:

RETURNING * will return all columns of the updated rows.

Using FOR UPDATE SKIP LOCKED ensures multiple scheduler instances don’t pick the same jobs.

Atomicity is maintained with START TRANSACTION + COMMIT.

This is fully compatible with MySQL 8.0.27+.
This is now a single querry that updates and returns the affecteed rows-perfect for sending jobs directly to executors.

---

## 📤 Job Execution with Retry Logic (part of Job Executor)

### Executor Steps:

1. Polls job from queue.
2. Marks job as `RUNNING`.
3. Inserts execution record:

```sql
INSERT INTO job_executions (job_id, attempt_number, status, executor_id)
VALUES (?, ?, 'RUNNING', ?);
```

4. Executes job logic.
5. On **Success**:

```sql
UPDATE jobs
SET status='SUCCESS', result=JSON_OBJECT('outputUrl', ?)
WHERE id=?;

UPDATE job_executions
SET status='SUCCESS', ended_at=NOW()
WHERE job_id=? AND attempt_number=?;
```

6. On **Failure**:

```sql
UPDATE job_executions
SET status='FAILED', ended_at=NOW(), error=?
WHERE job_id=? AND attempt_number=?;

UPDATE jobs
SET retry_count = retry_count + 1,
    status = CASE 
               WHEN retry_count + 1 >= max_retries THEN 'FAILED' 
               ELSE 'PENDING' 
             END
WHERE id=?;
```

> Failed jobs are sent back to scheduler for retry (up to `max_retries`).

---

## 📜 API Design

### 1. **POST** `/api/job`

**Input:**

```json
{
  "name": "ImageProcessing",
  "payload": { "imageUrl": "http://img.com/1.png" },
  "maxRetries": 2
}
```

**Output:**

```json
{
  "jobId": "b123-456",
  "status": "PENDING"
}
```

---

### 2. **GET** `/api/job/{id}`

**Output:**

```json
{
  "jobId": "b123-456",
  "name": "ImageProcessing",
  "status": "FAILED",
  "retryCount": 2,
  "maxRetries": 2,
  "executions": [
    {
      "attempt": 1,
      "startedAt": "2023-09-20T10:00:00Z",
      "status": "FAILED",
      "error": "Connection timeout"
    },
    {
      "attempt": 2,
      "startedAt": "2023-09-20T10:05:00Z",
      "status": "FAILED",
      "error": "Image not found"
    }
  ]
}
```

---

## 🧠 Executor Failure Handling

### Problem: Executor crashes mid-job

**Solution:**

* Record `job_executions` on start
* Periodically scan for:

```sql
SELECT * FROM job_executions 
WHERE status = 'RUNNING' 
  AND TIMESTAMPDIFF(MINUTE, started_at, NOW()) > 10;
```

* Mark job as `PENDING` again and increment retry (if allowed)

```sql
UPDATE jobs 
SET status='PENDING', retry_count=retry_count + 1 
WHERE id = ? AND retry_count < max_retries;
```

* Mark hanging `job_executions` as `FAILED`

---

## ✅ Job Status Lifecycle

```txt
PENDING → QUEUED → RUNNING → SUCCESS
                         ↘ FAILED → PENDING (retry) → ...
                                       ↘ (max retries reached) → FINAL FAILED
```

---

## ⚙️ Configuration Knobs

| Parameter           | Description                           |
| ------------------- | ------------------------------------- |
| `max_retries`       | Per-job maximum retry attempts        |
| `retry_delay`       | Delay before retrying (executor side) |
| `execution_timeout` | Max time before job is marked stuck   |

---

## 🧰 Additional Enhancements

* **Dead Letter Queue (DLQ):** Jobs failing after retries.
* **Job Priority:** Add `priority INT` column.
* **Job TTL:** `expires_at DATETIME` to discard old jobs.
* **Cron Jobs:** Add `schedule` column for recurring jobs.
* **Heartbeat:** Executors update `last_heartbeat` in a separate table.

---

## 🧪 Test Scenarios

| Test                              | Expected Outcome                       |
| --------------------------------- | -------------------------------------- |
| Submit 1000 jobs concurrently     | No duplicates, all saved               |
| Executor crashes during execution | Job re-attempted or marked failed      |
| Scheduler restarts mid-polling    | No duplicate queueing (thanks to lock) |
| 5 retries configured, 5 failures  | Final status: `FAILED` + DLQ entry     |
| Queue stalls                      | Executors idle but schedulers okay     |

---

## 📊 Monitoring & Observability

* **Metrics:**

  * Jobs per status
  * Queue length
  * Execution time
  * Retry count
* **Tools:**

  * Prometheus + Grafana
  * OpenTelemetry (tracing)
  * Centralized logging (ELK, Loki)

---

## 📌 Summary

✅ Horizontally scalable
✅ Fault-tolerant with retry and DLQ
✅ Concurrent-safe polling with `SKIP LOCKED`
✅ Observable, queryable execution logs
✅ Extensible architecture for cron, scheduling, metrics

```

Just save this text into a file named, for example, `distributed-job-scheduler.md` in your GitHub repository to preserve all formatting.
```

