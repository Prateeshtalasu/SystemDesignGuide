# News Feed - Data Model & Architecture

## Component Overview

Before looking at diagrams, let's understand each component and why it exists.

### Components Explained

| Component              | Purpose                                | Why It Exists                                    |
| ---------------------- | -------------------------------------- | ------------------------------------------------ |
| **Post Service**       | Handles post creation                  | Core content creation functionality              |
| **Feed Service**       | Serves personalized feeds              | Main user-facing service                         |
| **Fan-out Service**    | Distributes posts to feeds             | Handles write amplification                      |
| **Ranking Service**    | Scores and orders posts                | Relevance is key to engagement                   |
| **Graph Service**      | Manages follow relationships           | Who follows whom                                 |
| **Timeline Cache**     | Stores precomputed feeds               | Fast feed reads                                  |
| **Notification Service**| Alerts users of activity              | Engagement driver                                |

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

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                           │
│                    (Mobile Apps, Web Browsers, API Consumers)                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
            ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
            │  CDN/Edge   │     │ WebSocket   │     │ API Gateway │
            │  (Static)   │     │ Gateway     │     │             │
            └─────────────┘     └──────┬──────┘     └──────┬──────┘
                                       │                   │
                                       └─────────┬─────────┘
                                                 │
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              APPLICATION LAYER                                       │
│                                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │    Feed     │  │    Post     │  │   Graph     │  │  Ranking    │               │
│  │   Service   │  │   Service   │  │  Service    │  │  Service    │               │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│         │                │                │                │                       │
└─────────┼────────────────┼────────────────┼────────────────┼───────────────────────┘
          │                │                │                │
          │                │                │                │
┌─────────┼────────────────┼────────────────┼────────────────┼───────────────────────┐
│         ▼                ▼                ▼                ▼                       │
│                              DATA LAYER                                             │
│                                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Redis     │  │ PostgreSQL  │  │  Cassandra  │  │   Kafka     │              │
│  │   Cluster   │  │   Cluster   │  │   Cluster   │  │   Cluster   │              │
│  │  (Cache)    │  │  (Primary)  │  │  (Feeds)    │  │  (Events)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ASYNC PROCESSING                                        │
│                                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  Fan-out    │  │ Engagement  │  │Notification │  │  Analytics  │               │
│  │  Workers    │  │  Workers    │  │  Workers    │  │  Pipeline   │               │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Post Creation Flow

### Sequence Diagram

```
┌──────┐     ┌─────────┐     ┌─────────┐     ┌───────┐     ┌─────────┐     ┌─────────┐
│Client│     │   API   │     │  Post   │     │ Kafka │     │Fan-out  │     │  Redis  │
│      │     │ Gateway │     │ Service │     │       │     │ Worker  │     │ (Feeds) │
└──┬───┘     └────┬────┘     └────┬────┘     └───┬───┘     └────┬────┘     └────┬────┘
   │              │               │              │              │              │
   │ POST /posts  │               │              │              │              │
   │─────────────>│               │              │              │              │
   │              │               │              │              │              │
   │              │ Create post   │              │              │              │
   │              │──────────────>│              │              │              │
   │              │               │              │              │              │
   │              │               │ Store in DB  │              │              │
   │              │               │──────────────────────────────────────────────────>
   │              │               │              │              │              │
   │              │               │ Publish event│              │              │
   │              │               │─────────────>│              │              │
   │              │               │              │              │              │
   │              │ 201 Created   │              │              │              │
   │              │<──────────────│              │              │              │
   │              │               │              │              │              │
   │ Post created │               │              │              │              │
   │<─────────────│               │              │              │              │
   │              │               │              │              │              │
   │              │               │              │ Consume      │              │
   │              │               │              │─────────────>│              │
   │              │               │              │              │              │
   │              │               │              │              │ Get followers│
   │              │               │              │              │──────────────│
   │              │               │              │              │              │
   │              │               │              │              │ For each:    │
   │              │               │              │              │ Add to feed  │
   │              │               │              │              │─────────────>│
   │              │               │              │              │              │
```

