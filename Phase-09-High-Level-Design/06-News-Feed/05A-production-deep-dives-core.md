# News Feed - Production Deep Dives (Core)

## Overview

This document covers the core technical implementation details for the News Feed system: asynchronous messaging with Kafka for fan-out, caching strategies with Redis, and the ranking algorithm. Search is not applicable for a news feed system as feeds are personalized rather than searched.

---

## 1. Asynchronous Messaging (Kafka for Fan-out)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant data pipelines. Unlike traditional message queues (RabbitMQ, ActiveMQ), Kafka stores messages durably on disk and allows multiple consumers to read the same data independently.

**What problems does Kafka solve?**

1. **Buffering**: Absorbs traffic spikes without overwhelming downstream services
2. **Decoupling**: Producers and consumers operate independently
3. **Ordering**: Guarantees message order within a partition
4. **Durability**: Messages persist even if consumers are temporarily down
5. **Replay**: Consumers can re-read historical messages

**Core Kafka concepts:**

| Concept | Definition |
|---------|------------|
| **Topic** | A named stream of messages (like a database table) |
| **Partition** | A topic is split into partitions for parallelism |
| **Offset** | A unique ID for each message within a partition |
| **Consumer Group** | A group of consumers that share the work of reading a topic |
| **Broker** | A Kafka server that stores and serves messages |

### B) OUR USAGE: How We Use Kafka Here

**Why async vs sync for fan-out?**

News feed systems require distributing posts to millions of followers. If we did this synchronously:
- For a user with 10,000 followers: 10,000 database writes
- Each write: ~5-10ms
- Total blocking time: 50-100 seconds (unacceptable)

By using Kafka, the post creation returns immediately (< 100ms), and fan-out happens asynchronously.

**Events in our system:**

| Event | Purpose | Volume |
|-------|---------|--------|
| `post-events` | Triggers fan-out to followers | 100K posts/second peak |
| `engagement-events` | Tracks likes, comments, shares | 1M events/second peak |

**Topic/Queue Design:**

```
Topic: post-events
Partitions: 32
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
```

**Why 32 partitions?**
- Allows up to 32 parallel fan-out workers
- Distributes load evenly across consumers
- Supports high-throughput fan-out processing

**Partition Key Choice:**

We partition by `author_id` hash. This ensures:
- All posts from the same author go to the same partition
- Ordering is preserved per author (posts appear in chronological order)
- Even distribution across partitions

### Hot Partition Mitigation

**Problem:** If a celebrity posts frequently, one partition could become hot, creating a bottleneck.

**Example Scenario:**
```
Normal user: 1 post/hour → Partition 15
Celebrity: 10 posts/hour → Partition 15 (same partition!)
Result: Partition 15 receives 10x traffic, becomes bottleneck
```

**Detection:**
- Monitor partition lag per partition
- Alert when partition lag > 2x average
- Track partition throughput metrics
- Monitor consumer lag per partition

**Mitigation Strategies:**

