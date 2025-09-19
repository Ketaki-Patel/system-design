
Media processing is an important part of any video platform. It takes raw video files, validates it, converts them into different formats and sizes, and delivers them to viewers on different devices and network speeds.  

This document explains the **high-level flow** of media processing—from uploading raw video to adaptive streaming—covering key steps like **transcoding**, **chunking**, **packaging** and **distribution**.  

## High-Level Media Processing Flow

1. **Upload / Ingest** – Creator uploads raw `.mov` or `.mp4` file.

2. **Transcode** – Platform creates multiple renditions (different resolutions, bitrates, codecs).

3. **Package & Chunk** – Files are split into small segments and manifests are generated.

4. **Store** – Final chunks and manifests live in **origin storage** (e.g., Amazon S3).

5. **Distribute** – A **CDN** caches and serves these chunks to viewers globally.

6. **Playback** – User’s video player downloads the **master manifest**, selects a rendition, and adaptively switches quality during playback.


<img width="573" height="800" alt="image" src="https://github.com/user-attachments/assets/5625f909-84e8-4446-b63e-fc28f1c344ee" />
