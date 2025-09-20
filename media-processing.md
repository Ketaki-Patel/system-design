## 1️⃣ Preface

Media processing is an important part of any video platform. It takes raw video files, validates it, converts them into different formats and sizes, and delivers them to viewers on different devices and network speeds.  

This document describes a **highly available, scalable architecture** for media processing, based on the uploaded diagram, using **React frontend**, **Spring Boot backend**, **AWS S3**, message queues, and a metadata database.


This document explains the **high-level flow** of media processing—from uploading raw video to adaptive streaming—covering key steps like **transcoding**, **chunking**, **packaging** and **distribution**.  

## High-Level Media Processing Flow

1. **Upload / Ingest** – Creator uploads raw `.mov` or `.mp4` file.

2. **Transcode** – Platform creates multiple renditions (different resolutions, bitrates, codecs).

3. **Package & Chunk** – Files are split into small segments and manifests are generated.

4. **Store** – Final chunks and manifests live in **origin storage** (e.g., Amazon S3).

5. **Distribute** – A **CDN** caches and serves these chunks to viewers globally.

6. **Playback** – User’s video player downloads the **master manifest**, selects a rendition, and adaptively switches quality during playback.

For media processing background details --> https://github.com/Ketaki-Patel/system-design/blob/main/media-processing-primer.md


<img width="573" height="800" alt="image" src="https://github.com/user-attachments/assets/5625f909-84e8-4446-b63e-fc28f1c344ee" />

### Media Processing Architecture

<img width="994" height="720" alt="image" src="https://github.com/user-attachments/assets/7ef43ce9-584d-4dab-97ae-f1b665f6acee" />

Here’s a **comprehensive document** explaining the architecture in your diagram, how it works end-to-end, and suggested database tables with schema. I’ve written it in **Markdown** so you can copy-paste to GitHub or docs.

---

# Scalable Media Processing System Architecture

## 2️⃣ System Components Overview

| Component                          | Responsibility                                                                                                                |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Client (React / Browser)**       | File upload interface; slices large files, uploads via presigned URLs                                                         |
| **Upload Service**                 | Authenticates user, validates file, generates presigned URLs, coordinates multipart upload, triggers upload completion events |
| **Media Processing Service**       | Transcodes video, creates multiple renditions, performs HLS/DASH packaging                                                    |
| **Metadata Service**               | Manages video metadata, tracks processing status, maps files and user info                                                    |
| **Message Queues**                 | Event-driven decoupling for processing pipelines (`Upload → Media Queue`, `Media → Metadata Queue`)                           |
| **AWS S3 Bucket**                  | Stores raw videos, processed renditions, chunked segments, and manifests                                                      |
| **Metadata Database (PostgreSQL)** | Stores video records, processing status, user associations                                                                    |

---

For large media (abovee 10MB or more), multipart upload with presigned URLs (the earlier design) is strongly recommended.

❌ For large videos (hundreds of MB or GB), direct upload from seveer is not advisable:

pushes all bytes through your Spring Boot server (higher bandwidth/memory load),

cannot resume if the connection drops,

and will require long timeouts.

## 3️⃣ High-Level Flow

### Step 1 – Client Upload

* User selects a video in React.
* For **large files**, React slices the file into chunks (e.g., 8 MB).
* Sends an API request to Spring Boot **Upload Service** to initiate the upload.

### Step 2 – Upload Service

* Validates the file type and size.
* Initiates **multipart upload** on S3 → receives `uploadId`.
* Generates **presigned URLs** for each chunk.
* Returns presigned URLs to React client.

### Step 3 – Client Uploads Chunks

* React uploads each chunk **directly to S3** using presigned URLs (PUT request).
* After uploading all chunks, React calls backend to **complete the multipart upload**.
* Backend triggers an **upload completion event** to `Upload → Media Queue`.

### Step 4 – Media Processing Service

* Listens to `Upload → Media Queue`.
* Downloads the raw video from S3 `/raw/`.
* Performs **transcoding** to create multiple **renditions** (resolutions, bitrates, codecs).
* Performs **HLS/DASH packaging** → splits into small segments and generates manifests.
* Uploads **processed renditions**, **segments**, and **manifests** to S3 `/processed/`, `/segments/`, `/manifests/`.
* Publishes completion event to `Media → Metadata Queue`.

### Step 5 – Metadata Service

* Listens to `Media → Metadata Queue`.
* Updates database with:

  * Processed video details
  * File locations (S3 paths)
  * Processing status
  * Rendition information
* Metadata is used for **playback and CDN delivery**.

### Step 6 – Playback

* Client requests a video.
* React (or any player) downloads the **master manifest** from S3.
* Player selects appropriate rendition based on network speed and device, adaptively switching quality.

---

## 4️⃣ S3 Storage Structure

| Folder        | Content                                        |
| ------------- | ---------------------------------------------- |
| `/raw/`       | Original uploaded files                        |
| `/processed/` | Transcoded renditions                          |
| `/segments/`  | HLS/DASH chunks                                |
| `/manifests/` | Master and variant playlists (`.m3u8`, `.mpd`) |

---

## 5️⃣ Database Design

### Table: `videos`

| Column        | Type      | Description                                           |
| ------------- | --------- | ----------------------------------------------------- |
| `id`          | UUID (PK) | Unique video ID                                       |
| `user_id`     | UUID (FK) | Owner of the video                                    |
| `file_name`   | VARCHAR   | Original filename                                     |
| `s3_raw_path` | VARCHAR   | S3 path to raw video                                  |
| `upload_id`   | VARCHAR   | S3 multipart upload ID                                |
| `status`      | VARCHAR   | Processing status (`UPLOADED`, `PROCESSING`, `READY`) |
| `created_at`  | TIMESTAMP | Upload timestamp                                      |
| `updated_at`  | TIMESTAMP | Last update timestamp                                 |

### Table: `video_renditions`

| Column          | Type      | Description                                  |
| --------------- | --------- | -------------------------------------------- |
| `id`            | UUID (PK) | Unique rendition ID                          |
| `video_id`      | UUID (FK) | Parent video                                 |
| `resolution`    | VARCHAR   | e.g., 360p, 480p, 720p, 1080p                |
| `bitrate`       | INT       | Approx bitrate in kbps                       |
| `s3_path`       | VARCHAR   | S3 path for processed rendition              |
| `chunk_count`   | INT       | Number of HLS/DASH segments                  |
| `manifest_path` | VARCHAR   | Path to variant playlist (`.m3u8` or `.mpd`) |
| `created_at`    | TIMESTAMP | Timestamp                                    |

### Table: `video_segments`

| Column            | Type      | Description         |
| ----------------- | --------- | ------------------- |
| `id`              | UUID (PK) | Unique segment ID   |
| `rendition_id`    | UUID (FK) | Parent rendition    |
| `sequence_number` | INT       | Segment order       |
| `s3_path`         | VARCHAR   | Path to `.ts` chunk |
| `duration`        | FLOAT     | Duration in seconds |

---

## 6️⃣ Key Features

* **Scalable**: All services are stateless, behind a load balancer; multiple instances can handle concurrent uploads and processing.
* **Decoupled**: Event queues (Upload → Media, Media → Metadata) allow asynchronous processing.
* **Efficient Upload**: Large files uploaded directly to S3 using **presigned URLs**; backend does not handle file payload.
* **Adaptive Streaming**: HLS/DASH manifests allow client to switch renditions based on bandwidth.
* **Metadata Tracking**: Database keeps track of video, renditions, and processing status.

---

This document can now serve as a **primer for your media processing system design**, and the tables provide a **reference for implementing DB schema**.

---