### Fan-out Decision Logic

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAN-OUT DECISION                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

                         ┌───────────────────┐
                         │   New Post Event  │
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │  Check Author's   │
                         │  Follower Count   │
                         └─────────┬─────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
        ┌─────────────────────┐      ┌─────────────────────┐
        │  < 10K Followers    │      │  > 10K Followers    │
        │  (Regular User)     │      │  (Celebrity)        │
        └──────────┬──────────┘      └──────────┬──────────┘
                   │                            │
                   ▼                            ▼
        ┌─────────────────────┐      ┌─────────────────────┐
        │  FAN-OUT ON WRITE   │      │  FAN-OUT ON READ    │
        │                     │      │                     │
        │  Write to all       │      │  Store post only    │
        │  followers' feeds   │      │  Merge at read time │
        │                     │      │                     │
        │  Latency: High      │      │  Latency: Low       │
        │  Read: Fast         │      │  Read: Slower       │
        └─────────────────────┘      └─────────────────────┘
```

---

## Feed Read Flow

### Sequence Diagram

```
┌──────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│Client│     │   API   │     │  Feed   │     │ Ranking │     │  Redis  │     │Cassandra│
│      │     │ Gateway │     │ Service │     │ Service │     │ (Cache) │     │ (Feeds) │
└──┬───┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
   │              │               │              │              │              │
   │ GET /feed    │               │              │              │              │
   │─────────────>│               │              │              │              │
   │              │               │              │              │              │
   │              │ Get feed      │              │              │              │
   │              │──────────────>│              │              │              │
   │              │               │              │              │              │
   │              │               │ Get cached feed             │              │
   │              │               │─────────────────────────────>│              │
   │              │               │              │              │              │
   │              │               │              │    Feed posts│              │
   │              │               │<─────────────────────────────│              │
   │              │               │              │              │              │
   │              │               │ Get celebrity posts          │              │
   │              │               │────────────────────────────────────────────>│
   │              │               │              │              │              │
   │              │               │              │Celebrity posts│              │
   │              │               │<────────────────────────────────────────────│
   │              │               │              │              │              │
   │              │               │ Merge & Rank │              │              │
   │              │               │─────────────>│              │              │
   │              │               │              │              │              │
   │              │               │ Ranked feed  │              │              │
   │              │               │<─────────────│              │              │
   │              │               │              │              │              │
   │              │ Feed response │              │              │              │
   │              │<──────────────│              │              │              │
   │              │               │              │              │              │
   │ Feed data    │               │              │              │              │
   │<─────────────│               │              │              │              │
```

### Hybrid Feed Assembly

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FEED ASSEMBLY                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────────────────────┐
                    │          Feed Request             │
                    │          (User opens app)         │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
        ┌───────────────────────┐           ┌───────────────────────┐
        │   PRECOMPUTED FEED    │           │   CELEBRITY POSTS     │
        │   (Redis/Cassandra)   │           │   (Fan-out on read)   │
        │                       │           │                       │
        │   Posts from regular  │           │   Posts from users    │
        │   users (< 10K        │           │   with > 10K          │
        │   followers)          │           │   followers           │
        │                       │           │                       │
        │   Already ranked      │           │   Need to fetch       │
        │   and cached          │           │   and merge           │
        └───────────┬───────────┘           └───────────┬───────────┘
                    │                                   │
                    │                                   │
                    └─────────────────┬─────────────────┘
                                      │
                                      ▼
                    ┌───────────────────────────────────┐
                    │           MERGE & RANK            │
                    │                                   │
                    │   1. Combine both sources         │
                    │   2. Apply ranking algorithm      │
                    │   3. Deduplicate                  │
                    │   4. Apply user preferences       │
                    └─────────────────┬─────────────────┘
                                      │
                                      ▼
                    ┌───────────────────────────────────┐
                    │          FINAL FEED               │
                    │   (Top N posts, paginated)        │
                    └───────────────────────────────────┘
```

---

