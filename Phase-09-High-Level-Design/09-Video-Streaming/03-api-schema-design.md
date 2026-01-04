# Video Streaming - API & Schema Design

## API Design Philosophy

Video streaming APIs prioritize:

1. **Efficiency**: Large file handling with chunked uploads
2. **Resumability**: Support interrupted uploads/playback
3. **Scalability**: CDN-friendly URLs and caching
4. **Flexibility**: Multiple quality levels and formats

---

## Base URL Structure

```
REST API:     https://api.videoplatform.com/v1
Upload API:   https://upload.videoplatform.com/v1
CDN:          https://cdn.videoplatform.com
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths

**Deprecation Policy:**
1. Announce deprecation 6 months in advance
2. Return Deprecation header
3. Maintain old version for 12 months after new version release

---

## Authentication

### JWT Token Authentication

```http
POST /v1/videos/upload
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Tier | Requests/minute | Upload Size/hour |
|------|-----------------|------------------|
| Free | 60 | 1 GB |
| Pro | 1000 | 100 GB |
| Enterprise | 10000 | Unlimited |

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",  // Optional
      "reason": "Specific reason"  // Optional
    },
    "request_id": "req_123456"  // For tracing
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | FILE_TOO_LARGE | Video file exceeds size limit |
| 400 | INVALID_FORMAT | Video format not supported |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Video not found |
| 409 | CONFLICT | Upload session conflict |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - File too large
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "Video file exceeds size limit",
    "details": {
      "field": "file_info.size_bytes",
      "reason": "File size 2GB exceeds limit of 1GB"
    },
    "request_id": "req_abc123"
  }
}

