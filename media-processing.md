# Media Processing & Streaming Primer

Lets get familar with some media proceessing terminologies.

---

## 1️⃣ Key Terms

| Term | Simple Definition |
|------|------------------|
| **Ingest** | First step where a raw video is uploaded or captured into the platform. |
| **Transcoding / Encoding** | Converting the original video into multiple **resolutions**, **bitrates**, and **codecs** so it plays well on different devices and network speeds. |
| **Packaging** | Splitting video into small **chunks (segments)** and creating a **playlist/manifest** so players can stream adaptively. |
| **Manifest / Playlist** | A small text file (e.g., `.m3u8` for HLS, `.mpd` for DASH) that lists the locations of all video segments and available quality levels. |
| **Adaptive Bitrate (ABR) Streaming** | Player starts at a safe quality and **switches between high or low resolution chunks** in real time based on network speed. |
| **Origin Storage** | The “source of truth” for finished video assets (often a cloud object store like Amazon S3). |
| **CDN (Content Delivery Network)** | A global network of edge servers that caches video segments close to viewers to reduce buffering and latency. |
| **Metadata** | Descriptive info about the video—title, duration, available variants, captions, thumbnails, owner, etc. |
| **Rendition** | Each distinct combination of **resolution + bitrate + codec** generated during transcoding (e.g., a 1080p/H.264/5 Mbps file). |

---

## 2️⃣ Adaptive Streaming (The Big Idea)

Instead of delivering one giant file, the platform:
1. **Transcodes** the original video into **multiple renditions** (different quality levels).
2. **Packages** each rendition into small chunks (commonly 2–10 seconds each).
3. Generates:
   - A **Master Manifest** listing all renditions.
   - One **Playlist per Rendition** listing that rendition’s chunks.

The player:
* Reads the **master manifest**.
* Starts streaming the lowest-risk quality.
* **Jumps to higher or lower renditions** on the fly as bandwidth changes.

---

## 3️⃣ Example Renditions Table

| Rendition | Resolution | Approx Bitrate | Typical Devices | Chunks or Full File? | Approx Chunk Size* |
|-----------|-----------|---------------|----------------|----------------------|-------------------|
| Low | 360p | ~0.6 Mbps | Phones on slow data | Chunks | ~0.15 MB |
| Medium | 480p | ~1 Mbps | Most mobile, tablets | Chunks | ~0.25 MB |
| HD | 720p | ~2.5 Mbps | Tablets, laptops | Chunks | ~0.6 MB |
| Full HD | 1080p | ~5 Mbps | Laptops, TVs | Chunks | ~1.2 MB |

\*Assuming ~4-second HLS segments; actual size varies.

---

## 4️⃣ Sample Master Manifest (HLS)

```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=600000,RESOLUTION=640x360
low/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
medium/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
high/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
fullhd/playlist.m3u8

