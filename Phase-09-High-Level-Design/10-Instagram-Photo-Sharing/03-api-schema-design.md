# Instagram / Photo Sharing - API & Schema Design

## API Design Philosophy

Photo sharing APIs prioritize:

1. **Mobile-first**: Optimize for mobile bandwidth and battery
2. **Fast Loading**: Progressive image loading with placeholders
3. **Efficient Pagination**: Cursor-based for infinite scroll
4. **Real-time Updates**: WebSocket for notifications

---

## Base URL Structure

```
REST API:     https://api.instagram.com/v1
WebSocket:    wss://ws.instagram.com/v1
CDN:          https://cdn.instagram.com
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
POST /v1/posts
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

| Tier | Requests/minute |
|------|-----------------|
| Free | 60 |
| Pro | 1000 |
| Enterprise | 10000 |

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
| 400 | IMAGE_TOO_LARGE | Image file exceeds size limit |
| 400 | INVALID_FORMAT | Image format not supported |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Post or user not found |
| 409 | CONFLICT | Resource conflict |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Image too large
{
  "error": {
    "code": "IMAGE_TOO_LARGE",
    "message": "Image file exceeds size limit",
    "details": {
      "field": "image",
      "reason": "Image size 20MB exceeds limit of 10MB"
    },
    "request_id": "req_abc123"
  }
}

// 404 Not Found - Post not found
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Post not found",
    "details": {
      "field": "post_id",
      "reason": "Post 'post_abc123' does not exist"
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
| POST /v1/posts | Yes | Idempotency-Key header |
| PUT /v1/posts/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/posts/{id} | Yes | Safe to retry (idempotent by design) |
| POST /v1/posts/{id}/like | Yes | Idempotency-Key header (dedupe by user+post) |
| POST /v1/posts/{id}/comment | Yes | Idempotency-Key header |

**Implementation Example:**

```java
@PostMapping("/v1/posts")
public ResponseEntity<PostResponse> createPost(
        @RequestBody CreatePostRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            PostResponse response = objectMapper.readValue(cachedResponse, PostResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create post
    Post post = postService.createPost(request, idempotencyKey);
    PostResponse response = PostResponse.from(post);
    
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

### 1. Upload Photo

**Endpoint:** `POST /v1/posts`

**Purpose:** Upload a new photo post

**Request:**

```http
POST /v1/posts HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
Content-Type: multipart/form-data

{
  "image": <binary>,
  "caption": "Beautiful sunset! 🌅 #sunset #photography",
  "location": {
    "name": "Santa Monica Beach",
    "latitude": 34.0195,
    "longitude": -118.4912
  },
  "tagged_users": ["user_123", "user_456"],
  "filter": "clarendon"
}
```

**Response (201 Created):**

```json
{
  "post_id": "post_abc123",
  "user": {
    "user_id": "user_me",
    "username": "photographer",
    "profile_pic_url": "https://cdn.instagram.com/avatars/user_me.jpg"
  },
  "images": {
    "thumbnail": "https://cdn.instagram.com/p/abc123/thumb.jpg",
    "medium": "https://cdn.instagram.com/p/abc123/medium.jpg",
    "large": "https://cdn.instagram.com/p/abc123/large.jpg"
  },
  "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj",
  "caption": "Beautiful sunset! 🌅 #sunset #photography",
  "location": {
    "name": "Santa Monica Beach"
  },
  "like_count": 0,
  "comment_count": 0,
  "created_at": "2024-01-20T15:30:00Z"
}
```

---

### 2. Get Feed

**Endpoint:** `GET /v1/feed`

**Purpose:** Get personalized home feed

**Request:**

```http
GET /v1/feed?cursor=abc123&limit=20 HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "posts": [
    {
      "post_id": "post_xyz",
      "user": {
        "user_id": "user_friend",
        "username": "friend",
        "profile_pic_url": "https://cdn.instagram.com/avatars/friend.jpg",
        "is_verified": false
      },
      "images": {
        "thumbnail": "https://cdn.instagram.com/p/xyz/thumb.jpg",
        "medium": "https://cdn.instagram.com/p/xyz/medium.jpg",
        "large": "https://cdn.instagram.com/p/xyz/large.jpg"
      },
      "blurhash": "LGF5]+Yk^6#M@-5c,1J5@[or[Q6.",
      "aspect_ratio": 1.0,
      "caption": "Amazing day at the park!",
      "location": {"name": "Central Park"},
      "like_count": 1542,
      "comment_count": 89,
      "is_liked": false,
      "is_saved": false,
      "created_at": "2024-01-20T14:00:00Z"
    }
  ],
  "next_cursor": "cursor_xyz",
  "has_more": true
}
```

---

### 3. Get Stories

**Endpoint:** `GET /v1/stories`

**Purpose:** Get stories from followed users

**Request:**

```http
GET /v1/stories HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "story_trays": [
    {
      "user": {
        "user_id": "user_friend",
        "username": "friend",
        "profile_pic_url": "https://cdn.instagram.com/avatars/friend.jpg"
      },
      "has_unseen": true,
      "latest_story_at": "2024-01-20T15:00:00Z",
      "stories": [
        {
          "story_id": "story_123",
          "media_type": "image",
          "media_url": "https://cdn.instagram.com/stories/story_123.jpg",
          "thumbnail_url": "https://cdn.instagram.com/stories/story_123_thumb.jpg",
          "duration_seconds": 5,
          "created_at": "2024-01-20T15:00:00Z",
          "expires_at": "2024-01-21T15:00:00Z",
          "is_seen": false
        }
      ]
    }
  ]
}
```

---

### 4. Like/Unlike Post

**Endpoint:** `POST /v1/posts/{post_id}/like`

**Request:**

```http
POST /v1/posts/post_xyz/like HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "post_id": "post_xyz",
  "is_liked": true,
  "like_count": 1543
}
```

**Unlike:** `DELETE /v1/posts/{post_id}/like`

---

### 5. Follow/Unfollow User

**Endpoint:** `POST /v1/users/{user_id}/follow`

**Request:**

```http
POST /v1/users/user_friend/follow HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "user_id": "user_friend",
  "is_following": true,
  "follower_count": 12346
}
```

---

### 6. Search

**Endpoint:** `GET /v1/search`

**Request:**

```http
GET /v1/search?q=sunset&type=hashtag&limit=20 HTTP/1.1
Host: api.instagram.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "results": [
    {
      "type": "hashtag",
      "hashtag": {
        "name": "sunset",
        "post_count": 125000000
      }
    },
    {
      "type": "user",
      "user": {
        "user_id": "user_sunset",
        "username": "sunset_photos",
        "profile_pic_url": "...",
        "is_verified": true
      }
    }
  ],
  "next_cursor": "cursor_abc"
}
```

---

### 7. Get User Profile

**Endpoint:** `GET /v1/users/{user_id}`

**Response:**

```json
{
  "user_id": "user_123",
  "username": "photographer",
  "full_name": "John Doe",
  "bio": "📸 Photography enthusiast",
  "profile_pic_url": "https://cdn.instagram.com/avatars/user_123.jpg",
  "post_count": 245,
  "follower_count": 15000,
  "following_count": 500,
  "is_following": false,
  "is_private": false,
  "is_verified": false
}
```

---

## Database Schema Design

### Database Choices

| Data Type           | Database       | Rationale                                |
| ------------------- | -------------- | ---------------------------------------- |
| Users/Posts         | PostgreSQL     | ACID, complex queries                    |
| Feed cache          | Redis          | Fast reads, sorted sets                  |
| Social graph        | PostgreSQL + Redis | Persistent + cached                   |
| Stories             | Cassandra      | TTL support, time-series                 |
| Images              | S3/Object Store| Blob storage                             |
| Search              | Elasticsearch  | Full-text search                         |

---

### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Profile
    username VARCHAR(30) UNIQUE NOT NULL,
    full_name VARCHAR(100),
    bio VARCHAR(150),
    profile_pic_url TEXT,
    website VARCHAR(200),
    
    -- Privacy
    is_private BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    
    -- Stats (denormalized)
    post_count INTEGER DEFAULT 0,
    follower_count BIGINT DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_followers ON users(follower_count DESC);
```

---

### Posts Table

```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    post_id VARCHAR(50) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id),
    
    -- Content
    caption TEXT,
    
    -- Images (JSON array of URLs)
    image_urls JSONB NOT NULL,
    blurhash VARCHAR(50),
    aspect_ratio DECIMAL(4, 2) DEFAULT 1.0,
    
    -- Location
    location_name VARCHAR(200),
    location_lat DECIMAL(10, 8),
    location_lng DECIMAL(11, 8),
    
    -- Stats (denormalized)
    like_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    
    -- Status
    is_archived BOOLEAN DEFAULT FALSE,
    is_comments_disabled BOOLEAN DEFAULT FALSE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_posts_user ON posts(user_id, created_at DESC);
CREATE INDEX idx_posts_created ON posts(created_at DESC);
CREATE INDEX idx_posts_location ON posts(location_lat, location_lng) 
    WHERE location_lat IS NOT NULL;
```

---

### Follows Table

```sql
CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT NOT NULL REFERENCES users(id),
    following_id BIGINT NOT NULL REFERENCES users(id),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(follower_id, following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);
```

---

### Likes Table

```sql
CREATE TABLE likes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    post_id BIGINT NOT NULL REFERENCES posts(id),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(user_id, post_id)
);

