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

## 3️⃣ Original Source File (Before Any Processing)

| Property          | Example Value                  |
|-------------------|--------------------------------|
| File Name         | `vacation.mov`                 |
| Resolution        | 3840x2160 (4K)                 |
| Duration          | 1 hour                         |
| Codec             | ProRes / H.264 (camera output) |
| Bitrate           | ~50 Mbps (high-quality capture)|
| Approx File Size  | ~20 GB                          |

This is the raw video as it comes from the camera or initial upload—large, high-bitrate, and not optimized for streaming.

---

## 🎯 Full-File Renditions (Before Chunking)

These are the **complete transcoded files** produced from the original source **before** they are split into adaptive streaming segments.

| Rendition Name | Resolution  | Approx Bitrate | Typical Devices        | Approx File Size (1-hour video)* |
|----------------|------------|---------------|------------------------|----------------------------------|
| Low            | 640x360    | ~0.6 Mbps     | Phones on slow data    | ~250 MB                          |
| Medium         | 854x480    | ~1 Mbps       | Most mobile, tablets   | ~450 MB                          |
| HD             | 1280x720   | ~2.5 Mbps     | Tablets, laptops       | ~1.1 GB                           |
| Full HD        | 1920x1080  | ~5 Mbps       | Laptops, TVs           | ~2.2 GB                           |
| 4K (optional)  | 3840x2160  | ~12 Mbps     | TVs, high-end monitors | ~5.3 GB                           |

\*Estimates assume H.264 encoding at 30 fps and ~1-hour duration.  
Actual sizes vary by codec, frame rate, and content complexity.


## Example Renditions Table

| Rendition | Resolution | Approx Bitrate | Typical Devices | Chunks or Full File? | Approx Chunk Size* |
|-----------|-----------|---------------|----------------|----------------------|-------------------|
| Low | 360p | ~0.6 Mbps | Phones on slow data | Chunks | ~0.15 MB |
| Medium | 480p | ~1 Mbps | Most mobile, tablets | Chunks | ~0.25 MB |
| HD | 720p | ~2.5 Mbps | Tablets, laptops | Chunks | ~0.6 MB |
| Full HD | 1080p | ~5 Mbps | Laptops, TVs | Chunks | ~1.2 MB |

\*Assuming ~4-second HLS segments; actual size varies.

---

## 4️⃣ Sample Master Manifest (HLS) – With Full URLs

The master manifest lists every available rendition and the **full location** of its variant playlist.

```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=600000,RESOLUTION=640x360
https://cdn.example.com/video123/low/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
https://cdn.example.com/video123/medium/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
https://cdn.example.com/video123/high/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
https://cdn.example.com/video123/fullhd/playlist.m3u8
```

## 5️⃣ Detailed Variant Playlist Example – 720p HD Rendition

If the player selects the **720p HD** stream, it downloads the **variant playlist**.  

```
https://cdn.example.com/video123/high/playlist.m3u8
```

This file simply lists each 4-second video chunk with its full CDN URL:

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:4.0,
https://cdn.example.com/video123/high/segment0.ts
#EXTINF:4.0,
https://cdn.example.com/video123/high/segment1.ts
#EXTINF:4.0,
https://cdn.example.com/video123/high/segment2.ts
#EXTINF:4.0,
https://cdn.example.com/video123/high/segment3.ts
#EXT-X-ENDLIST
```

Each segmentN.ts is a ~4-second MPEG-TS chunk of the 720p rendition.

The player downloads these sequentially and can switch to another rendition’s playlist at any time.

---

### **Section 6 – End-to-End Flow**

## 6️⃣ End-to-End Flow

1. **Upload / Ingest** – Creator uploads raw `.mov` or `.mp4` file.  
2. **Transcode** – Platform creates multiple renditions (different resolutions, bitrates, codecs).  
3. **Package & Chunk** – Each rendition is split into small segments (chunks) and manifests are generated.  
4. **Store** – Final chunks and manifests are saved in **origin storage** (e.g., Amazon S3).  
5. **Distribute** – A **CDN** caches and serves these chunks to viewers globally.  
6. **Playback** – The user’s video player downloads the **master manifest**, selects a rendition, and adaptively switches quality during playback.
