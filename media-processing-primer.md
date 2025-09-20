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

# File Upload process 
## Large File upload with presigned URLS
## Small File upload directly from backend

## Large File upload with presigned URLS (lets understand with some code and api)

- **sequenceDiagram**
```
    sequenceDiagram
    participant Client as React Client
    participant API as Spring Boot Auth/API
    participant S3 as AWS S3
    participant DB as Metadata/Upload Tracking

    Client->>API: 1. Upload Request
    API->>S3: 2. Create Multipart Upload
    S3-->>API: 3. Return uploadId
    API-->>Client: 4. Return uploadId to client

    Client->>API: 5. Request presigned URLs for each chunk
    API->>S3: 6. Generate Presigned URLs
    S3-->>API: 7. Return presigned URLs
    API-->>Client: 8. Return presigned URLs

    loop Upload each chunk i = 1..N
        Client->>S3: 9. Upload chunk i via PUT
        S3-->>Client: 10. Return ETag_i
    end

    Client->>API: 11. Complete Multipart Upload (with all ETags)
    API->>S3: 12. Complete upload
    S3-->>S3: 13. File assembled (Final Video Object in S3)
    
    API->>DB: 14. Optional: update DB / enqueue for transcoding
  ```


Absolutely! Here’s a **React example** for uploading a large video file using **multipart upload with presigned URLs**, along with **sample input and output in comments** for clarity.

---
### React Client

```javascript
import React, { useState } from "react";

export default function LargeFileUpload() {
  const [file, setFile] = useState(null);

  const CHUNK_SIZE = 8 * 1024 * 1024; // 8 MB chunks

  const handleFileChange = (e) => setFile(e.target.files[0]);

  const uploadFile = async () => {
    if (!file) return;

    // -----------------------------
    // 1️⃣ Initiate Multipart Upload
    // -----------------------------
    /*
      Sample Input (to backend):
      {
        "fileName": "my_video.mp4",
        "contentType": "video/mp4",
        "fileSize": 1073741824
      }

      Sample Response (from backend):
      {
        "uploadId": "exampleUploadId123",
        "fileName": "my_video.mp4"
      }
    */
    const initResp = await fetch("/uploads/initiate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        fileName: file.name,
        contentType: file.type,
        fileSize: file.size,
      }),
    });
    const { uploadId } = await initResp.json();

    // -----------------------------
    // 2️⃣ Generate Presigned URLs
    // -----------------------------
    const totalParts = Math.ceil(file.size / CHUNK_SIZE);
    const partNumbers = Array.from({ length: totalParts }, (_, i) => i + 1);

    /*
      Sample Input (to backend):
      {
        "partNumbers": [1,2,3,...]
      }

      Sample Response (from backend):
      [
        { "partNumber": 1, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=..." },
        ...
      ]
    */
    const urlResp = await fetch(`/uploads/${uploadId}/parts`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ partNumbers }),
    });
    const presignedUrls = await urlResp.json();

    // -----------------------------
    // 3️⃣ Upload Chunks
    // -----------------------------
    const uploadedParts = [];

    for (let i = 0; i < totalParts; i++) {
      const start = i * CHUNK_SIZE;
      const end = Math.min(start + CHUNK_SIZE, file.size);
      const chunk = file.slice(start, end);

      const res = await fetch(presignedUrls[i].url, {
        method: "PUT",
        body: chunk,
      });

      uploadedParts.push({
        partNumber: i + 1,
        eTag: res.headers.get("ETag"),
      });
    }

    // -----------------------------
    // 4️⃣ Complete Multipart Upload
    // -----------------------------
    /*
      Sample Input (to backend):
      {
        "parts": [
          { "partNumber": 1, "eTag": "etag-value-1" },
          { "partNumber": 2, "eTag": "etag-value-2" },
          ...
        ]
      }

      Sample Response (from backend):
      {
        "message": "Upload completed successfully",
        "s3Key": "raw-videos/my_video.mp4"
      }
    */
    const completeResp = await fetch(`/uploads/${uploadId}/complete`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ parts: uploadedParts }),
    });
    const result = await completeResp.json();

    alert(result.message);
  };

  return (
    <div>
      <input type="file" onChange={handleFileChange} />
      <button onClick={uploadFile}>Upload Large Video</button>
    </div>
  );
}
```

---

### ✅ Explanation