## Fan-out Worker Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAN-OUT SYSTEM                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KAFKA CLUSTER                                           │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Topic: post-events                                   │    │
│  │                                                                              │    │
│  │  Partition 0: [post1, post4, post7, ...]                                    │    │
│  │  Partition 1: [post2, post5, post8, ...]                                    │    │
│  │  Partition 2: [post3, post6, post9, ...]                                    │    │
│  │  ...                                                                        │    │
│  │  Partition N: [...]                                                         │    │
│  │                                                                              │    │
│  │  Key: author_id (ensures ordering per author)                               │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
          │ Fan-out Worker 1│ │ Fan-out Worker 2│ │ Fan-out Worker N│
          │                 │ │                 │ │                 │
          │ Partitions: 0-3 │ │ Partitions: 4-7 │ │ Partitions: ... │
          └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
                   │                   │                   │
                   │                   │                   │
                   ▼                   ▼                   ▼
          ┌─────────────────────────────────────────────────────────┐
          │                    PROCESSING STEPS                      │
          │                                                          │
          │  1. Receive post event                                   │
          │  2. Check if celebrity (> 10K followers)                 │
          │     - If yes: Skip fan-out, store in celebrity posts     │
          │     - If no: Continue to step 3                          │
          │  3. Get follower list from Graph Service                 │
          │  4. Batch followers (1000 per batch)                     │
          │  5. For each batch:                                      │
          │     - Calculate ranking score                            │
          │     - Write to follower's feed cache                     │
          │  6. Acknowledge Kafka offset                             │
          └─────────────────────────────────────────────────────────┘
                                        │
                                        ▼
          ┌─────────────────────────────────────────────────────────┐
          │                    FEED CACHES                           │
          │                                                          │
          │  Redis Cluster:                                          │
          │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
          │  │ feed:user_1  │  │ feed:user_2  │  │ feed:user_N  │   │
          │  │ [post_ids]   │  │ [post_ids]   │  │ [post_ids]   │   │
          │  └──────────────┘  └──────────────┘  └──────────────┘   │
          │                                                          │
          │  Cassandra (overflow/persistence):                       │
          │  ┌──────────────────────────────────────────────────┐   │
          │  │ user_feed table (user_id, created_at, post_id)   │   │
          │  └──────────────────────────────────────────────────┘   │
          └─────────────────────────────────────────────────────────┘
```

---

## Real-time Updates (WebSocket)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              REAL-TIME SYSTEM                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT CONNECTIONS                                      │
│                                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  Client 1   │  │  Client 2   │  │  Client 3   │  │  Client N   │               │
│  │  (User A)   │  │  (User B)   │  │  (User C)   │  │  (User ...)│               │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│         │                │                │                │                       │
│         └────────────────┴────────────────┴────────────────┘                       │
│                                   │                                                 │
│                                   ▼                                                 │
│                    ┌───────────────────────────┐                                   │
│                    │   WebSocket Load Balancer │                                   │
│                    │   (Sticky Sessions)       │                                   │
│                    └─────────────┬─────────────┘                                   │
└──────────────────────────────────┼──────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
        ┌───────────────────────┐     ┌───────────────────────┐
        │   WS Server 1         │     │   WS Server N         │
        │                       │     │                       │
        │   Connected users:    │     │   Connected users:    │
        │   [A, D, G, ...]      │     │   [B, C, E, F, ...]   │
        │                       │     │                       │
        │   User → Connection   │     │   User → Connection   │
        │   mapping in memory   │     │   mapping in memory   │
        └───────────┬───────────┘     └───────────┬───────────┘
                    │                             │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────┐
                    │   Redis Pub/Sub           │
                    │                           │
                    │   Channel: user:{user_id} │
                    │   Message: new_post event │
                    └───────────────────────────┘
                                   ▲
                                   │
                    ┌───────────────────────────┐
                    │   Fan-out Worker          │
                    │                           │
                    │   After writing to feed:  │
                    │   PUBLISH user:{id} event │
                    └───────────────────────────┘
```

---

## Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MULTI-LAYER CACHING                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                               ┌───────────────────┐
                               │      CLIENT       │
                               └─────────┬─────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: CLIENT CACHE                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  - In-memory cache on mobile/web                                             │    │
