# News Feed - API & Schema Design

## API Design Philosophy

News feed APIs prioritize:

1. **Speed**: Feed must load instantly
2. **Pagination**: Infinite scroll support
3. **Real-time**: Push updates for new content
4. **Efficiency**: Minimize data transfer

---

## Core API Endpoints

### 1. Get Feed

**Endpoint:** `GET /v1/feed`

**Purpose:** Retrieve personalized news feed for authenticated user

**Request:**

```http
GET /v1/feed?cursor=abc123&limit=20 HTTP/1.1
Host: api.social.com
Authorization: Bearer user_token
Accept: application/json
X-Client-Version: 2.5.0
```

**Query Parameters:**

| Parameter   | Type    | Required | Default | Description                        |
| ----------- | ------- | -------- | ------- | ---------------------------------- |
| cursor      | string  | No       | null    | Pagination cursor                  |
| limit       | int     | No       | 20      | Posts per page (max 50)            |
| since_id    | string  | No       | null    | Posts newer than this ID           |

**Response (200 OK):**

```json
{
  "posts": [
    {
      "id": "post_abc123",
      "author": {
        "id": "user_xyz",
        "name": "John Doe",
        "avatar_url": "https://cdn.social.com/avatars/xyz.jpg",
        "is_verified": false
      },
      "content": {
        "text": "Just had an amazing pizza in NYC! 🍕",
        "media": [
          {
            "type": "image",
            "url": "https://cdn.social.com/media/img123.jpg",
            "thumbnail_url": "https://cdn.social.com/media/img123_thumb.jpg",
            "width": 1080,
            "height": 1080
          }
        ],
        "links": []
      },
      "engagement": {
        "likes_count": 1542,
        "comments_count": 89,
        "shares_count": 23,
        "is_liked": false,
        "is_shared": false
      },
      "created_at": "2024-01-20T15:30:00Z",
      "ranking_score": 0.95,
      "reason": "friend_post"
    },
    {
      "id": "post_def456",
      "author": {
        "id": "page_news",
        "name": "Tech News",
        "avatar_url": "https://cdn.social.com/avatars/news.jpg",
        "is_verified": true,
        "type": "page"
      },
      "content": {
        "text": "Breaking: New AI breakthrough announced...",
        "media": [],
        "links": [
          {
            "url": "https://technews.com/article/123",
            "title": "AI Breakthrough",
            "description": "Researchers announce...",
            "image_url": "https://technews.com/img.jpg"
          }
        ]
      },
      "engagement": {
        "likes_count": 15420,
        "comments_count": 892,
        "shares_count": 2341,
        "is_liked": true,
        "is_shared": false
      },
      "created_at": "2024-01-20T14:00:00Z",
      "ranking_score": 0.92,
      "reason": "followed_page"
    }
  ],
  "pagination": {
    "next_cursor": "cursor_xyz789",
    "has_more": true
  },
  "new_posts_count": 5,
  "last_refresh_time": "2024-01-20T15:35:00Z"
}
```

---

### 2. Create Post

**Endpoint:** `POST /v1/posts`

**Request:**

```http
POST /v1/posts HTTP/1.1
Host: api.social.com
Authorization: Bearer user_token
Content-Type: application/json

{
  "content": {
    "text": "Hello world! 👋",
    "media_ids": ["media_123", "media_456"],
    "link_url": "https://example.com/article"
  },
  "privacy": "friends",
  "location": {
    "latitude": 40.7128,
    "longitude": -74.0060,
    "name": "New York, NY"
  }
}
```

**Response (201 Created):**

```json
{
  "id": "post_new123",
  "author": {
    "id": "user_me",
    "name": "My Name"
  },
  "content": {
    "text": "Hello world! 👋",
    "media": [
      {
        "id": "media_123",
        "type": "image",
        "url": "https://cdn.social.com/media/123.jpg"
      }
    ]
  },
  "privacy": "friends",
  "created_at": "2024-01-20T16:00:00Z",
  "estimated_reach": 200
}
```

---

### 3. Get New Posts Count

**Endpoint:** `GET /v1/feed/updates`

**Purpose:** Check for new posts without fetching full feed

**Request:**

```http
GET /v1/feed/updates?since=2024-01-20T15:30:00Z HTTP/1.1
Host: api.social.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "new_posts_count": 12,
  "has_important_updates": true,
  "top_update_preview": {
    "author_name": "Close Friend",
    "text_preview": "Just got engaged! 💍"
  }
}
```

---

### 4. Like/Unlike Post

**Endpoint:** `POST /v1/posts/{post_id}/like`

**Request:**

```http
POST /v1/posts/post_abc123/like HTTP/1.1
Host: api.social.com
Authorization: Bearer user_token
```

**Response (200 OK):**

```json
{
  "post_id": "post_abc123",
  "is_liked": true,
  "likes_count": 1543
}
```

**Unlike:** `DELETE /v1/posts/{post_id}/like`

---

### 5. Follow/Unfollow User

**Endpoint:** `POST /v1/users/{user_id}/follow`