CREATE INDEX idx_likes_post ON likes(post_id);
CREATE INDEX idx_likes_user ON likes(user_id, created_at DESC);
```

---

### Comments Table

```sql
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    comment_id VARCHAR(50) UNIQUE NOT NULL,
    post_id BIGINT NOT NULL REFERENCES posts(id),
    user_id BIGINT NOT NULL REFERENCES users(id),
    parent_id BIGINT REFERENCES comments(id),  -- For replies
    
    text VARCHAR(2200) NOT NULL,
    
    like_count INTEGER DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON comments(post_id, created_at DESC);
CREATE INDEX idx_comments_user ON comments(user_id);
```

---

### Stories Table (Cassandra)

```sql
CREATE TABLE stories (
    user_id TEXT,
    story_id TIMEUUID,
    media_type TEXT,
    media_url TEXT,
    thumbnail_url TEXT,
    duration_seconds INT,
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
    view_count COUNTER,
    PRIMARY KEY (user_id, story_id)
) WITH CLUSTERING ORDER BY (story_id DESC)
  AND default_time_to_live = 86400;  -- 24 hours TTL

-- Story views
CREATE TABLE story_views (
    story_id TEXT,
    viewer_id TEXT,
    viewed_at TIMESTAMP,
    PRIMARY KEY (story_id, viewer_id)
) WITH default_time_to_live = 86400;
```

---

### Feed Cache (Redis)

```
# User's precomputed feed (sorted set)
# Score = timestamp (for chronological) or ranking score
ZADD feed:{user_id} <score> <post_id>

