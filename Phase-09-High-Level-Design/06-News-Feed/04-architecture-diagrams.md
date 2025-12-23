# News Feed - Architecture Diagrams

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