**Request:**

```http
POST /v1/users/user_xyz/follow HTTP/1.1
Host: api.social.com
Authorization: Bearer user_token
```

**Response:**

```json
{
  "user_id": "user_xyz",
  "is_following": true,
  "followers_count": 1234
}
```

---

### 6. WebSocket: Real-time Feed Updates

**Connection:**

```
wss://realtime.social.com/feed?token=user_token
```

**Server → Client Messages:**

```json
// New post from friend
{
  "type": "new_post",
  "post": {
    "id": "post_new",
    "author": {"id": "user_friend", "name": "Friend"},
    "text_preview": "Check this out!",
    "created_at": "2024-01-20T16:05:00Z"
  }
}

// Engagement update
{
  "type": "engagement_update",
  "post_id": "post_abc123",
  "likes_count": 1600,
  "comments_count": 95
}

// New posts available notification
{
  "type": "new_posts_available",
  "count": 5
}
```

---

## Database Schema Design

### Database Choices

| Data Type          | Database       | Rationale                                |
| ------------------ | -------------- | ---------------------------------------- |
| Posts              | PostgreSQL     | ACID, relational queries                 |
| Feed cache         | Redis          | Fast reads, TTL support                  |
| Social graph       | PostgreSQL + Redis | Persistent + cached                   |
| Engagement counts  | Redis          | High-frequency updates                   |
| Activity stream    | Cassandra      | Time-series, write-heavy                 |
| Media metadata     | PostgreSQL     | Relational to posts                      |

---

### Posts Table

```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    post_id VARCHAR(50) UNIQUE NOT NULL,  -- External ID
    user_id BIGINT NOT NULL REFERENCES users(id),
    
    -- Content
    content_text TEXT,
    content_json JSONB,  -- Rich content (mentions, hashtags)
    
    -- Privacy
    privacy VARCHAR(20) DEFAULT 'public',
    
    -- Location
    location_name VARCHAR(255),
    location_lat DECIMAL(10, 8),
    location_lng DECIMAL(11, 8),
    
    -- Metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    -- Denormalized counts (updated async)
    likes_count INTEGER DEFAULT 0,
    comments_count INTEGER DEFAULT 0,
    shares_count INTEGER DEFAULT 0,
    
    -- Ranking signals
    engagement_score FLOAT DEFAULT 0.0,
    
    CONSTRAINT valid_privacy CHECK (privacy IN ('public', 'friends', 'private'))
);

-- Index for user's posts
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC)
    WHERE deleted_at IS NULL;

-- Index for feed generation
CREATE INDEX idx_posts_created ON posts(created_at DESC)
    WHERE deleted_at IS NULL AND privacy != 'private';

-- Index for engagement-based queries
CREATE INDEX idx_posts_engagement ON posts(engagement_score DESC, created_at DESC)
    WHERE deleted_at IS NULL;
```

---

### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Profile
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    avatar_url TEXT,
    bio TEXT,
    
    -- Verification
    is_verified BOOLEAN DEFAULT FALSE,
    is_celebrity BOOLEAN DEFAULT FALSE,  -- > 10K followers
    
    -- Counts (denormalized)
    followers_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    posts_count INTEGER DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_active_at TIMESTAMP WITH TIME ZONE,
    
    -- Settings
    settings JSONB DEFAULT '{}'
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_celebrity ON users(is_celebrity) WHERE is_celebrity = TRUE;
```

---

### Follows Table (Social Graph)

```sql
CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT NOT NULL REFERENCES users(id),
    following_id BIGINT NOT NULL REFERENCES users(id),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Relationship strength (for ranking)
    interaction_score FLOAT DEFAULT 0.0,
    
    UNIQUE(follower_id, following_id)
);

-- Index for "who do I follow"
CREATE INDEX idx_follows_follower ON follows(follower_id);

-- Index for "who follows me"
CREATE INDEX idx_follows_following ON follows(following_id);

-- Index for fan-out (get all followers)
CREATE INDEX idx_follows_following_created ON follows(following_id, created_at DESC);
```

---

### Feed Cache (Redis)

```
# User's precomputed feed (sorted set)
# Score = ranking_score * 1000000 + timestamp
ZADD feed:{user_id} <score> <post_id>

# Example
ZADD feed:user_123 1705764600950000 post_abc123
ZADD feed:user_123 1705764500920000 post_def456

# Get feed (most recent first)
ZREVRANGE feed:user_123 0 19 WITHSCORES

# Feed TTL
EXPIRE feed:{user_id} 86400  # 24 hours
```

**Feed Entry Structure (if using Hash):**

```
HSET feed_entry:{post_id} 
    author_id "user_xyz"
    text_preview "Just had an amazing..."
    media_count 1
    likes_count 1542
    created_at 1705764600
```

---

### Engagement Table

```sql
CREATE TABLE post_likes (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL REFERENCES posts(id),
    user_id BIGINT NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(post_id, user_id)
);