1. **Step 1 – Initiate Upload:** React sends file info (`fileName`, `contentType`, `fileSize`) → Backend returns `uploadId`.
2. **Step 2 – Get Presigned URLs:** React requests URLs for each chunk → Backend returns an array of `{partNumber, url}`.
3. **Step 3 – Upload Chunks:** React slices the file and uploads each chunk directly to S3 using the presigned URLs.
4. **Step 4 – Complete Upload:** React sends `{partNumber, eTag}` array → Backend calls `CompleteMultipartUpload` on S3 → file is assembled.

This mirrors the **Java Spring Boot API example** and gives a clear **end-to-end workflow** for large video uploads.

---

### Java Spring Boot Backend 
 Here's a **clean Spring Boot example** with **both sample input and sample response in comments** for each step of the multipart upload flow using presigned URLs:

```java
@RestController
@RequestMapping("/uploads")
public class VideoUploadController {

    // ----------------------------
    // 1️⃣ Initiate Multipart Upload
    // ----------------------------
    /*
        Sample Input (client → backend):
        {
            "fileName": "my_video.mp4",
            "contentType": "video/mp4",
            "fileSize": 1073741824  // 1 GB
        }

        Sample Response (backend → client):
        {
            "uploadId": "exampleUploadId123",
            "fileName": "my_video.mp4"
        }
    */
    @PostMapping("/initiate")
    public Map<String, String> initiateUpload(@RequestBody Map<String, Object> payload) {
        String fileName = (String) payload.get("fileName");
        String contentType = (String) payload.get("contentType");

        // Normally call S3 createMultipartUpload here
        String uploadId = "exampleUploadId123";

        return Map.of(
            "uploadId", uploadId,
            "fileName", fileName
        );
    }

    // --------------------------------
    // 2️⃣ Generate Pre-signed URLs for Parts
    // --------------------------------
    /*
        Sample Input (client → backend):
        {
            "partNumbers": [1, 2, 3, 4, 5]
        }

        Sample Response (backend → client):
        [
            { "partNumber": 1, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE1" },
            { "partNumber": 2, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE2" },
            { "partNumber": 3, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE3" },
            { "partNumber": 4, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE4" },
            { "partNumber": 5, "url": "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE5" }
        ]
    */
    @PostMapping("/{uploadId}/parts")
    public List<Map<String, Object>> generatePartUrls(
            @PathVariable String uploadId,
            @RequestBody Map<String, Object> payload) {

        List<Integer> partNumbers = (List<Integer>) payload.get("partNumbers");

        List<Map<String, Object>> presignedUrls = new ArrayList<>();
        for (Integer partNumber : partNumbers) {
            presignedUrls.add(Map.of(
                "partNumber", partNumber,
                "url", "https://bucket.s3.amazonaws.com/my_video.mp4?X-Amz-Signature=EXAMPLE" + partNumber
            ));
        }

        return presignedUrls;
    }

    // ----------------------------
    // 3️⃣ Complete Multipart Upload
    // ----------------------------
    /*
        Sample Input (client → backend):
        {
            "parts": [
                { "partNumber": 1, "eTag": "etag-value-1" },
                { "partNumber": 2, "eTag": "etag-value-2" },
                { "partNumber": 3, "eTag": "etag-value-3" },
                { "partNumber": 4, "eTag": "etag-value-4" },
                { "partNumber": 5, "eTag": "etag-value-5" }
            ]
        }

        Sample Response (backend → client):
        {
            "message": "Upload completed successfully",
            "s3Key": "raw-videos/my_video.mp4"
        }
    */
    @PostMapping("/{uploadId}/complete")
    public Map<String, String> completeUpload(@PathVariable String uploadId,
                                 @RequestBody Map<String, Object> payload) {
        // Normally call S3 CompleteMultipartUpload here
        return Map.of(
            "message", "Upload completed successfully",
            "s3Key", "raw-videos/my_video.mp4"
        );
    }
}
```

✅ **Explanation**

1. Step 1: Client sends filename, size, type → Backend returns `uploadId`.
2. Step 2: Client requests URLs for each chunk → Backend returns array of presigned URLs.
3. Step 3: Client sends `{partNumber, eTag}` for all uploaded parts → Backend completes the multipart upload.

This format makes it **easy to see exactly what the client sends and what it receives** for each step.

If you want, I can also add a **matching React sample showing file chunking and calling these APIs** with these exact payloads. Do you want me to do that?







   