1. **Hybrid Fan-out Strategy:**
   - Regular users (< 10K followers): Fan-out on write (pre-compute feeds)
   - Celebrities (> 10K followers): Fan-out on read (don't pre-compute, merge at query time)
   - This prevents celebrity posts from overwhelming fan-out workers

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute celebrity posts across multiple partitions

3. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

4. **Rate Limiting:**
   - Rate limit per author (prevent single author from overwhelming)
   - Throttle high-volume authors
   - Buffer posts locally if rate limit exceeded

**Implementation:**

```java
@Service
public class HotPartitionDetector {
    
    public void detectAndMitigate() {
        Map<Integer, Long> partitionLag = getPartitionLag();
        double avgLag = partitionLag.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);
        
        partitionLag.entrySet().stream()
            .filter(e -> e.getValue() > avgLag * 2)
            .forEach(e -> {
                int partition = e.getKey();
                long lag = e.getValue();
                
                // Alert ops team
                alertOps("Hot partition detected: " + partition + ", lag: " + lag);
                
                // Auto-scale consumers for this partition
                scaleConsumers(partition, lag);
            });
    }
}
```

**Monitoring:**
- Partition lag per partition (Kafka metrics)
- Partition throughput (posts/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Consumer Group Design:**

```
Consumer Group: fanout-workers
Consumers: 32 instances (one per partition)
Processing: Batch processing of 500-1000 followers
```

**Ordering Guarantees:**

- **Per-author ordering**: Guaranteed (all posts from same author go to same partition)
- **Global ordering**: Not guaranteed (not needed for feeds)

**Offset Management:**

We use manual offset commits to ensure at-least-once delivery:

```java
@KafkaListener(topics = "post-events", groupId = "fanout-workers")
public void handlePostEvent(PostEvent event, Acknowledgment ack) {
    try {
        processFanout(event);
        ack.acknowledge();  // Commit offset only after successful processing
    } catch (Exception e) {
        // Don't acknowledge, will be reprocessed
        log.error("Failed to process fan-out", e);
    }
}
```

**Deduplication Strategy:**

Since we use at-least-once delivery, duplicates are possible. We handle this by:
1. Using idempotent Redis operations (ZADD updates score if exists)
2. Post IDs are unique (primary key constraint)
3. Accepting slight over-delivery (feed deduplication handles it)

**Fan-out Strategy:**

We use a hybrid approach:
- **Fan-out on Write**: For regular users (< 10,000 followers)
  - Pre-compute feeds in Redis
  - Fast reads, higher write cost
- **Fan-out on Read**: For celebrities (> 10,000 followers)
  - Store posts in database
  - Compute feed on demand
  - Lower write cost, acceptable read latency

### Fan-out on Write Implementation

```java
@Service
public class FanoutService {
    
    private final GraphService graphService;
    private final FeedCacheService feedCache;
    private final KafkaTemplate<String, PostEvent> kafkaTemplate;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final int BATCH_SIZE = 1000;
    
    @KafkaListener(topics = "post-events")
    public void handlePostEvent(PostEvent event) {
        Post post = event.getPost();
        User author = post.getAuthor();
        
        // Check if celebrity
        if (author.getFollowersCount() > CELEBRITY_THRESHOLD) {
            // Store in celebrity posts table, no fan-out
            celebrityPostStore.save(post);
            log.info("Celebrity post stored without fan-out: {}", post.getId());
            return;
        }
        
        // Get all followers
        List<Long> followers = graphService.getFollowers(author.getId());
        
        // Calculate ranking score
        double baseScore = calculateBaseScore(post);
        
        // Batch process followers
        Lists.partition(followers, BATCH_SIZE).forEach(batch -> {
            processBatch(batch, post, baseScore);
        });
    }
    
    private void processBatch(List<Long> followers, Post post, double baseScore) {
        // Prepare feed entries
        Map<String, Double> feedEntries = new HashMap<>();
        
        for (Long followerId : followers) {
            // Personalize score based on relationship
            double personalizedScore = personalizeScore(followerId, post, baseScore);
            String feedKey = "feed:" + followerId;
            feedEntries.put(feedKey, personalizedScore);
        }
        
        // Batch write to Redis
        feedCache.batchAddToFeeds(feedEntries, post.getId());
        
        // Publish real-time notifications
        notifyActiveUsers(followers, post);
    }
    
    private double calculateBaseScore(Post post) {
        long ageMinutes = ChronoUnit.MINUTES.between(post.getCreatedAt(), Instant.now());
        
        // Time decay: half-life of 6 hours
        double recencyScore = Math.exp(-ageMinutes / 360.0);
        
        // Engagement boost (for posts with early engagement)
        double engagementScore = Math.log1p(
            post.getLikesCount() + 
            post.getCommentsCount() * 2 + 
            post.getSharesCount() * 3
        ) / 10.0;
        
        return recencyScore * 0.7 + engagementScore * 0.3;
    }
}
```

### Fan-out on Read Implementation

```java
@Service
public class FeedService {
    
    private final FeedCacheService feedCache;
    private final CelebrityPostService celebrityPostService;
    private final GraphService graphService;
    private final RankingService rankingService;
    
    public FeedResponse getFeed(Long userId, String cursor, int limit) {
        // 1. Get precomputed feed (regular users' posts)
        List<FeedEntry> precomputedFeed = feedCache.getFeed(userId, cursor, limit * 2);
        
        // 2. Get celebrity posts (fan-out on read)
        List<Long> followedCelebrities = graphService.getFollowedCelebrities(userId);
        List<Post> celebrityPosts = fetchCelebrityPosts(followedCelebrities, cursor);
        
        // 3. Merge feeds
        List<FeedEntry> merged = mergeFeed(precomputedFeed, celebrityPosts, userId);
        
        // 4. Apply final ranking
        List<FeedEntry> ranked = rankingService.rank(merged, userId);
        
        // 5. Paginate
        List<FeedEntry> page = ranked.subList(0, Math.min(limit, ranked.size()));
        
        return FeedResponse.builder()
            .posts(page)
            .nextCursor(generateCursor(page))
            .hasMore(ranked.size() > limit)
            .build();
    }
    
    private List<Post> fetchCelebrityPosts(List<Long> celebrityIds, String cursor) {
        if (celebrityIds.isEmpty()) {
            return Collections.emptyList();
        }
        
        // Batch fetch recent posts from all followed celebrities
        Instant since = parseCursorTime(cursor);
        
        return celebrityPostService.getRecentPosts(
            celebrityIds,
            since,
            100  // Max celebrity posts to consider
        );
    }
    
    private List<FeedEntry> mergeFeed(
            List<FeedEntry> precomputed, 
            List<Post> celebrityPosts,
            Long userId) {
        
        // Convert celebrity posts to feed entries
        List<FeedEntry> celebrityEntries = celebrityPosts.stream()
            .map(post -> FeedEntry.builder()
                .postId(post.getId())
                .authorId(post.getAuthorId())
                .score(calculateScore(post, userId))
                .createdAt(post.getCreatedAt())
                .source("celebrity")
                .build())
            .collect(Collectors.toList());
        
        // Merge and sort by score
        List<FeedEntry> merged = new ArrayList<>();
        merged.addAll(precomputed);
        merged.addAll(celebrityEntries);
        
        // Deduplicate (same post might appear in both)
        Map<Long, FeedEntry> deduped = merged.stream()
            .collect(Collectors.toMap(
                FeedEntry::getPostId,
                e -> e,
                (e1, e2) -> e1.getScore() > e2.getScore() ? e1 : e2
            ));
        
        return new ArrayList<>(deduped.values());
    }
}
```

### Kafka Configuration for Fan-out

```yaml
# Kafka topic configuration
topics:
  post-events:
    partitions: 32
    replication-factor: 3
    config:
      retention.ms: 604800000  # 7 days
      min.insync.replicas: 2
      
  engagement-events:
    partitions: 16
    replication-factor: 3
    config:
      retention.ms: 86400000  # 1 day

# Consumer configuration
consumer:
  group-id: fanout-workers
  auto-offset-reset: earliest
  enable-auto-commit: false
  max-poll-records: 500
  max-poll-interval-ms: 300000
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Post Fan-out**

**Scenario**: User with 5,000 followers creates a post

```
Step 1: Post Creation
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/posts                                               │
│ Author: user_123 (5,000 followers)                          │
│ Content: "Hello world!"                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Post Service
┌─────────────────────────────────────────────────────────────┐
│ 1. Validate content                                          │
│ 2. Store in PostgreSQL (posts table)                        │
│ 3. Generate post_id: "post_abc123"                          │
│ 4. Publish to Kafka topic: post-events                      │
│                                                              │
│ Kafka Message:                                               │
│ {                                                            │
│   "event_type": "POST_CREATED",                             │
│   "post_id": "post_abc123",                                 │
│   "author_id": "user_123",                                  │
│   "follower_count": 5000,                                   │
│   "timestamp": "2024-01-20T10:00:00Z"                       │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Fan-out Worker (consumes from Kafka)
┌─────────────────────────────────────────────────────────────┐
│ 1. Check follower count: 5,000 < 10,000 (not celebrity)     │
│ 2. Fetch follower list from Graph Service                   │
│ 3. Calculate base score: 0.95 (very recent)                 │
│                                                              │
│ Processing:                                                  │
│ - Batch 1: followers 0-999                                  │
│ - Batch 2: followers 1000-1999                              │
│ - Batch 3: followers 2000-2999                              │
│ - Batch 4: followers 3000-3999                              │
│ - Batch 5: followers 4000-4999                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Redis Writes (per batch)
┌─────────────────────────────────────────────────────────────┐
│ For each follower in batch:                                  │
│                                                              │
│ ZADD feed:follower_1 0.95 post_abc123                       │
│ ZADD feed:follower_2 0.92 post_abc123  (lower affinity)     │
│ ZADD feed:follower_3 0.95 post_abc123                       │
│ ...                                                          │
│                                                              │
│ Pipeline execution: ~50ms per batch of 1000                 │
│ Total fan-out time: ~250ms for 5,000 followers              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Real-time Notification
┌─────────────────────────────────────────────────────────────┐
│ For active followers (currently online):                     │
│                                                              │
│ PUBLISH user:follower_1:feed {                              │
│   "type": "new_post",                                       │
│   "post_id": "post_abc123",                                 │
│   "author": "user_123"                                      │
│ }                                                            │
│                                                              │
│ WebSocket servers receive and push to connected clients     │
└─────────────────────────────────────────────────────────────┘
```

### Failure Scenario: Fan-out Worker Crash

```
Scenario: Worker crashes mid-fan-out (after processing 3,000 of 5,000 followers)

┌─────────────────────────────────────────────────────────────┐
│ Before Crash:                                                │
│ - Batches 1-3 completed (followers 0-2999)                  │
│ - Kafka offset NOT committed yet                            │
│ - Worker crashes                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Recovery (Kafka rebalance):                                  │
│ 1. Kafka detects worker failure (heartbeat timeout)         │
│ 2. Partition reassigned to another worker                   │
│ 3. New worker starts from last committed offset             │
│ 4. Re-processes entire post event                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Idempotent Handling:                                         │
│                                                              │
│ ZADD feed:follower_1 0.95 post_abc123                       │
│ → Already exists, score updated (no duplicate)              │
│                                                              │
│ Redis ZADD is naturally idempotent:                         │
│ - Same member with same score = no change                   │
│ - Same member with different score = update                 │
│                                                              │
│ Result: All 5,000 followers get post exactly once           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Caching (Redis for Feeds)

### Why Redis Sorted Sets?

| Structure        | Feed Storage                  | Pros                | Cons                |
| ---------------- | ----------------------------- | ------------------- | ------------------- |
| Redis List       | [post1, post2, post3]         | Simple              | No scoring          |
| Redis Set        | {post1, post2, post3}         | Fast membership     | No ordering         |
| Redis Sorted Set | {(score1, post1), ...}        | Ordered by score    | Slightly more memory|
| Redis Hash       | {post1: data, post2: data}    | Rich data           | No ordering         |

**Why Sorted Set Wins:**
1. **Natural Ranking**: Posts ordered by score automatically
2. **Range Queries**: `ZREVRANGE` for pagination is O(log N + M)
3. **Score Updates**: Can update post score without removing
4. **Memory Efficient**: Just stores post_id + score (~20 bytes per entry)
5. **Atomic Operations**: ZADD with NX/XX flags for safe updates

### Redis Feed Cache Implementation

```java
@Service
public class FeedCacheService {
    
    private final RedisTemplate<String, String> redis;
    private static final int MAX_FEED_SIZE = 500;
    private static final Duration FEED_TTL = Duration.ofHours(24);
    
    public void addToFeed(Long userId, Long postId, double score) {
        String key = "feed:" + userId;
        
        // Add to sorted set with score
        redis.opsForZSet().add(key, postId.toString(), score);
        
        // Trim to max size (remove lowest scores)
        redis.opsForZSet().removeRange(key, 0, -MAX_FEED_SIZE - 1);
        
        // Refresh TTL
        redis.expire(key, FEED_TTL);
    }
    
    public void batchAddToFeeds(Map<Long, Double> userScores, Long postId) {
        // Use pipeline for efficiency
        redis.executePipelined((RedisCallback<Object>) connection -> {
            userScores.forEach((userId, score) -> {
                byte[] key = ("feed:" + userId).getBytes();
                connection.zAdd(key, score, postId.toString().getBytes());
            });
            return null;
        });
    }
    
    public List<FeedEntry> getFeed(Long userId, String cursor, int limit) {
        String key = "feed:" + userId;
        
        double maxScore = cursor != null ? 
            parseCursorScore(cursor) : Double.MAX_VALUE;
        
        // Get posts with scores less than cursor
        Set<ZSetOperations.TypedTuple<String>> results = redis.opsForZSet()
            .reverseRangeByScoreWithScores(key, 0, maxScore, 0, limit);
        
        if (results == null || results.isEmpty()) {
            return Collections.emptyList();
        }
        
        return results.stream()
            .map(tuple -> FeedEntry.builder()
                .postId(Long.parseLong(tuple.getValue()))
                .score(tuple.getScore())
                .build())
            .collect(Collectors.toList());
    }
    
    public void removeFromFeed(Long userId, Long postId) {
        String key = "feed:" + userId;
        redis.opsForZSet().remove(key, postId.toString());
    }
    
    public void invalidateFeed(Long userId) {
        String key = "feed:" + userId;
        redis.delete(key);
    }
}
```

### Cache Warming Strategy

```java
@Service
public class FeedWarmingService {
    
    private final FeedCacheService feedCache;
    private final FeedGenerationService feedGenerator;
    
    // Warm cache for users who will likely open app soon
    @Scheduled(fixedRate = 60000)  // Every minute
    public void warmCachesForActiveUsers() {
        // Get users who were active in last hour but cache is cold
        List<Long> usersToWarm = getUsersNeedingWarmCache();
        
        for (Long userId : usersToWarm) {
            try {
                warmUserFeed(userId);
            } catch (Exception e) {
                log.warn("Failed to warm cache for user {}", userId, e);
            }
        }
    }
    
    private void warmUserFeed(Long userId) {
        // Check if cache exists
        if (feedCache.exists(userId)) {
            return;
        }
        
        // Generate feed
        List<FeedEntry> feed = feedGenerator.generateFeed(userId, 500);
        
        // Populate cache
        feed.forEach(entry -> 
            feedCache.addToFeed(userId, entry.getPostId(), entry.getScore())
        );
        
        log.info("Warmed cache for user {} with {} entries", userId, feed.size());
    }
    
    // Predictive warming based on user patterns
    public void predictiveWarm(Long userId) {
        // Get user's typical active times
        List<Integer> activeHours = getUserActiveHours(userId);
        int currentHour = LocalTime.now().getHour();
        
        // If user typically active in next hour, warm cache
        if (activeHours.contains((currentHour + 1) % 24)) {
            warmUserFeed(userId);
        }
    }
}
```

### Step-by-Step Simulation: Feed Read with Cache

**Scenario**: User opens app and requests feed

```
Step 1: Feed Request
┌─────────────────────────────────────────────────────────────┐
│ GET /v1/feed?limit=20                                        │
│ User: user_456                                               │
│ Following: 200 users (5 celebrities, 195 regular)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache
┌─────────────────────────────────────────────────────────────┐
│ ZREVRANGE feed:user_456 0 39 WITHSCORES                     │
│                                                              │
│ Result: 40 entries found (cache HIT)                        │
│ [                                                            │
│   (post_abc, 0.95),                                         │
│   (post_def, 0.92),                                         │
│   (post_ghi, 0.88),                                         │
│   ...                                                        │
│ ]                                                            │
│                                                              │
│ Latency: 2ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Fetch Celebrity Posts (Fan-out on Read)
┌─────────────────────────────────────────────────────────────┐
│ 1. Get followed celebrities: [celeb_1, celeb_2, ..., celeb_5]│
│                                                              │
│ 2. Query celebrity posts table:                             │
│    SELECT * FROM celebrity_posts                            │
│    WHERE author_id IN (celeb_1, ..., celeb_5)               │
│    AND created_at > NOW() - INTERVAL '24 hours'             │
│    LIMIT 50                                                  │
│                                                              │
│ Result: 15 celebrity posts                                  │
│ Latency: 10ms (indexed query)                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Merge and Rank
┌─────────────────────────────────────────────────────────────┐
│ Input:                                                       │
│ - 40 precomputed posts (from cache)                         │
│ - 15 celebrity posts (fetched)                              │
│                                                              │
│ Processing:                                                  │
│ 1. Calculate scores for celebrity posts                     │
│ 2. Merge into single list                                   │
│ 3. Deduplicate (if any overlap)                             │
│ 4. Sort by final score                                      │
│ 5. Apply diversity (max 3 consecutive from same author)     │
│                                                              │
│ Latency: 5ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Return Response
┌─────────────────────────────────────────────────────────────┐
│ {                                                            │
│   "posts": [...top 20 posts...],                            │
│   "pagination": {                                            │
│     "next_cursor": "cursor_xyz",                            │
│     "has_more": true                                        │
│   },                                                         │
│   "new_posts_count": 0                                      │
│ }                                                            │
│                                                              │
│ Total latency: 17ms (2 + 10 + 5)                            │
└─────────────────────────────────────────────────────────────┘
```

### Cache Miss Scenario

```
Scenario: Cold user (hasn't opened app in 7 days, cache expired)

Step 1: Cache Miss
┌─────────────────────────────────────────────────────────────┐
│ ZREVRANGE feed:user_789 0 39 WITHSCORES                     │
│ Result: (empty) - Key doesn't exist                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Regenerate Feed
┌─────────────────────────────────────────────────────────────┐
│ 1. Get following list: 200 users                            │
│                                                              │
│ 2. Query recent posts from followed users:                  │
│    SELECT p.*, u.display_name                               │
│    FROM posts p                                              │
│    JOIN follows f ON p.user_id = f.following_id             │
│    WHERE f.follower_id = user_789                           │
│    AND p.created_at > NOW() - INTERVAL '7 days'             │
│    ORDER BY p.created_at DESC                               │
│    LIMIT 500                                                 │
│                                                              │
│ Latency: 150ms (complex join)                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Rank and Cache (with Stampede Prevention)
┌─────────────────────────────────────────────────────────────┐
│ 1. Calculate scores for all 500 posts                       │
│ 2. Sort by score                                            │
│ 3. Check lock before regenerating:                          │
│    - Try acquire lock: "lock:feed:user_789"                │
│    - If lock unavailable: Return stale cache or wait       │
│    - If lock acquired: Regenerate feed                     │
│ 4. Store in Redis (async, don't block response):            │
│                                                              │
│    ZADD feed:user_789 0.95 post_1 0.92 post_2 ...          │
│    EXPIRE feed:user_789 86400                               │
│    Release lock                                              │
│                                                              │
│ 4. Return top 20 to user                                    │
│                                                              │
│ Total latency: ~200ms (first load)                          │
│ Subsequent loads: ~20ms (cached)                            │
└─────────────────────────────────────────────────────────────┘
```

### Cache Stampede Prevention

**Problem: Celebrity Post Thundering Herd**

When a celebrity posts, millions of followers' feed caches expire or become stale, causing a stampede of feed regeneration requests.

**Scenario:**
```
T+0s:   Celebrity posts (10M followers)
T+0s:   Millions of users open app
T+0s:   Feed cache expired or stale for many users
T+0s:   Millions of feed regeneration requests hit database
T+1s:   Database overwhelmed, latency spikes to seconds
```

**Prevention Strategy: Distributed Lock + Stale-While-Revalidate**

```java
@Service
public class FeedService {
    
    private final RedisTemplate<String, String> redis;
    private final DistributedLock lockService;
    
    public List<Post> getFeed(Long userId, int limit) {
        // 1. Try cache first
        Set<String> cachedPostIds = redis.opsForZSet()
            .reverseRange("feed:" + userId, 0, limit * 2 - 1);
        
        if (!cachedPostIds.isEmpty()) {
            // Probabilistic early expiration (1% chance)
            if (Math.random() < 0.01) {
                refreshFeedAsync(userId);
            }
            return fetchPostsByIds(cachedPostIds, limit);
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:feed:" + userId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(10));
        
        if (!acquired) {
            // Another thread is regenerating, return stale cache if available
            // Or wait briefly and retry
            try {
                Thread.sleep(100);
                cachedPostIds = redis.opsForZSet()
                    .reverseRange("feed:" + userId, 0, limit * 2 - 1);
                if (!cachedPostIds.isEmpty()) {
                    return fetchPostsByIds(cachedPostIds, limit);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache
            cachedPostIds = redis.opsForZSet()
                .reverseRange("feed:" + userId, 0, limit * 2 - 1);
            if (!cachedPostIds.isEmpty()) {
                return fetchPostsByIds(cachedPostIds, limit);
            }
            
            // 4. Regenerate feed (only one thread does this)
            List<Post> feed = regenerateFeed(userId);
            
            // 5. Cache feed
            cacheFeed(userId, feed);
            
            return feed.stream().limit(limit).collect(Collectors.toList());
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
    
    private void refreshFeedAsync(Long userId) {
        // Background refresh without blocking
        CompletableFuture.runAsync(() -> {
            List<Post> feed = regenerateFeed(userId);
            cacheFeed(userId, feed);
        });
    }
}
```

**Cache Stampede Simulation:**

```
Scenario: Celebrity posts, 1M followers open app simultaneously

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Celebrity posts                                        │
│ T+0s: 1M users open app (feed cache expired)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Without Protection:                                          │
│ - All 1M requests see cache MISS                           │
│ - All 1M requests hit database                             │
│ - Database overwhelmed (1M concurrent queries)            │
│ - Latency: 20ms → 5 seconds                                │
│ - Many requests timeout                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ With Protection (Locking):                                  │
│ - Request 1: Acquires lock, regenerates feed                │
│ - Requests 2-1M: Lock unavailable, wait 100ms              │
│ - Request 1: Populates cache, releases lock                │
│ - Requests 2-1M: Retry cache → HIT                         │
│ - Database: Only 1 query (not 1M)                          │
│ - Latency: 20ms (cache hit) for 99.99% of requests         │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Ranking Algorithm Deep Dive

### Multi-Signal Ranking

```java
@Service
public class RankingService {
    
    private final UserInteractionService interactionService;
    private final ContentAnalyzer contentAnalyzer;
    
    public List<FeedEntry> rank(List<FeedEntry> entries, Long viewerId) {
        UserProfile viewerProfile = getUserProfile(viewerId);
        
        // Score each entry
        entries.forEach(entry -> {
            double score = calculateScore(entry, viewerProfile);
            entry.setFinalScore(score);
        });
        
        // Sort by score descending
        entries.sort(Comparator.comparingDouble(FeedEntry::getFinalScore).reversed());
        
        // Apply diversity rules (avoid too many posts from same author)
        return applyDiversity(entries);
    }
    
    private double calculateScore(FeedEntry entry, UserProfile viewer) {
        double score = 0.0;
        
        // 1. RECENCY (30% weight)
        // Exponential decay with 6-hour half-life
        long ageMinutes = entry.getAgeMinutes();
        double recencyScore = Math.exp(-ageMinutes / 360.0);
        score += recencyScore * 0.30;
        
        // 2. ENGAGEMENT (25% weight)
        // Log-normalized engagement
        double engagementScore = calculateEngagementScore(entry);
        score += engagementScore * 0.25;
        
        // 3. AFFINITY (25% weight)
        // How close is viewer to author
        double affinityScore = calculateAffinityScore(viewer, entry.getAuthorId());
        score += affinityScore * 0.25;
        
        // 4. CONTENT RELEVANCE (15% weight)
        // Does content match viewer's interests
        double relevanceScore = calculateRelevanceScore(viewer, entry);
        score += relevanceScore * 0.15;
        
        // 5. AUTHOR QUALITY (5% weight)
        // Verified, engagement rate, spam score
        double authorScore = calculateAuthorScore(entry.getAuthorId());
        score += authorScore * 0.05;
        
        return score;
    }
    
    private double calculateEngagementScore(FeedEntry entry) {
        // Weighted engagement
        double rawEngagement = 
            entry.getLikesCount() * 1.0 +
            entry.getCommentsCount() * 3.0 +
            entry.getSharesCount() * 5.0;
        
        // Log normalization to handle viral posts
        return Math.log1p(rawEngagement) / 15.0;  // Normalize to ~0-1
    }
    
    private double calculateAffinityScore(UserProfile viewer, Long authorId) {
        // Get interaction history
        InteractionStats stats = interactionService.getStats(viewer.getId(), authorId);
        
        double score = 0.0;
        
        // Recent interactions (last 30 days)
        score += Math.min(stats.getRecentLikes() / 10.0, 0.3);
        score += Math.min(stats.getRecentComments() / 5.0, 0.3);
        score += Math.min(stats.getRecentProfileViews() / 3.0, 0.2);
        
        // Relationship type
        if (stats.isCloseFriend()) {
            score += 0.2;
        }
        
        return Math.min(score, 1.0);
    }
    
    private List<FeedEntry> applyDiversity(List<FeedEntry> entries) {
        List<FeedEntry> diversified = new ArrayList<>();
        Map<Long, Integer> authorCounts = new HashMap<>();
        
        for (FeedEntry entry : entries) {
            Long authorId = entry.getAuthorId();
            int count = authorCounts.getOrDefault(authorId, 0);
            
            // Max 3 consecutive posts from same author
            if (count < 3) {
                diversified.add(entry);
                authorCounts.put(authorId, count + 1);
            } else {
                // Add to end of list
                diversified.add(diversified.size(), entry);
            }
        }
        
        return diversified;
    }
}
```

### Ranking Signal Weights

| Signal           | Weight | Why                                    |
| ---------------- | ------ | -------------------------------------- |
| Recency          | 30%    | Fresh content is more relevant         |
| Engagement       | 25%    | Popular posts are likely interesting   |
| Affinity         | 25%    | Posts from close friends matter more   |
| Content match    | 15%    | Match user's interests                 |
| Author quality   | 5%     | Verified, low spam score               |

### Ranking Formula

```
Score = 0.30 × RecencyScore +
        0.25 × EngagementScore +
        0.25 × AffinityScore +
        0.15 × ContentScore +
        0.05 × AuthorScore

Where:
- RecencyScore = e^(-age_hours / 6)  // Half-life of 6 hours
- EngagementScore = log(1 + likes + 3×comments + 5×shares) / 15
- AffinityScore = interaction_frequency with author
- ContentScore = ML model based on user preferences
- AuthorScore = verified bonus + engagement rate
```

### Real-time Engagement Updates

```java
@Service
public class EngagementService {
    
    private final RedisTemplate<String, String> redis;
    private final KafkaTemplate<String, EngagementEvent> kafka;
    
    public void recordLike(Long postId, Long userId) {
        // 1. Increment counter in Redis (real-time)
        String key = "engagement:" + postId;
        redis.opsForHash().increment(key, "likes", 1);
        
        // 2. Update user's interaction history
        String interactionKey = "interactions:" + userId + ":" + getAuthorId(postId);
        redis.opsForHash().increment(interactionKey, "likes", 1);
        redis.expire(interactionKey, 30, TimeUnit.DAYS);
        
        // 3. Publish event for async processing
        kafka.send("engagement-events", EngagementEvent.builder()
            .type("like")
            .postId(postId)
            .userId(userId)
            .timestamp(Instant.now())
            .build());
        
        // 4. Update ranking score (async)
        recalculatePostScore(postId);
    }
    
    @Async
    public void recalculatePostScore(Long postId) {
        // Get current engagement
        Map<Object, Object> engagement = redis.opsForHash()
            .entries("engagement:" + postId);
        
        int likes = Integer.parseInt((String) engagement.getOrDefault("likes", "0"));
        int comments = Integer.parseInt((String) engagement.getOrDefault("comments", "0"));
        int shares = Integer.parseInt((String) engagement.getOrDefault("shares", "0"));
        
        // Calculate new score
        double score = Math.log1p(likes + comments * 3 + shares * 5);
        
        // Update in database (periodic batch)
        postRepository.updateEngagementScore(postId, score);
    }
}
```

---

## 4. Social Graph Management

### Graph Service Implementation

```java
@Service
public class GraphService {
    
    private final FollowRepository followRepository;
    private final RedisTemplate<String, String> redis;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    
    public void follow(Long followerId, Long followingId) {
        // 1. Create follow relationship
        Follow follow = Follow.builder()
            .followerId(followerId)
            .followingId(followingId)
            .createdAt(Instant.now())
            .build();
        
        followRepository.save(follow);
        
        // 2. Update counts
        userRepository.incrementFollowersCount(followingId);
        userRepository.incrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().add("following:" + followerId, followingId.toString());
        redis.opsForSet().add("followers:" + followingId, followerId.toString());
        
        // 4. Check if now celebrity
        long followersCount = userRepository.getFollowersCount(followingId);
        if (followersCount >= CELEBRITY_THRESHOLD) {
            markAsCelebrity(followingId);
        }
        
        // 5. Backfill recent posts to follower's feed
        backfillRecentPosts(followerId, followingId);
    }
    
    public void unfollow(Long followerId, Long followingId) {
        // 1. Remove relationship
        followRepository.delete(followerId, followingId);
        
        // 2. Update counts
        userRepository.decrementFollowersCount(followingId);
        userRepository.decrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().remove("following:" + followerId, followingId.toString());
        redis.opsForSet().remove("followers:" + followingId, followerId.toString());
        
        // 4. Remove posts from feed
        removePostsFromFeed(followerId, followingId);
    }
    
    public List<Long> getFollowers(Long userId) {
        String cacheKey = "followers:" + userId;
        
        // Try cache first
        Set<String> cached = redis.opsForSet().members(cacheKey);
        if (cached != null && !cached.isEmpty()) {
            return cached.stream()
                .map(Long::parseLong)
                .collect(Collectors.toList());
        }
        
        // Load from database
        List<Long> followers = followRepository.findFollowerIds(userId);
        
        // Populate cache
        if (!followers.isEmpty()) {
            String[] followerStrs = followers.stream()
                .map(String::valueOf)
                .toArray(String[]::new);
            redis.opsForSet().add(cacheKey, followerStrs);
            redis.expire(cacheKey, CACHE_TTL);
        }
        
        return followers;
    }
    
    public List<Long> getFollowedCelebrities(Long userId) {
        // Get all users this person follows
        List<Long> following = getFollowing(userId);
        
        // Filter to celebrities only
        return following.stream()
            .filter(this::isCelebrity)
            .collect(Collectors.toList());
    }
    
    private void backfillRecentPosts(Long followerId, Long followingId) {
        // Get recent posts from new following
        List<Post> recentPosts = postRepository.findRecentByAuthor(
            followingId, 
            Instant.now().minus(7, ChronoUnit.DAYS),
            20
        );
        
        // Add to follower's feed
        for (Post post : recentPosts) {
            double score = calculatePostScore(post, followerId);
            feedCacheService.addToFeed(followerId, post.getId(), score);
        }
    }
}
```

---

## 5. Real-time Updates (WebSocket)

### WebSocket Handler

```java
@Component
public class FeedWebSocketHandler extends TextWebSocketHandler {
    
    private final Map<Long, Set<WebSocketSession>> userSessions = new ConcurrentHashMap<>();
    private final RedisMessageListenerContainer redisListener;
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long userId = extractUserId(session);
        
        // Track session
        userSessions.computeIfAbsent(userId, k -> ConcurrentHashMap.newKeySet())
            .add(session);
        
        // Subscribe to user's channel
        subscribeToUserChannel(userId);
        
        log.info("WebSocket connected for user {}", userId);
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        Long userId = extractUserId(session);
        
        Set<WebSocketSession> sessions = userSessions.get(userId);
        if (sessions != null) {
            sessions.remove(session);
            if (sessions.isEmpty()) {
                userSessions.remove(userId);
                unsubscribeFromUserChannel(userId);
            }
        }
    }
    
    public void sendNewPostNotification(Long userId, Post post) {
        Set<WebSocketSession> sessions = userSessions.get(userId);
        if (sessions == null || sessions.isEmpty()) {
            return;
        }
        
        String message = objectMapper.writeValueAsString(Map.of(
            "type", "new_post",
            "post", Map.of(
                "id", post.getId(),
                "author", post.getAuthor().getDisplayName(),
                "preview", truncate(post.getContent(), 100),
                "timestamp", post.getCreatedAt()
            )
        ));
        
        sessions.forEach(session -> {
            try {
                session.sendMessage(new TextMessage(message));
            } catch (IOException e) {
                log.warn("Failed to send message to session", e);
            }
        });
    }
    
    private void subscribeToUserChannel(Long userId) {
        String channel = "user:" + userId + ":feed";
        
        redisListener.addMessageListener(
            (message, pattern) -> {
                String payload = new String(message.getBody());
                handleRedisMessage(userId, payload);
            },
            new ChannelTopic(channel)
        );
    }
}
```

### Push Notification Service

```java
@Service
public class PushNotificationService {
    
    private final FirebaseMessaging firebaseMessaging;
    private final APNsService apnsService;
    private final UserDeviceRepository deviceRepository;
    
    @Async
    public void notifyNewPost(Long userId, Post post) {
        // Check user's notification preferences
        NotificationPreferences prefs = getPreferences(userId);
        if (!prefs.isNewPostsEnabled()) {
            return;
        }
        
        // Check if user is currently active (don't push if app is open)
        if (isUserCurrentlyActive(userId)) {
            return;
        }
        
        // Get user's devices
        List<UserDevice> devices = deviceRepository.findByUserId(userId);
        
        // Build notification
        String title = post.getAuthor().getDisplayName();
        String body = truncate(post.getContent(), 100);
        
        for (UserDevice device : devices) {
            try {
                if (device.getPlatform().equals("ios")) {
                    sendAPNs(device.getToken(), title, body, post.getId());
                } else {
                    sendFCM(device.getToken(), title, body, post.getId());
                }
            } catch (Exception e) {
                log.warn("Failed to send push to device {}", device.getId(), e);
                handleInvalidToken(device);
            }
        }
    }
    
    private void sendFCM(String token, String title, String body, Long postId) {
        Message message = Message.builder()
            .setToken(token)
            .setNotification(Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build())
            .putData("post_id", postId.toString())
            .putData("type", "new_post")
            .build();
        
        firebaseMessaging.send(message);
    }
}
```

---

## 6. Search

**Not Applicable for News Feed**

News feed systems are fundamentally different from search systems:
- **Feeds are personalized**: Each user sees different content based on who they follow
- **No query required**: Content is pushed based on social graph, not pulled via search
- **Ranking is proactive**: Posts are scored and ordered before the user requests them

For search functionality within a social platform (e.g., searching for users, posts, or hashtags), see the **Search Engine** HLD design which covers inverted indexes, query processing, and relevance ranking.

---

## Summary

| Component       | Technology/Algorithm          | Key Configuration                |
| --------------- | ----------------------------- | -------------------------------- |
| Fan-out         | Kafka + Hybrid strategy       | 10K follower threshold           |
| Ranking         | Multi-signal scoring          | 5 signals, weighted combination  |
| Feed cache      | Redis Sorted Sets             | 500 posts, 24h TTL               |
| Real-time       | WebSocket + Redis Pub/Sub     | Per-user channels                |
| Social graph    | PostgreSQL + Redis cache      | 1h cache TTL                     |