│  │  - Last fetched feed                                                         │    │
│  │  - TTL: 5 minutes                                                            │    │
│  │  - Show immediately, then refresh                                            │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS or stale
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: CDN/EDGE CACHE                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  - Static assets (images, videos)                                            │    │
│  │  - NOT for personalized feeds (can't cache)                                  │    │
│  │  - User profile images, post media                                           │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: FEED CACHE (Redis)                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Hot users (active in last 24h):                                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │    │
│  │  │ feed:{user_id} = Sorted Set of (score, post_id)                      │   │    │
│  │  │ Size: 500 posts per user                                              │   │    │
│  │  │ TTL: 24 hours                                                         │   │    │
│  │  └──────────────────────────────────────────────────────────────────────┘   │    │
│  │                                                                              │    │
│  │  Post data cache:                                                            │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │    │
│  │  │ post:{post_id} = Hash of post fields                                 │   │    │
│  │  │ TTL: 1 hour (frequently accessed posts)                              │   │    │
│  │  └──────────────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│  Hit Rate: ~90% for active users                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ Cache MISS
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: PERSISTENT STORAGE                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Cassandra (feed history):                                                   │    │
│  │  - Full feed history per user                                                │    │
│  │  - Used for cold users or pagination beyond cache                            │    │
│  │                                                                              │    │
│  │  PostgreSQL (posts, users):                                                  │    │
│  │  - Source of truth for all data                                              │    │
│  │  - Used for feed regeneration                                                │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Failure Points and Mitigation

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FAILURE ANALYSIS                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

Component              Failure Mode           Impact              Mitigation
─────────────────────────────────────────────────────────────────────────────────────

┌─────────────────┐
│ Feed Service    │ ─── Service crash ───── Feed unavailable ── Multiple replicas,
│                 │                                               load balancer
└─────────────────┘

┌─────────────────┐
│ Redis Cache     │ ─── Node failure ────── Slower feeds ─────── Redis Cluster,
│                 │                         (cache miss)          automatic failover
└─────────────────┘

┌─────────────────┐
│ Fan-out Worker  │ ─── Worker crash ────── Delayed feed ─────── Kafka rebalancing,
│                 │                         updates               idempotent writes
└─────────────────┘

┌─────────────────┐
│ Kafka           │ ─── Broker down ─────── Events delayed ───── 3x replication,
│                 │                                               automatic failover
└─────────────────┘

┌─────────────────┐
│ PostgreSQL      │ ─── Primary down ────── Writes fail ──────── Automatic failover
│                 │                                               to replica
└─────────────────┘

┌─────────────────┐
│ Ranking Service │ ─── Service crash ───── Unranked feed ────── Fallback to
│                 │                                               chronological
└─────────────────┘

┌─────────────────┐
│ WebSocket       │ ─── Server crash ────── Lost connections ─── Client reconnect,
│                 │                                               sticky sessions
└─────────────────┘

GRACEFUL DEGRADATION STRATEGY:
─────────────────────────────
1. If ranking fails → Show chronological feed
2. If cache fails → Regenerate from database
3. If fan-out delayed → Show slightly stale feed
4. If celebrity posts fail → Show only regular posts
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES DEPLOYMENT                                   │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                              REGION: US-EAST                                   │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │  │
│  │  │  feed-service   │  │  post-service   │  │ ranking-service │               │  │
│  │  │  Replicas: 50   │  │  Replicas: 10   │  │  Replicas: 100  │               │  │
│  │  │  CPU: 8 cores   │  │  CPU: 4 cores   │  │  CPU: 8 cores   │               │  │
│  │  │  Memory: 16 GB  │  │  Memory: 8 GB   │  │  Memory: 32 GB  │               │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘               │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │  │
│  │  │ fanout-workers  │  │  ws-servers     │  │  graph-service  │               │  │
│  │  │  Replicas: 50   │  │  Replicas: 20   │  │  Replicas: 10   │               │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘               │  │
│  │                                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                         DATA STORES                                      │ │  │
│  │  │                                                                          │ │  │
│  │  │  Redis Cluster (22 nodes)  │  Kafka (10 brokers)  │  PostgreSQL (HA)    │ │  │
│  │  │  Cassandra (15 nodes)      │  Zookeeper (3 nodes) │                     │ │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  Similar deployments in: US-WEST, EU-WEST, APAC                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect            | Decision                | Rationale                               |
| ----------------- | ----------------------- | --------------------------------------- |
| Fan-out strategy  | Hybrid                  | Write for regular, read for celebrities |
| Feed cache        | Redis Sorted Sets       | Fast reads, natural ordering            |
| Event streaming   | Kafka                   | Reliable async processing               |
| Feed storage      | Cassandra               | Time-series optimized                   |
| Real-time         | WebSocket + Redis Pub/Sub | Low-latency updates                   |
| Ranking           | ML service              | Personalized relevance                  |