// 409 Conflict - Upload session exists
{
  "error": {
    "code": "CONFLICT",
    "message": "Upload session already exists",
    "details": {
      "field": "video_id",
      "reason": "Video 'vid_abc123' already has an active upload session"
    },
    "request_id": "req_xyz789"
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 60,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_def456"
  }
}
```

---

## Idempotency Implementation

**Idempotency-Key Header:**

Clients must include `Idempotency-Key` header for all write operations:
```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends request with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/videos/upload | Yes | Idempotency-Key header |
| PUT /v1/videos/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/videos/{id} | Yes | Safe to retry (idempotent by design) |
| POST /v1/videos/{id}/comments | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/videos/upload")
public ResponseEntity<UploadResponse> initiateUpload(
        @RequestBody InitiateUploadRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            UploadResponse response = objectMapper.readValue(cachedResponse, UploadResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create upload session
    UploadSession session = uploadService.createUploadSession(request, idempotencyKey);
    UploadResponse response = UploadResponse.from(session);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(201).body(response);
}
```

---

## Core API Endpoints

### 1. Initiate Video Upload

**Endpoint:** `POST /v1/videos/upload`

**Purpose:** Start a new video upload session

**Request:**

```http
POST /v1/videos/upload HTTP/1.1
Host: api.videoplatform.com
Authorization: Bearer creator_token
Content-Type: application/json

{
  "title": "My Awesome Video",
  "description": "This is a great video about...",
  "tags": ["tutorial", "tech", "coding"],
  "category": "education",
  "privacy": "public",
  "file_info": {
    "filename": "video.mp4",
    "size_bytes": 524288000,
    "content_type": "video/mp4",
    "duration_seconds": 600
  }
}
```

**Response (201 Created):**

```json
{
  "video_id": "vid_abc123",
  "upload_url": "https://upload.videoplatform.com/v1/upload/vid_abc123",
  "upload_token": "upload_token_xyz",
  "chunk_size": 10485760,
  "expires_at": "2024-01-20T16:30:00Z"
}
```

---

### 2. Upload Video Chunk

**Endpoint:** `PUT /v1/upload/{video_id}`

**Purpose:** Upload a chunk of video data

**Request:**

```http
PUT /v1/upload/vid_abc123 HTTP/1.1
Host: upload.videoplatform.com
Authorization: Bearer upload_token_xyz
Content-Type: application/octet-stream
Content-Range: bytes 0-10485759/524288000
Content-Length: 10485760

[binary data]
```

**Response (200 OK):**

```json
{
  "video_id": "vid_abc123",
  "bytes_received": 10485760,
  "bytes_total": 524288000,
  "progress_percent": 2,
  "status": "uploading"
}
```

**Final Chunk Response:**

```json
{
  "video_id": "vid_abc123",
  "bytes_received": 524288000,
  "bytes_total": 524288000,
  "progress_percent": 100,
  "status": "processing",
  "estimated_processing_time": "30 minutes"
}
```

---

### 3. Get Video Metadata

**Endpoint:** `GET /v1/videos/{video_id}`

**Request:**

```http
GET /v1/videos/vid_abc123 HTTP/1.1
Host: api.videoplatform.com
```

**Response:**

```json
{
  "video_id": "vid_abc123",
  "title": "My Awesome Video",
  "description": "This is a great video about...",
  "channel": {
    "id": "ch_xyz",
    "name": "Tech Channel",
    "avatar_url": "https://cdn.videoplatform.com/avatars/ch_xyz.jpg",
    "subscriber_count": 1500000
  },
  "duration_seconds": 600,
  "view_count": 125000,
  "like_count": 8500,
  "dislike_count": 120,
  "comment_count": 450,
  "published_at": "2024-01-15T10:00:00Z",
  "thumbnails": {
    "default": "https://cdn.videoplatform.com/thumbs/vid_abc123/default.jpg",
    "medium": "https://cdn.videoplatform.com/thumbs/vid_abc123/medium.jpg",
    "high": "https://cdn.videoplatform.com/thumbs/vid_abc123/high.jpg"
  },
  "tags": ["tutorial", "tech", "coding"],
  "category": "education",
  "privacy": "public",
  "status": "ready"
}
```

---

### 4. Get Video Stream URL

**Endpoint:** `GET /v1/videos/{video_id}/stream`

**Purpose:** Get streaming manifest URL

**Request:**

```http
GET /v1/videos/vid_abc123/stream HTTP/1.1
Host: api.videoplatform.com
Authorization: Bearer viewer_token
X-Device-Type: mobile
X-Network-Quality: high
```

**Response:**

```json
{
  "video_id": "vid_abc123",
  "stream_url": "https://cdn.videoplatform.com/streams/vid_abc123/master.m3u8",
  "format": "hls",
  "available_qualities": [
    {"resolution": "240p", "bitrate": 400000},
    {"resolution": "360p", "bitrate": 800000},
    {"resolution": "480p", "bitrate": 1500000},
    {"resolution": "720p", "bitrate": 3000000},
    {"resolution": "1080p", "bitrate": 6000000}
  ],
  "subtitles": [
    {"language": "en", "url": "https://cdn.videoplatform.com/subs/vid_abc123/en.vtt"},
    {"language": "es", "url": "https://cdn.videoplatform.com/subs/vid_abc123/es.vtt"}
  ],
  "expires_at": "2024-01-20T17:30:00Z"
}
```

---

### 5. Update Watch Progress

**Endpoint:** `POST /v1/videos/{video_id}/progress`

**Purpose:** Save watch position for resume

**Request:**

```http
POST /v1/videos/vid_abc123/progress HTTP/1.1
Host: api.videoplatform.com
Authorization: Bearer viewer_token
Content-Type: application/json

{
  "position_seconds": 245,
  "duration_seconds": 600,
  "completed": false
}
```

**Response:**

```json
{
  "video_id": "vid_abc123",
  "position_seconds": 245,
  "saved_at": "2024-01-20T15:30:00Z"
}
```

---

### 6. Record View

**Endpoint:** `POST /v1/videos/{video_id}/view`

**Purpose:** Record a video view for analytics

**Request:**

```http
POST /v1/videos/vid_abc123/view HTTP/1.1
Host: api.videoplatform.com
Content-Type: application/json

{
  "viewer_id": "user_123",
  "session_id": "sess_abc",
  "watch_duration_seconds": 180,
  "quality_played": "720p",
  "device_type": "mobile",
  "referrer": "search"
}
```

**Response (202 Accepted):**

```json
{
  "status": "recorded"
}
```

---

### 7. Search Videos

**Endpoint:** `GET /v1/videos/search`

**Request:**

```http
GET /v1/videos/search?q=python+tutorial&sort=relevance&limit=20 HTTP/1.1
Host: api.videoplatform.com
```

**Response:**

```json
{
  "results": [
    {
      "video_id": "vid_xyz",
      "title": "Python Tutorial for Beginners",
      "channel": {"id": "ch_abc", "name": "Code Academy"},
      "duration_seconds": 3600,
      "view_count": 5000000,
      "published_at": "2023-06-15T10:00:00Z",
      "thumbnail_url": "https://cdn.videoplatform.com/thumbs/vid_xyz/medium.jpg"
    }
  ],
  "total_results": 15420,
  "next_page_token": "token_xyz"
}
```

---

## Database Schema Design

### Database Choices

| Data Type           | Database       | Rationale                                |
| ------------------- | -------------- | ---------------------------------------- |
| Video metadata      | PostgreSQL     | ACID, complex queries                    |
| Video files         | Object Storage | Blob storage (S3/GCS)                    |
| View counts         | Redis + Kafka  | High write volume, eventual consistency  |
| Watch history       | Cassandra      | Time-series, user-partitioned            |
| Search index        | Elasticsearch  | Full-text search                         |
| Recommendations     | Redis          | Fast access to precomputed recs          |

---

### Videos Table (PostgreSQL)

```sql
CREATE TABLE videos (
    id BIGSERIAL PRIMARY KEY,
    video_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Ownership
    channel_id BIGINT NOT NULL REFERENCES channels(id),
    
    -- Content
    title VARCHAR(500) NOT NULL,
    description TEXT,
    tags TEXT[],
    category VARCHAR(100),
    
    -- Technical
    duration_seconds INTEGER NOT NULL,
    original_filename VARCHAR(255),
    original_size_bytes BIGINT,
    
    -- Status
    status VARCHAR(20) DEFAULT 'uploading',
    processing_progress INTEGER DEFAULT 0,
    
    -- Privacy
    privacy VARCHAR(20) DEFAULT 'public',
    
    -- Timestamps
    uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    published_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Denormalized counts (updated async)
    view_count BIGINT DEFAULT 0,
    like_count INTEGER DEFAULT 0,
    dislike_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    
    CONSTRAINT valid_status CHECK (status IN ('uploading', 'processing', 'ready', 'failed', 'deleted')),
    CONSTRAINT valid_privacy CHECK (privacy IN ('public', 'unlisted', 'private'))
);

CREATE INDEX idx_videos_channel ON videos(channel_id, published_at DESC);
CREATE INDEX idx_videos_status ON videos(status) WHERE status != 'ready';
CREATE INDEX idx_videos_published ON videos(published_at DESC) WHERE privacy = 'public';
CREATE INDEX idx_videos_views ON videos(view_count DESC) WHERE privacy = 'public';
```

---

### Video Encodings Table

```sql
CREATE TABLE video_encodings (
    id BIGSERIAL PRIMARY KEY,
    video_id BIGINT NOT NULL REFERENCES videos(id),
    
    -- Encoding details
    resolution VARCHAR(20) NOT NULL,  -- '240p', '720p', '1080p', '4k'
    codec VARCHAR(20) NOT NULL,       -- 'h264', 'h265', 'vp9', 'av1'
    container VARCHAR(20) NOT NULL,   -- 'mp4', 'webm', 'ts'
    bitrate INTEGER NOT NULL,         -- bits per second
    
    -- Storage
    storage_path TEXT NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    
    -- HLS/DASH
    manifest_path TEXT,
    segment_duration INTEGER DEFAULT 10,
    
    -- Status
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(video_id, resolution, codec)
);

CREATE INDEX idx_encodings_video ON video_encodings(video_id);
```

---

### Channels Table

```sql
CREATE TABLE channels (
    id BIGSERIAL PRIMARY KEY,
    channel_id VARCHAR(50) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id),
    
    -- Profile
    name VARCHAR(100) NOT NULL,
    description TEXT,
    avatar_url TEXT,
    banner_url TEXT,
    
    -- Stats (denormalized)
    subscriber_count BIGINT DEFAULT 0,
    video_count INTEGER DEFAULT 0,
    total_views BIGINT DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_channels_user ON channels(user_id);
CREATE INDEX idx_channels_subscribers ON channels(subscriber_count DESC);
```

---

### Watch History (Cassandra)

```sql
-- Partition by user_id for efficient user history queries
CREATE TABLE watch_history (
    user_id TEXT,
    watched_at TIMESTAMP,
    video_id TEXT,
    watch_duration_seconds INT,
    completed BOOLEAN,
    last_position_seconds INT,
    PRIMARY KEY (user_id, watched_at, video_id)
) WITH CLUSTERING ORDER BY (watched_at DESC, video_id ASC);

-- For "continue watching" feature
CREATE TABLE watch_progress (
    user_id TEXT,
    video_id TEXT,
    position_seconds INT,
    duration_seconds INT,
    updated_at TIMESTAMP,
    PRIMARY KEY (user_id, video_id)
);
```

---

### View Counts (Redis + PostgreSQL)

```
# Real-time view count in Redis
INCR views:vid_abc123

# Hourly aggregation to PostgreSQL
# Cron job: Every hour, flush Redis to PostgreSQL
```

```sql
-- Hourly view aggregates
CREATE TABLE view_aggregates (
    video_id BIGINT NOT NULL REFERENCES videos(id),
    hour_bucket TIMESTAMP WITH TIME ZONE NOT NULL,
    view_count INTEGER NOT NULL,
    unique_viewers INTEGER,
    avg_watch_duration INTEGER,
    PRIMARY KEY (video_id, hour_bucket)
);
```

---

### Processing Queue (PostgreSQL)

```sql
CREATE TABLE processing_jobs (
    id BIGSERIAL PRIMARY KEY,
    video_id BIGINT NOT NULL REFERENCES videos(id),
    job_type VARCHAR(50) NOT NULL,  -- 'transcode', 'thumbnail', 'manifest'
    
    -- Job details
    input_path TEXT NOT NULL,
    output_path TEXT,
    parameters JSONB,
    
    -- Status
    status VARCHAR(20) DEFAULT 'pending',
    progress INTEGER DEFAULT 0,
    error_message TEXT,
    
    -- Worker
    worker_id VARCHAR(100),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    
    -- Scheduling
    priority INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_job_status CHECK (status IN ('pending', 'processing', 'completed', 'failed'))
);

CREATE INDEX idx_jobs_pending ON processing_jobs(priority DESC, created_at ASC) 
    WHERE status = 'pending';
CREATE INDEX idx_jobs_video ON processing_jobs(video_id);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │      channels       │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id             │───────│ user_id (FK)        │
│ email               │       │ name                │
│ username            │       │ subscriber_count    │
└─────────────────────┘       └─────────────────────┘
                                       │
                                       │
                                       ▼
                              ┌─────────────────────┐
                              │       videos        │
                              ├─────────────────────┤
                              │ id (PK)             │
                              │ video_id            │
                              │ channel_id (FK)     │
                              │ title               │
                              │ duration_seconds    │
                              │ view_count          │
                              │ status              │
                              └─────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
┌─────────────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│    video_encodings      │ │  video_thumbs   │ │  processing_jobs    │
├─────────────────────────┤ ├─────────────────┤ ├─────────────────────┤
│ id (PK)                 │ │ id (PK)         │ │ id (PK)             │
│ video_id (FK)           │ │ video_id (FK)   │ │ video_id (FK)       │
│ resolution              │ │ timestamp_sec   │ │ job_type            │
│ codec                   │ │ url             │ │ status              │
│ storage_path            │ │ size            │ │ progress            │
└─────────────────────────┘ └─────────────────┘ └─────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    CASSANDRA (Time-Series)                          │
├─────────────────────────────────────────────────────────────────────┤
│ watch_history: (user_id, watched_at, video_id) → watch details     │
│ watch_progress: (user_id, video_id) → position, duration           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    OBJECT STORAGE (S3/GCS)                          │
├─────────────────────────────────────────────────────────────────────┤
│ /originals/{video_id}/original.mp4                                 │
│ /encoded/{video_id}/720p/segment_0.ts                              │
│ /encoded/{video_id}/720p/playlist.m3u8                             │
│ /thumbnails/{video_id}/thumb_0.jpg                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CDN URL Structure

```
# Master manifest
https://cdn.videoplatform.com/v/{video_id}/master.m3u8

# Quality-specific manifest
https://cdn.videoplatform.com/v/{video_id}/720p/playlist.m3u8

# Video segments
https://cdn.videoplatform.com/v/{video_id}/720p/segment_0.ts
https://cdn.videoplatform.com/v/{video_id}/720p/segment_1.ts

# Thumbnails
https://cdn.videoplatform.com/t/{video_id}/default.jpg
https://cdn.videoplatform.com/t/{video_id}/sprite.jpg

# Subtitles
https://cdn.videoplatform.com/s/{video_id}/en.vtt
```

---

## Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for video uploads, transcoding jobs, and view tracking to prevent duplicates.

### Idempotent Operations in Video Streaming

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Initiate Upload** | ✅ Yes | Idempotency key prevents duplicate upload sessions |
| **Upload Chunk** | ✅ Yes | Content-Range ensures chunks are idempotent |
| **Complete Upload** | ✅ Yes | Idempotency key prevents duplicate processing |
| **Transcoding Job** | ✅ Yes | Job ID prevents duplicate transcoding |
| **Record View** | ⚠️ At-least-once | Deduplication by (video_id, user_id, timestamp) |

### Idempotency Implementation

**1. Upload Initiation with Idempotency Key:**

```java
@PostMapping("/v1/videos/upload")
public ResponseEntity<UploadResponse> initiateUpload(
        @RequestBody InitiateUploadRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // Check if upload session already exists
    UploadSession existing = uploadSessionRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(UploadResponse.from(existing));
    }
    
    // Create new upload session
    String videoId = generateVideoId();
    UploadSession session = UploadSession.builder()
        .videoId(videoId)
        .idempotencyKey(idempotencyKey)
        .userId(request.getUserId())
        .fileInfo(request.getFileInfo())
        .status(UploadStatus.INITIATED)
        .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
        .build();
    
    session = uploadSessionRepository.save(session);
    
    return ResponseEntity.status(201).body(UploadResponse.from(session));
}
```

**2. Chunk Upload Idempotency:**

```java
@PutMapping("/v1/upload/{videoId}")
public ResponseEntity<ChunkResponse> uploadChunk(
        @PathVariable String videoId,
        @RequestHeader("Content-Range") String contentRange,
        @RequestBody byte[] chunkData) {
    
    // Parse Content-Range: "bytes 0-10485759/524288000"
    Range range = parseContentRange(contentRange);
    
    // Check if chunk already uploaded
    ChunkStatus existing = chunkRepository.findByVideoIdAndRange(
        videoId, range.getStart(), range.getEnd()
    );
    
    if (existing != null && existing.getStatus() == ChunkStatus.COMPLETED) {
        // Chunk already uploaded, return existing status
        return ResponseEntity.ok(ChunkResponse.from(existing));
    }
    
    // Upload chunk to S3 (idempotent: same key = overwrite)
    String chunkKey = String.format("uploads/%s/chunk_%d_%d", 
        videoId, range.getStart(), range.getEnd());
    s3Service.putObject(chunkKey, chunkData);
    
    // Record chunk completion
    ChunkStatus chunkStatus = ChunkStatus.builder()
        .videoId(videoId)
        .startByte(range.getStart())
        .endByte(range.getEnd())
        .status(ChunkStatus.COMPLETED)
        .uploadedAt(Instant.now())
        .build();
    
    chunkRepository.save(chunkStatus);
    
    return ResponseEntity.ok(ChunkResponse.from(chunkStatus));
}
```

**3. Transcoding Job Idempotency:**

```java
// Transcoding jobs are idempotent via job ID
public void processTranscodingJob(TranscodingJob job) {
    // Check if job already processed
    VideoProcessingStatus existing = processingRepository.findByJobId(job.getJobId());
    if (existing != null && existing.getStatus() == ProcessingStatus.COMPLETED) {
        // Job already processed, skip
        return;
    }
    
    // Update status to PROCESSING (idempotent)
    processingRepository.updateStatus(job.getJobId(), ProcessingStatus.PROCESSING);
    
    try {
        // Process transcoding
        TranscodingResult result = transcoder.transcode(job);
        
        // Upload segments to S3 (idempotent: same key = overwrite)
        for (Segment segment : result.getSegments()) {
            s3Service.putObject(segment.getS3Key(), segment.getData());
        }
        
        // Update status to COMPLETED
        processingRepository.updateStatus(job.getJobId(), ProcessingStatus.COMPLETED);
        
    } catch (Exception e) {
        processingRepository.updateStatus(job.getJobId(), ProcessingStatus.FAILED);
        throw e;
    }
}
```

**4. View Count Deduplication:**

```java
// Deduplicate view counts by (video_id, user_id, timestamp)
public void recordView(String videoId, String userId) {
    long timestamp = System.currentTimeMillis() / 1000; // Round to second
    String dedupKey = String.format("view:%s:%s:%d", videoId, userId, timestamp);
    
    // Set if absent (idempotent)
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", Duration.ofMinutes(5));
    
    if (Boolean.TRUE.equals(isNew)) {
        // New view, increment counter
        redisTemplate.opsForValue().increment("views:" + videoId);
        // Async flush to database
        kafkaTemplate.send("view-events", ViewEvent.builder()
            .videoId(videoId)
            .userId(userId)
            .timestamp(Instant.ofEpochSecond(timestamp))
            .build());
    }
    // Duplicate view, ignore
}
```

### Idempotency Key Generation

**Client-Side (Recommended):**
```javascript
// Generate UUID v4 for each upload session
const idempotencyKey = crypto.randomUUID();

fetch('/v1/videos/upload', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'Idempotency-Key': idempotencyKey
    },
    body: uploadData
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide key, generate from request hash
String idempotencyKey = DigestUtils.sha256Hex(
    request.getUserId() + 
    request.getFileInfo().getFilename() +
    request.getFileInfo().getSizeBytes() +
    Instant.now().toString()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Upload Initiation | 24 hours | PostgreSQL (idempotency_key unique index) |
| Chunk Upload | Forever | S3 (same key = overwrite) |
| Transcoding Jobs | Forever | PostgreSQL (job_id unique) |
| View Counts | 5 minutes | Redis |
| Watch History | 1 hour | Cassandra (deduplication on write) |

### Idempotency Key Storage

```sql
-- Add idempotency_key to upload_sessions
ALTER TABLE upload_sessions ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_upload_sessions_idempotency_key 
ON upload_sessions(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Transcoding jobs already have job_id as unique identifier
CREATE UNIQUE INDEX idx_transcoding_jobs_job_id ON transcoding_jobs(job_id);
```

---

## Summary

| Component           | Technology/Approach                     |
| ------------------- | --------------------------------------- |
| Video metadata      | PostgreSQL                              |
| Video storage       | Object Storage (S3/GCS)                 |
| Watch history       | Cassandra                               |
| View counts         | Redis (real-time) + PostgreSQL (batch)  |
| Search              | Elasticsearch                           |
| Streaming           | HLS/DASH via CDN                        |
| Upload              | Chunked, resumable                      |

