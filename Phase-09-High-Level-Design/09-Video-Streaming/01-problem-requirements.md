# Video Streaming (YouTube/Netflix) - Problem & Requirements

## Problem Statement

Design a video streaming platform that allows users to upload, process, store, and stream video content to millions of concurrent viewers. The system should support adaptive bitrate streaming, content delivery at global scale, and handle both user-generated content (YouTube) and licensed content (Netflix) use cases.

---

## Functional Requirements

### Core Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **Video Upload**         | Upload videos of various formats and sizes            | P0       |
| **Video Processing**     | Transcode to multiple resolutions and formats         | P0       |
| **Video Streaming**      | Stream videos with adaptive bitrate                   | P0       |
| **Video Search**         | Search videos by title, description, tags             | P0       |
| **Video Metadata**       | Title, description, thumbnails, duration              | P0       |
| **View Count**           | Track and display view counts                         | P0       |

### Extended Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **Recommendations**      | Personalized video recommendations                    | P1       |
| **Comments/Likes**       | User engagement features                              | P1       |
| **Subscriptions**        | Subscribe to channels                                 | P1       |
| **Watch History**        | Track user's viewing history                          | P1       |
| **Resume Playback**      | Continue watching from last position                  | P1       |
| **Live Streaming**       | Real-time video broadcast                             | P2       |
| **Offline Download**     | Download for offline viewing                          | P2       |

---

## Non-Functional Requirements

| Requirement      | Target                | Rationale                                      |
| ---------------- | --------------------- | ---------------------------------------------- |
| **Latency**      | < 2s startup time     | Users expect quick playback                    |
| **Throughput**   | 1M concurrent streams | Peak viewing (live events)                     |
| **Availability** | 99.99%                | Streaming is core service                      |
| **Durability**   | 99.999999999%         | Videos must never be lost                      |
| **Quality**      | Adaptive 240p-4K      | Support all network conditions                 |
| **Scalability**  | 2B users, 500M videos | YouTube scale                                  |

---

## User Stories

### Content Creator

1. **As a creator**, I want to upload videos easily so I can share my content.

2. **As a creator**, I want my videos processed quickly so viewers can watch soon after upload.

3. **As a creator**, I want to see analytics (views, watch time) so I can understand my audience.

### Viewer

1. **As a viewer**, I want videos to start quickly so I don't wait.

2. **As a viewer**, I want smooth playback without buffering so I can enjoy content.

3. **As a viewer**, I want quality to adapt to my network so playback doesn't stop.

4. **As a viewer**, I want to resume where I left off so I don't lose my place.

---

## Constraints & Assumptions

### Constraints

1. **Storage**: Videos are large (average 500MB, up to 12 hours)
2. **Processing**: Transcoding is CPU-intensive
3. **Bandwidth**: Video streaming dominates internet traffic
4. **Latency**: CDN required for global delivery

### Assumptions

1. Most videos are watched within first 24 hours of upload
2. 80% of views are for 20% of videos (power law)
3. Average video length: 10 minutes
4. Average viewing session: 30 minutes

---

## Out of Scope

1. Content moderation/copyright detection
2. Monetization/ads system
3. Creator studio analytics
4. Social features (sharing, playlists)
5. DRM implementation details

---

## Key Challenges

### 1. Video Processing Pipeline

**Challenge:** Process uploaded videos into multiple formats/resolutions

**Solution:**
- Distributed transcoding with worker pools
- Parallel processing of different resolutions
- Priority queues for popular creators

### 2. Global Content Delivery

**Challenge:** Serve videos with low latency worldwide

**Solution:**
- Multi-tier CDN architecture
- Edge caching for popular content
- Origin shield to protect origin servers

### 3. Adaptive Bitrate Streaming

**Challenge:** Provide smooth playback across varying network conditions

**Solution:**
- HLS/DASH protocols
- Multiple quality levels (240p to 4K)
- Client-side quality adaptation