# Example
ZADD feed:user_123 1705764600 post_abc
ZADD feed:user_123 1705764500 post_def

# Get feed (most recent first)
ZREVRANGE feed:user_123 0 19 WITHSCORES

# Feed TTL
EXPIRE feed:{user_id} 86400  # 24 hours

# User's story tray (sorted set)
ZADD stories:{user_id} <timestamp> <story_owner_id>
```

---

### Hashtags Table

```sql
CREATE TABLE hashtags (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    post_count BIGINT DEFAULT 0
);

CREATE TABLE post_hashtags (
    post_id BIGINT REFERENCES posts(id),
    hashtag_id BIGINT REFERENCES hashtags(id),
    PRIMARY KEY (post_id, hashtag_id)
);

CREATE INDEX idx_post_hashtags_hashtag ON post_hashtags(hashtag_id);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │       posts         │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id (unique)    │───────│ user_id (FK)        │
│ username            │       │ post_id             │
│ follower_count      │       │ image_urls          │
│ following_count     │       │ caption             │
│ post_count          │       │ like_count          │
└─────────────────────┘       └─────────────────────┘
         │                             │
         │                             │
         ▼                             ▼
┌─────────────────────┐       ┌─────────────────────┐
│      follows        │       │       likes         │
├─────────────────────┤       ├─────────────────────┤
│ follower_id (FK)    │       │ user_id (FK)        │
│ following_id (FK)   │       │ post_id (FK)        │
│ created_at          │       │ created_at          │
└─────────────────────┘       └─────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  comments                            │
├─────────────────────────────────────────────────────┤
│ id (PK)                                             │
│ post_id (FK)                                        │
│ user_id (FK)                                        │
│ parent_id (FK, self)                                │
│ text                                                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  CASSANDRA                           │
├─────────────────────────────────────────────────────┤
│ stories: (user_id, story_id) → story data           │
│ story_views: (story_id, viewer_id) → viewed_at      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  REDIS CACHE                         │
├─────────────────────────────────────────────────────┤
│ feed:{user_id}        → Sorted Set of post_ids      │
│ stories:{user_id}     → Sorted Set of story owners  │
│ user:{user_id}        → Hash of user data           │
└─────────────────────────────────────────────────────┘
```

---

## Image URL Structure

```
# Profile pictures
https://cdn.instagram.com/avatars/{user_id}/{size}.jpg
Sizes: 150, 320, 640

# Post images
https://cdn.instagram.com/p/{post_id}/{size}.jpg
Sizes: thumb (150), medium (640), large (1080)

# Stories
https://cdn.instagram.com/stories/{story_id}/{size}.jpg
Sizes: thumb, full
```

---

## Summary

| Component           | Technology/Approach                     |
| ------------------- | --------------------------------------- |
| User/Post data      | PostgreSQL                              |
| Feed cache          | Redis Sorted Sets                       |
| Stories             | Cassandra with TTL                      |
| Images              | S3 + CDN                                |
| Search              | Elasticsearch                           |
| Social graph        | PostgreSQL + Redis cache                |