CREATE INDEX idx_likes_post ON post_likes(post_id);
CREATE INDEX idx_likes_user ON post_likes(user_id, created_at DESC);

-- Similar tables for comments, shares
```

---

### Activity Stream (Cassandra)

```sql
-- For fan-out writes
CREATE TABLE user_feed (
    user_id BIGINT,
    created_at TIMESTAMP,
    post_id TEXT,
    author_id BIGINT,
    ranking_score FLOAT,
    PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC, post_id ASC);

-- For reading user's own posts
CREATE TABLE user_posts (
    user_id BIGINT,
    created_at TIMESTAMP,
    post_id TEXT,
    PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │       posts         │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id (unique)    │       │ post_id (unique)    │
│ username            │──┐    │ user_id (FK)        │──┐
│ display_name        │  │    │ content_text        │  │
│ is_celebrity        │  │    │ privacy             │  │
│ followers_count     │  │    │ likes_count         │  │
└─────────────────────┘  │    │ created_at          │  │
                         │    └─────────────────────┘  │
                         │                             │
                         │    ┌─────────────────────┐  │
                         │    │      follows        │  │
                         │    ├─────────────────────┤  │
                         └───>│ follower_id (FK)    │  │
                         └───>│ following_id (FK)   │  │
                              │ interaction_score   │  │
                              │ created_at          │  │
                              └─────────────────────┘  │
                                                       │
┌─────────────────────┐       ┌─────────────────────┐  │
│    post_likes       │       │     post_media      │  │
├─────────────────────┤       ├─────────────────────┤  │
│ id (PK)             │       │ id (PK)             │  │
│ post_id (FK)        │───────│ post_id (FK)        │──┘
│ user_id (FK)        │       │ media_url           │
│ created_at          │       │ media_type          │
└─────────────────────┘       │ dimensions          │
                              └─────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  REDIS CACHE                         │
├─────────────────────────────────────────────────────┤
│ feed:{user_id}        → Sorted Set of post_ids      │
│ post:{post_id}        → Hash of post data           │
│ user:{user_id}:following → Set of following_ids     │
│ engagement:{post_id}  → Hash of counts              │
└─────────────────────────────────────────────────────┘
```

---

## Ranking Algorithm

### Signal Weights

```java
public class FeedRanker {
    
    public double calculateScore(Post post, User viewer) {
        double score = 0.0;
        
        // 1. Recency (decay over time)
        long ageHours = getAgeHours(post.getCreatedAt());
        double recencyScore = Math.exp(-ageHours / 24.0);  // Half-life: 24 hours
        score += recencyScore * 0.30;
        
        // 2. Engagement (normalized)
        double engagementScore = normalizeEngagement(
            post.getLikesCount(),
            post.getCommentsCount(),
            post.getSharesCount()
        );
        score += engagementScore * 0.25;
        
        // 3. Relationship strength
        double relationshipScore = getRelationshipScore(viewer, post.getAuthor());
        score += relationshipScore * 0.25;
        
        // 4. Content type preference
        double contentScore = getContentPreference(viewer, post.getContentType());
        score += contentScore * 0.10;
        
        // 5. Author authority
        double authorScore = post.getAuthor().isVerified() ? 0.1 : 0.05;
        score += authorScore * 0.10;
        
        return score;
    }
    
    private double getRelationshipScore(User viewer, User author) {
        Follow follow = followRepository.find(viewer.getId(), author.getId());
        if (follow == null) return 0.0;
        
        // Based on interaction history
        return follow.getInteractionScore();
    }
}
```

---

## Cursor-Based Pagination

```java
public class FeedCursor {
    
    // Cursor encodes: last_score|last_post_id|last_timestamp
    
    public static String encode(double score, String postId, long timestamp) {
        String data = score + "|" + postId + "|" + timestamp;
        return Base64.getEncoder().encodeToString(data.getBytes());
    }
    
    public static FeedCursor decode(String cursor) {
        String data = new String(Base64.getDecoder().decode(cursor));
        String[] parts = data.split("\\|");
        return new FeedCursor(
            Double.parseDouble(parts[0]),
            parts[1],
            Long.parseLong(parts[2])
        );
    }
}

// Query with cursor
public List<Post> getFeed(String userId, String cursor, int limit) {
    if (cursor == null) {
        // First page
        return redis.zrevrange("feed:" + userId, 0, limit - 1);
    }
    
    FeedCursor c = FeedCursor.decode(cursor);
    
    // Get posts with score less than cursor
    return redis.zrevrangeByScore(
        "feed:" + userId,
        c.getScore(),
        "-inf",
        limit
    );
}
```

---

## Summary

| Component         | Technology/Approach                     |
| ----------------- | --------------------------------------- |
| Feed API          | REST + WebSocket for real-time          |
| Posts storage     | PostgreSQL                              |
| Feed cache        | Redis Sorted Sets                       |
| Social graph      | PostgreSQL + Redis cache                |
| Activity stream   | Cassandra (for fan-out writes)          |
| Pagination        | Cursor-based (score + id)               |
| Ranking           | Multi-signal scoring                    |