### 4. View Count Aggregation

**Challenge:** Accurately count views at massive scale

**Solution:**
- Approximate counting with HyperLogLog
- Batch aggregation
- Eventually consistent counts

### 5. Storage Optimization

**Challenge:** Store petabytes of video efficiently

**Solution:**
- Tiered storage (hot/warm/cold)
- Deduplication for repeated uploads
- Compression optimization

---

## Success Metrics

| Metric              | Definition                              | Target        |
| ------------------- | --------------------------------------- | ------------- |
| **Startup Time**    | Time to first frame                     | < 2 seconds   |
| **Rebuffer Rate**   | % of sessions with rebuffering          | < 1%          |
| **Upload Success**  | % of uploads completing                 | > 99.9%       |
| **Processing Time** | Time from upload to playable            | < 30 minutes  |
| **CDN Hit Rate**    | % of requests served from CDN           | > 95%         |

---

## Video Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          VIDEO PROCESSING PIPELINE                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

Upload                  Processing                              Storage
  │                         │                                      │
  ▼                         ▼                                      ▼
┌─────────┐           ┌─────────────┐                        ┌─────────────┐
│ Original│           │  Transcode  │                        │   Encoded   │
│  Video  │──────────>│   Workers   │───────────────────────>│   Videos    │
│ (1080p) │           │             │                        │             │
└─────────┘           └─────────────┘                        └─────────────┘
                            │                                      │
                            │                                      │
              ┌─────────────┼─────────────┐                       │
              ▼             ▼             ▼                       │
        ┌─────────┐   ┌─────────┐   ┌─────────┐                  │
        │  240p   │   │  480p   │   │  720p   │                  │
        │  H.264  │   │  H.264  │   │  H.264  │──────────────────┘
        └─────────┘   └─────────┘   └─────────┘
              │             │             │
              │             │             │
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │  1080p  │   │  1440p  │   │   4K    │
        │  H.264  │   │  H.264  │   │  H.264  │
        └─────────┘   └─────────┘   └─────────┘

Also generates:
- Thumbnails (multiple timestamps)
- Preview sprites
- HLS/DASH manifests
- Subtitles (auto-generated)
```

---

## Streaming Protocols

### HLS (HTTP Live Streaming)

```
┌─────────────────────────────────────────────────────────────────┐
│                    HLS STRUCTURE                                 │
└─────────────────────────────────────────────────────────────────┘

Master Playlist (master.m3u8):
┌─────────────────────────────────────────────────────────────────┐
│ #EXTM3U                                                         │
│ #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360           │
│ 360p/playlist.m3u8                                              │
│ #EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480          │
│ 480p/playlist.m3u8                                              │
│ #EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720         │
│ 720p/playlist.m3u8                                              │
│ #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080        │
│ 1080p/playlist.m3u8                                             │
└─────────────────────────────────────────────────────────────────┘

Media Playlist (720p/playlist.m3u8):
┌─────────────────────────────────────────────────────────────────┐
│ #EXTM3U                                                         │
│ #EXT-X-VERSION:3                                                │
│ #EXT-X-TARGETDURATION:10                                        │
│ #EXTINF:10.0,                                                   │
│ segment_0.ts                                                    │
│ #EXTINF:10.0,                                                   │
│ segment_1.ts                                                    │
│ #EXTINF:10.0,                                                   │
│ segment_2.ts                                                    │
│ #EXT-X-ENDLIST                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect              | Decision                                              |
| ------------------- | ----------------------------------------------------- |
| Upload              | Chunked upload with resume support                    |
| Processing          | Distributed transcoding to multiple resolutions       |
| Storage             | Object storage with tiered access                     |
| Delivery            | Multi-tier CDN with edge caching                      |
| Streaming protocol  | HLS/DASH with adaptive bitrate                        |
| Scale               | 2B users, 500M videos, 1M concurrent streams          |

