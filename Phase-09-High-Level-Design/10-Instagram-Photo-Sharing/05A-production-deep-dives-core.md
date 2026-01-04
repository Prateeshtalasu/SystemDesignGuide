# Instagram / Photo Sharing - Production Deep Dives (Core)

## Overview

This document covers the technical implementation details for key components: feed generation, image processing, stories, and the social graph.

---

## 1. Feed Generation Deep Dive

### Hybrid Fan-out Implementation

```java
@Service
public class FeedService {
    
    private final RedisTemplate<String, String> redis;
    private final PostRepository postRepository;
    private final FollowRepository followRepository;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final int FEED_SIZE = 500;
    
    /**
     * Fan-out on write for regular users
     */
    @KafkaListener(topics = "new-post-events")
    public void handleNewPost(PostEvent event) {
        Post post = event.getPost();
        User author = post.getAuthor();
        
        if (author.getFollowerCount() > CELEBRITY_THRESHOLD) {
            // Celebrity: Don't fan-out, store in celebrity posts
            redis.opsForZSet().add(
                "celebrity_posts:" + author.getId(),
                post.getId(),
                post.getCreatedAt().toEpochMilli()
            );
            return;
        }
        
        // Regular user: Fan-out to all followers
        List<Long> followers = followRepository.getFollowerIds(author.getId());
        
        // Batch process
        Lists.partition(followers, 1000).forEach(batch -> {
            fanOutToFollowers(batch, post);
        });
    }
    
    private void fanOutToFollowers(List<Long> followers, Post post) {
        redis.executePipelined((RedisCallback<Object>) connection -> {
            for (Long followerId : followers) {
                byte[] key = ("feed:" + followerId).getBytes();
                connection.zAdd(key, post.getCreatedAt().toEpochMilli(), 
                    post.getId().getBytes());
                
                // Trim to max size
                connection.zRemRangeByRank(key, 0, -FEED_SIZE - 1);
            }
            return null;
        });
    }
    
    /**
     * Get feed with celebrity posts merged
     */
    public List<Post> getFeed(Long userId, String cursor, int limit) {
        // 1. Get precomputed feed
        Set<String> feedPostIds = redis.opsForZSet()
            .reverseRangeByScore("feed:" + userId, 0, Double.MAX_VALUE, 0, limit * 2);
        
        // 2. Get celebrity posts (fan-out on read)
        List<Long> followedCelebrities = followRepository.getFollowedCelebrities(userId);
        Set<String> celebrityPostIds = new HashSet<>();
        
        for (Long celebrityId : followedCelebrities) {
            Set<String> posts = redis.opsForZSet()
                .reverseRange("celebrity_posts:" + celebrityId, 0, 20);
            celebrityPostIds.addAll(posts);
        }
        
        // 3. Merge and rank
        Set<String> allPostIds = new HashSet<>();
        allPostIds.addAll(feedPostIds);
        allPostIds.addAll(celebrityPostIds);
        
        List<Post> posts = postRepository.findByIds(allPostIds);
        
        // 4. Apply ranking
        return rankPosts(posts, userId).stream()
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    private List<Post> rankPosts(List<Post> posts, Long userId) {
        return posts.stream()
            .map(post -> new ScoredPost(post, calculateScore(post, userId)))
            .sorted(Comparator.comparingDouble(ScoredPost::getScore).reversed())
            .map(ScoredPost::getPost)
            .collect(Collectors.toList());
    }
    
    private double calculateScore(Post post, Long userId) {
        double score = 0.0;
        
        // Recency (exponential decay, half-life 6 hours)
        long ageHours = ChronoUnit.HOURS.between(post.getCreatedAt(), Instant.now());
        score += Math.exp(-ageHours / 6.0) * 0.4;
        
        // Engagement
        double engagement = Math.log1p(post.getLikeCount() + post.getCommentCount() * 2);
        score += (engagement / 20.0) * 0.3;
        
        // Relationship strength
        double relationship = getRelationshipStrength(userId, post.getAuthorId());
        score += relationship * 0.3;
        
        return score;
    }
}
```

### Feed Ranking Algorithm

```java
@Service
public class FeedRankingService {
    
    private final InteractionService interactionService;
    
    public double calculateFeedScore(Post post, User viewer) {
        FeedScore score = new FeedScore();
        
        // 1. INTEREST SCORE (40%)
        // How likely is the viewer interested in this content?
        score.interest = calculateInterestScore(post, viewer);
        
        // 2. RELATIONSHIP SCORE (30%)
        // How close is the viewer to the author?
        score.relationship = calculateRelationshipScore(viewer, post.getAuthor());
        
        // 3. TIMELINESS SCORE (20%)
        // How fresh is the content?
        score.timeliness = calculateTimelinessScore(post);
        
        // 4. ENGAGEMENT SCORE (10%)
        // How well is the post performing?
        score.engagement = calculateEngagementScore(post);
        
        return score.interest * 0.4 +
               score.relationship * 0.3 +
               score.timeliness * 0.2 +
               score.engagement * 0.1;
    }
    
    private double calculateRelationshipScore(User viewer, User author) {
        InteractionStats stats = interactionService.getStats(viewer.getId(), author.getId());
        
        double score = 0.0;
        
        // Likes given to author
        score += Math.min(stats.getLikesGiven() / 50.0, 0.3);
        
        // Comments on author's posts
        score += Math.min(stats.getCommentsGiven() / 20.0, 0.3);
        
        // Profile visits
        score += Math.min(stats.getProfileVisits() / 10.0, 0.2);
        
        // DM history
        if (stats.hasDMHistory()) {
            score += 0.2;
        }
        
        return Math.min(score, 1.0);
    }
    
    private double calculateTimelinessScore(Post post) {
        long ageMinutes = ChronoUnit.MINUTES.between(post.getCreatedAt(), Instant.now());
        
        // Exponential decay with 6-hour half-life
        return Math.exp(-ageMinutes / 360.0);
    }
}
```

---

## 1. Kafka for Fan-out (Async Messaging)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform. For Instagram, Kafka handles fan-out of posts to followers' feeds asynchronously, allowing the post creation API to return quickly while feed updates happen in the background.

**What problems does Kafka solves here?**

1. **Low latency**: Post creation returns in < 100ms (doesn't wait for fan-out)
2. **Decoupling**: Feed service scales independently from post service
3. **Reliability**: Fan-out jobs persist even if workers crash
4. **Buffering**: Handles traffic spikes (celebrity posts with millions of followers)

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

```
Topic: new-post-events
Partitions: 32
Replication Factor: 3
Retention: 7 days
```

**Partition Key Choice:**

We partition by `author_id` hash. This ensures:
- All posts from the same author go to the same partition
- Ordering is preserved per author (posts appear chronologically)
- Even distribution across partitions

**Consumer Group Design:**

```
Consumer Group: feed-fanout-workers
Consumers: 32 instances (one per partition)
Processing: Batch processing of 500-1000 followers
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Post Creation and Fan-out**

```
Step 1: User Creates Post
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/posts                                               │
│ Author: user_123 (5,000 followers)                          │
│ Image: uploaded to S3                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Post Service
┌─────────────────────────────────────────────────────────────┐
│ 1. Validate image                                            │
│ 2. Store in PostgreSQL (posts table)                        │
│ 3. Generate post_id: "post_abc123"                            │
│ 4. Publish to Kafka:                                          │
│    Topic: new-post-events                                    │
│    Partition: hash(user_123) % 32 = partition 7             │
│    Message: {                                                │
│      "event_type": "POST_CREATED",                           │
│      "post_id": "post_abc123",                              │
│      "author_id": "user_123",                               │
│      "follower_count": 5000,                                │
│      "timestamp": "2024-01-20T10:00:00Z"                    │
│    }                                                          │
│ 5. Return 201 Created (< 100ms)                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Fan-out Worker (consumes from Kafka)
┌─────────────────────────────────────────────────────────────┐
│ Worker 7 (assigned to partition 7) polls:                  │
│ - Receives batch of 1 event                                  │
│ - Checks follower count: 5,000 < 10,000 (not celebrity)    │
│ - Fetches follower list from Graph Service                  │
│ - Processes in batches of 1000:                             │
│   Batch 1: followers 0-999                                  │
│   Batch 2: followers 1000-1999                              │
│   Batch 3: followers 2000-2999                              │
│   Batch 4: followers 3000-3999                              │
│   Batch 5: followers 4000-4999                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Redis Writes (per batch)
┌─────────────────────────────────────────────────────────────┐
│ For each follower in batch:                                 │
│   ZADD feed:follower_1 0.95 post_abc123                    │
│   ZADD feed:follower_2 0.92 post_abc123                     │
│   ...                                                        │
│                                                              │
│ Pipeline execution: ~50ms per batch of 1000                │
│ Total fan-out time: ~250ms for 5,000 followers              │
│ Commits Kafka offset after all batches complete             │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Worker Crash During Fan-out**

```
Scenario: Worker crashes after processing 3,000 of 5,000 followers

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Worker 7 starts fan-out for post_abc123               │
│ T+50ms: Batch 1 complete (followers 0-999)                  │
│ T+100ms: Batch 2 complete (followers 1000-1999)            │
│ T+150ms: Batch 3 complete (followers 2000-2999)            │
│ T+151ms: Worker crashes (OOM, network issue)               │
│ T+151ms: Kafka offset NOT committed                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+60s: Kafka consumer group rebalance                      │
│ - Coordinator detects worker 7 left group                   │
│ - Reassigns partition 7 to worker 8                         │
│ - Worker 8 starts from last committed offset               │
│ - Receives post_abc123 event again                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Worker 8 Processing:                                        │
│ - Fetches follower list: 5,000 followers                   │
│ - Processes all 5 batches                                   │
│ - Redis ZADD operations are idempotent:                    │
│   ZADD feed:follower_1 0.95 post_abc123                    │
│   (If already exists, updates score - no duplicate)         │
│ - All 5,000 followers get post in feed                      │
│ - Commits offset                                            │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same post event processed twice (worker crash + retry)

Solution: Idempotent Redis operations
1. ZADD is idempotent: Same post_id in feed updates score, doesn't duplicate
2. Celebrity posts: Stored in sorted set, idempotent adds
3. Story tray updates: ZADD is idempotent

Example:
- Post event processed twice → ZADD called twice → Same result (idempotent)
- No duplicate posts in feeds
```

**Traffic Spike Handling:**

```
Scenario: Celebrity with 10M followers posts photo

┌─────────────────────────────────────────────────────────────┐
│ Normal: Regular user with 100 followers                     │
│ - Fan-out: 100 Redis ZADD operations (~5ms)                 │
│                                                              │
│ Spike: Celebrity with 10M followers                        │
│ - Celebrity threshold: 10,000 followers                   │
│ - Strategy: Fan-out on read (not on write)                 │
│ - Post stored in celebrity_posts:{author_id} sorted set    │
│ - No fan-out to individual feeds                            │
│ - When user requests feed: Merge celebrity posts            │
└─────────────────────────────────────────────────────────────┘
```

### Hot Partition Mitigation

**Problem:** If a celebrity posts frequently, one Kafka partition could become hot, creating a bottleneck for fan-out processing.

**Example Scenario:**
```
Normal user: 1 post/hour → Partition 15 (hash(author_id) % partitions)
Celebrity: 10 posts/hour → Partition 15 (same partition!)
Result: Partition 15 receives 10x traffic, becomes bottleneck
Fan-out workers processing partition 15 become overwhelmed
```

**Detection:**
- Monitor partition lag per partition (Kafka metrics)
- Alert when partition lag > 2x average
- Track partition throughput metrics (posts/second per partition)
- Monitor consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Mitigation Strategies:**

1. **Hybrid Fan-out Strategy (Primary Mitigation):**
   - Regular users (< 10K followers): Fan-out on write (pre-compute feeds)
   - Celebrities (> 10K followers): Fan-out on read (don't pre-compute, merge at query time)
   - This prevents celebrity posts from overwhelming fan-out workers
   - Celebrity posts stored in `celebrity_posts:{author_id}` sorted set
   - No fan-out to individual feeds for celebrities

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute celebrity posts across multiple partitions using composite key

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

**Recovery Behavior:**

```
Auto-healing:
- Worker crash → Kafka rebalance → Event reassigned → Retry
- Redis failure → Circuit breaker → Fallback to DB → Retry
- Network timeout → Exponential backoff → Retry

Human intervention:
- Fan-out stuck > 1 hour → Manual investigation
- Repeated failures → Check follower list service
- Redis cluster issues → Scale Redis or investigate
```

---

## 1.5. Redis for Feed Caching

### A) CONCEPT: What is Redis?

Redis is an in-memory data structure store. For Instagram feeds, Redis caches pre-computed feeds to avoid expensive database queries and feed generation on every request.

**What problems does Redis solve here?**

1. **Low latency**: Feed retrieval in < 10ms (vs 100-200ms from DB)
2. **Reduced load**: Avoids querying follower lists and posts for every feed request
3. **Scalability**: Handles millions of feed requests per second
4. **Real-time updates**: Sorted sets enable efficient feed updates

### B) OUR USAGE: How We Use Redis Here

**Feed Storage (Sorted Sets):**

```
Key: feed:{user_id}
Value: Sorted set of post IDs
Score: Feed score (recency + engagement + relationship)
TTL: None (manually managed, trimmed to 500 posts)
```

**Celebrity Posts (Sorted Sets):**

```
Key: celebrity_posts:{author_id}
Value: Sorted set of post IDs
Score: Timestamp (milliseconds)
TTL: None (trimmed to 100 posts)
```

**Following/Followers Cache (Sets):**

```
Key: following:{user_id}
Value: Set of user IDs being followed
TTL: 1 hour

Key: followers:{user_id}
Value: Set of user IDs following
TTL: 1 hour
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Feed Retrieval (Cache Hit)**

```
Step 1: User Requests Feed
┌─────────────────────────────────────────────────────────────┐
│ GET /v1/feed?cursor=abc123&limit=20                        │
│ User ID: user_456                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Redis Cache Check
┌─────────────────────────────────────────────────────────────┐
│ Redis: ZREVRANGE feed:user_456 0 19                        │
│ Returns: [post_123, post_456, post_789, ...] (20 post IDs) │
│ Latency: ~2ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Fetch Post Details
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT * FROM posts WHERE id IN (...)           │
│ Returns: Post objects with metadata                          │
│ Latency: ~5ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Merge Celebrity Posts (if following celebrities)
┌─────────────────────────────────────────────────────────────┐
│ 1. Get followed celebrities: [user_999, user_888]          │
│ 2. Redis: ZREVRANGE celebrity_posts:user_999 0 20          │
│ 3. Redis: ZREVRANGE celebrity_posts:user_888 0 20          │
│ 4. Merge with feed posts                                    │
│ 5. Re-rank by score                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Return Response
┌─────────────────────────────────────────────────────────────┐
│ 200 OK with 20 posts                                        │
│ Total latency: ~15ms                                        │
└─────────────────────────────────────────────────────────────┘
```

**Cache Miss Flow: Feed Generation**

```
Step 1: Cache Miss
┌─────────────────────────────────────────────────────────────┐
│ Redis: ZREVRANGE feed:user_456 0 19 → (empty)              │
│ Cache miss detected                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Generate Feed
┌─────────────────────────────────────────────────────────────┐
│ 1. Get following list: following:user_456 (Redis cache)   │
│ 2. If cache miss: Query PostgreSQL                         │
│ 3. Get recent posts from followed users                     │
│ 4. Calculate feed scores                                    │
│ 5. Sort by score                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Cache Feed
┌─────────────────────────────────────────────────────────────┐
│ Redis: ZADD feed:user_456 {score} {post_id} (pipeline)     │
│ Add top 500 posts to feed cache                             │
│ Trim to 500: ZREMRANGEBYRANK feed:user_456 500 -1          │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Redis Cluster Node Failure**

```
Scenario: Redis node handling slot 5461-10922 crashes

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Redis node 2 crashes                                  │
│ T+0-5s: Requests to affected slots fail                    │
│   - feed:user_456 (hash in affected slot) → Error          │
│   - following:user_456 → Error                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+5s: Redis cluster detects failure                          │
│ T+10s: Replica promoted to primary for affected slots      │
│ T+15s: Cluster topology updated                              │
│ T+20s: All requests succeeding again                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Impact:                                                      │
│ - Feed requests fail for ~1/3 of users (15-20 seconds)     │
│ - Circuit breaker opens, falls back to PostgreSQL           │
│ - Feed generation from DB (slower but works)                │
│ - Cache repopulates on demand                                │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same post added to feed multiple times (fan-out retry)

Solution: ZADD is idempotent
- ZADD feed:user_456 0.95 post_abc123
- If post_abc123 already exists, updates score (no duplicate)
- No duplicate posts in feeds
```

**Traffic Spike Handling:**

```
Scenario: Celebrity posts, millions request feed simultaneously

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 feed requests/second                          │
│ Spike: 100,000 feed requests/second (viral post)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Redis Handling:                                             │
│ - Sorted set operations: O(log N) per operation            │
│ - Pipeline operations: Batch 100 operations                │
│ - Read replicas: Distribute read load                       │
│ - Result: 100K ops/sec handled by Redis cluster            │
│                                                              │
│ Cache Hit Rate:                                             │
│ - Pre-computed feeds: 95% hit rate                          │
│ - Celebrity posts: Cached in separate sorted set            │
│ - Result: Most requests served from cache                  │
└─────────────────────────────────────────────────────────────┘
```

**Hot Key Mitigation:**

```
Problem: Popular user's feed requested by millions

Mitigation:
1. Feed pre-computation: Generate feed before requests
2. Read replicas: Distribute reads across replicas
3. Local cache: Cache hottest feeds in application memory
4. CDN: For public feeds (if applicable)

If hot key occurs:
- Monitor key access patterns
- Add read replicas
- Consider feed pre-warming for popular users
```

**Cache Stampede Prevention:**

```
Problem: Celebrity posts, millions of followers' feed caches expire simultaneously

Scenario:
T+0s:   Celebrity posts (10M followers)
T+0s:   Millions of users open app (feed cache expired)
T+0s:   Millions of feed regeneration requests hit database
T+1s:   Database overwhelmed, latency spikes to seconds
```

**Prevention Strategy: Distributed Lock + Probabilistic Early Expiration**

```java
@Service
public class FeedService {
    
    private final RedisTemplate<String, String> redis;
    private final DistributedLock lockService;
    
    private static final double EARLY_EXPIRATION_PROBABILITY = 0.01; // 1%
    
    public List<Post> getFeed(Long userId, int limit) {
        // 1. Try cache first
        Set<String> cachedPostIds = redis.opsForZSet()
            .reverseRange("feed:" + userId, 0, limit * 2 - 1);
        
        if (!cachedPostIds.isEmpty()) {
            // Probabilistic early expiration (1% chance)
            if (Math.random() < EARLY_EXPIRATION_PROBABILITY) {
                refreshFeedAsync(userId);
            }
            return fetchPostsByIds(cachedPostIds, limit);
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:feed:" + userId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(10));
        
        if (!acquired) {
            // Another thread is regenerating, wait briefly
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
│ - Database: Only 1 query (not 1M)                         │
│ - Latency: 20ms (cache hit) for 99.99% of requests         │
└─────────────────────────────────────────────────────────────┘
```

**Recovery Behavior:**

```
Auto-healing:
- Redis node failure → Cluster promotes replica → Automatic
- Network partition → Read from replica → Automatic
- Cache miss → Generate feed → Repopulate cache → Automatic
- Cache stampede → Distributed lock prevents thundering herd → Automatic

Human intervention:
- Cluster-wide failure → Manual failover to DR region
- Persistent hot keys → Scale Redis or optimize access pattern
```

---

## 2. Image Processing Deep Dive

### Image Processing Service

```java
@Service
public class ImageProcessingService {
    
    private final S3Client s3Client;
    private final KafkaTemplate<String, ProcessingResult> kafka;
    
    private static final List<ImageSize> SIZES = List.of(
        new ImageSize("thumb", 150, 150),
        new ImageSize("medium", 640, 640),
        new ImageSize("large", 1080, 1080)
    );
    
    @KafkaListener(topics = "image-processing-jobs")
    public void processImage(ImageProcessingJob job) {
        try {
            // 1. Download original
            byte[] original = downloadFromS3(job.getOriginalPath());
            
            // 2. Validate
            validateImage(original);
            
            // 3. Extract metadata
            ImageMetadata metadata = extractMetadata(original);
            
            // 4. Apply filter if specified
            byte[] filtered = job.getFilter() != null 
                ? applyFilter(original, job.getFilter())
                : original;
            
            // 5. Generate sizes
            Map<String, byte[]> resized = new HashMap<>();
            for (ImageSize size : SIZES) {
                resized.put(size.getName(), resize(filtered, size));
            }
            
            // 6. Generate blurhash
            String blurhash = generateBlurhash(resized.get("thumb"));
            
            // 7. Upload all sizes to S3
            for (Map.Entry<String, byte[]> entry : resized.entrySet()) {
                uploadToS3(job.getPostId(), entry.getKey(), entry.getValue());
            }
            
            // 8. Publish success event
            kafka.send("image-processing-results", ProcessingResult.success(
                job.getPostId(),
                generateUrls(job.getPostId()),
                blurhash,
                metadata
            ));
            
        } catch (Exception e) {
            log.error("Image processing failed for {}", job.getPostId(), e);
            kafka.send("image-processing-results", ProcessingResult.failure(
                job.getPostId(), e.getMessage()
            ));
        }
    }
    
    private byte[] applyFilter(byte[] image, String filterName) {
        BufferedImage buffered = toBufferedImage(image);
        
        switch (filterName.toLowerCase()) {
            case "clarendon":
                return applyClarendon(buffered);
            case "gingham":
                return applyGingham(buffered);
            case "moon":
                return applyMoon(buffered);
            case "lark":
                return applyLark(buffered);
            default:
                return image;
        }
    }
    
    private byte[] applyClarendon(BufferedImage image) {
        // Increase contrast and saturation
        BufferedImage result = new BufferedImage(
            image.getWidth(), image.getHeight(), BufferedImage.TYPE_INT_RGB);
        
        for (int y = 0; y < image.getHeight(); y++) {
            for (int x = 0; x < image.getWidth(); x++) {
                int rgb = image.getRGB(x, y);
                
                int r = (rgb >> 16) & 0xFF;
                int g = (rgb >> 8) & 0xFF;
                int b = rgb & 0xFF;
                
                // Increase contrast
                r = clamp((int) ((r - 128) * 1.2 + 128));
                g = clamp((int) ((g - 128) * 1.2 + 128));
                b = clamp((int) ((b - 128) * 1.2 + 128));
                
                // Slight warm tint
                r = clamp(r + 10);
                
                result.setRGB(x, y, (r << 16) | (g << 8) | b);
            }
        }
        
        return toBytes(result);
    }
    
    private String generateBlurhash(byte[] thumbnail) {
        BufferedImage image = toBufferedImage(thumbnail);
        int[] pixels = image.getRGB(0, 0, image.getWidth(), image.getHeight(), 
            null, 0, image.getWidth());
        
        return BlurHash.encode(pixels, image.getWidth(), image.getHeight(), 4, 3);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Image Processing Technology-Level

**Normal Flow: Image Upload and Processing**

```
Request: POST /v1/posts (with image file)
    ↓
Step 1: Image Upload
┌─────────────────────────────────────────────────────────────┐
│ Client: Uploads 5MB image file                             │
│ API Service: Receives multipart/form-data                  │
│ Validation: Check file size, format (JPEG/PNG)             │
│ Storage: Upload original to S3 (temp location)             │
│ S3 Key: temp/post_abc123_original.jpg                     │
│ Latency: ~500ms (S3 upload)                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Create Processing Job
┌─────────────────────────────────────────────────────────────┐
│ API Service: Create image processing job                   │
│ Job: {                                                       │
│   post_id: "post_abc123",                                  │
│   original_path: "temp/post_abc123_original.jpg",         │
│   filter: "clarendon",                                      │
│   status: "pending"                                         │
│ }                                                           │
│ Kafka: Publish to image-processing-jobs topic              │
│ Partition: hash(post_id) % 100                             │
│ Latency: ~10ms (Kafka publish)                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Image Processing Worker Consumes Job
┌─────────────────────────────────────────────────────────────┐
│ Worker: Consumes job from Kafka                            │
│ T+0ms: Download original from S3                           │
│ T+500ms: Original downloaded (5MB)                        │
│ T+501ms: Validate image format                             │
│ T+502ms: Extract metadata (dimensions, EXIF)                │
│ T+510ms: Apply filter "clarendon"                           │
│ T+800ms: Filter applied                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Generate Multiple Sizes
┌─────────────────────────────────────────────────────────────┐
│ T+801ms: Resize to thumbnail (150x150)                     │
│ T+850ms: Thumbnail generated                                │
│ T+851ms: Resize to medium (640x640)                         │
│ T+950ms: Medium generated                                   │
│ T+951ms: Resize to large (1080x1080)                        │
│ T+1100ms: Large generated                                   │
│ T+1101ms: Generate blurhash from thumbnail                  │
│ T+1110ms: Blurhash: "LGF5]+Yk^6#M@-5c,1J5@[or[Q6."         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Upload Processed Images
┌─────────────────────────────────────────────────────────────┐
│ T+1111ms: Upload thumbnail to S3                           │
│ S3 Key: posts/post_abc123/thumb.jpg                        │
│ T+1200ms: Upload medium to S3                              │
│ S3 Key: posts/post_abc123/medium.jpg                      │
│ T+1300ms: Upload large to S3                                │
│ S3 Key: posts/post_abc123/large.jpg                        │
│ T+1400ms: All sizes uploaded                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Publish Success Event
┌─────────────────────────────────────────────────────────────┐
│ T+1401ms: Publish to image-processing-results topic        │
│ Event: {                                                     │
│   post_id: "post_abc123",                                  │
│   status: "success",                                        │
│   urls: {                                                    │
│     thumb: "https://cdn.instagram.com/.../thumb.jpg",    │
│     medium: "https://cdn.instagram.com/.../medium.jpg",   │
│     large: "https://cdn.instagram.com/.../large.jpg"       │
│   },                                                         │
│   blurhash: "LGF5]+Yk^6#M@-5c,1J5@[or[Q6."                │
│ }                                                           │
│ Total processing time: ~1.4 seconds                        │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Image Processing Failure**

```
Step 1: Processing Attempt
┌─────────────────────────────────────────────────────────────┐
│ Worker: Consumes job from Kafka                             │
│ T+0ms: Download original from S3                           │
│ T+500ms: Original downloaded                               │
│ T+501ms: Validate image format                             │
│ T+502ms: ERROR - Invalid image format (corrupted file)      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Error Handling
┌─────────────────────────────────────────────────────────────┐
│ Worker: Catch exception                                     │
│ Action: Publish failure event to Kafka                      │
│ Event: {                                                     │
│   post_id: "post_abc123",                                  │
│   status: "failed",                                         │
│   error: "Invalid image format"                            │
│ }                                                           │
│ Retry: Job retried 3 times (exponential backoff)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Dead Letter Queue
┌─────────────────────────────────────────────────────────────┐
│ After 3 retries: Move to DLQ                               │
│ DLQ Topic: image-processing-jobs-dlq                       │
│ Alert: On-call engineer notified                            │
│ Manual: Engineer investigates and fixes                    │
└─────────────────────────────────────────────────────────────┘
```

**Duplicate/Idempotency Handling:**

```
Scenario: Same image processed twice (retry)
    ↓
Step 1: First Processing
┌─────────────────────────────────────────────────────────────┐
│ Job 1: Process post_abc123                                  │
│ Result: Success, images uploaded to S3                     │
│ S3 Keys: posts/post_abc123/thumb.jpg, etc.                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Retry (Duplicate Job)
┌─────────────────────────────────────────────────────────────┐
│ Job 2: Process post_abc123 (retry)                        │
│ Check: S3 key exists (posts/post_abc123/thumb.jpg)         │
│ Result: Images already exist                                │
│ Action: Skip processing, return existing URLs              │
│ Idempotent: Same result, no duplicate processing           │
└─────────────────────────────────────────────────────────────┘
```

**Traffic Spike Handling: Celebrity Photo Upload**

```
Scenario: Celebrity uploads photo, 10M followers need feed update
    ↓
Step 1: Initial Processing
┌─────────────────────────────────────────────────────────────┐
│ T+0s: Photo uploaded, processing job created               │
│ T+1.4s: Image processed, sizes generated                    │
│ T+2s: Fan-out job created (10M followers)                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: CDN Caching
┌─────────────────────────────────────────────────────────────┐
│ T+3s: First 1000 feed requests hit CDN                     │
│       - Cache MISS (new photo)                             │
│       - CDN fetches from origin                             │
│       - Origin serves from S3                               │
│ T+5s: CDN caches all sizes                                  │
│ T+10s: 95% of requests served from CDN cache               │
│        - Edge hit rate: 95%                                 │
│        - Origin load: 5% of requests                        │
└─────────────────────────────────────────────────────────────┘
```

**Recovery Behavior: Processing Worker Failure**

```
T+0s:    Processing worker crashes during resize
    ↓
T+5s:    Kafka detects consumer heartbeat timeout
    ↓
T+10s:   Kafka triggers consumer group rebalance
    ↓
T+15s:   Job reassigned to another worker
    ↓
T+20s:   Worker resumes from last committed offset
    ↓
T+25s:   Processing resumes, no job loss
    ↓
RTO: < 30 seconds
RPO: 0 (jobs in Kafka, no loss)
```

### Progressive Image Loading

```javascript
// Client-side progressive loading
class ProgressiveImageLoader {
    
    loadImage(imageData) {
        const container = document.createElement('div');
        
        // 1. Show blurhash placeholder immediately
        const placeholder = this.renderBlurhash(imageData.blurhash);
        container.appendChild(placeholder);
        
        // 2. Load thumbnail
        const thumbnail = new Image();
        thumbnail.src = imageData.thumbnailUrl;
        thumbnail.onload = () => {
            placeholder.style.backgroundImage = `url(${thumbnail.src})`;
            placeholder.classList.add('thumbnail-loaded');
        };
        
        // 3. Load full image
        const fullImage = new Image();
        fullImage.src = imageData.mediumUrl;
        fullImage.onload = () => {
            placeholder.style.backgroundImage = `url(${fullImage.src})`;
            placeholder.classList.add('full-loaded');
        };
        
        return container;
    }
    
    renderBlurhash(blurhash) {
        const pixels = BlurHash.decode(blurhash, 32, 32);
        const canvas = document.createElement('canvas');
        canvas.width = 32;
        canvas.height = 32;
        
        const ctx = canvas.getContext('2d');
        const imageData = ctx.createImageData(32, 32);
        imageData.data.set(pixels);
        ctx.putImageData(imageData, 0, 0);
        
        const div = document.createElement('div');
        div.style.backgroundImage = `url(${canvas.toDataURL()})`;
        div.style.backgroundSize = 'cover';
        div.classList.add('blurhash-placeholder');
        
        return div;
    }
}
```

---

## 3. Stories Implementation

### Story Service

```java
@Service
public class StoryService {
    
    private final CassandraTemplate cassandra;
    private final RedisTemplate<String, String> redis;
    
    private static final Duration STORY_TTL = Duration.ofHours(24);
    
    public Story createStory(Long userId, StoryRequest request) {
        // 1. Process media
        String mediaUrl = processStoryMedia(request.getMedia());
        
        // 2. Create story with TTL
        Story story = Story.builder()
            .storyId(generateStoryId())
            .userId(userId)
            .mediaType(request.getMediaType())
            .mediaUrl(mediaUrl)
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(STORY_TTL))
            .build();
        
        // Insert with TTL
        cassandra.insert(story, InsertOptions.builder()
            .ttl((int) STORY_TTL.toSeconds())
            .build());
        
        // 3. Update story tray for followers
        updateStoryTray(userId);
        
        // 4. Send notifications to close friends
        notifyCloseFriends(userId, story);
        
        return story;
    }
    
    private void updateStoryTray(Long userId) {
        List<Long> followers = followRepository.getFollowerIds(userId);
        
        long timestamp = System.currentTimeMillis();
        
        redis.executePipelined((RedisCallback<Object>) connection -> {
            for (Long followerId : followers) {
                // Add to unseen stories
                byte[] key = ("story_tray:" + followerId).getBytes();
                connection.zAdd(key, timestamp, userId.toString().getBytes());
                
                // Set TTL on the key
                connection.expire(key, STORY_TTL.toSeconds());
            }
            return null;
        });
    }
    
    public StoryTray getStoryTray(Long userId) {
        // Get users with active stories
        Set<ZSetOperations.TypedTuple<String>> storyOwners = redis.opsForZSet()
            .reverseRangeWithScores("story_tray:" + userId, 0, 50);
        
        // Get seen stories
        Set<String> seenStoryOwners = redis.opsForSet()
            .members("seen_stories:" + userId);
        
        List<StoryTrayItem> items = new ArrayList<>();
        
        for (ZSetOperations.TypedTuple<String> owner : storyOwners) {
            Long ownerId = Long.parseLong(owner.getValue());
            
            // Get stories for this user
            List<Story> stories = cassandra.select(
                Query.query(Criteria.where("user_id").is(ownerId)),
                Story.class
            );
            
            if (!stories.isEmpty()) {
                boolean hasUnseen = !seenStoryOwners.contains(owner.getValue());
                
                items.add(StoryTrayItem.builder()
                    .user(userService.getUser(ownerId))
                    .stories(stories)
                    .hasUnseen(hasUnseen)
                    .latestStoryAt(Instant.ofEpochMilli(owner.getScore().longValue()))
                    .build());
            }
        }
        
        // Sort: unseen first, then by recency
        items.sort((a, b) -> {
            if (a.isHasUnseen() != b.isHasUnseen()) {
                return a.isHasUnseen() ? -1 : 1;
            }
            return b.getLatestStoryAt().compareTo(a.getLatestStoryAt());
        });
        
        return new StoryTray(items);
    }
    
    public void markStorySeen(Long userId, Long storyOwnerId, String storyId) {
        // Record view
        cassandra.insert(StoryView.builder()
            .storyId(storyId)
            .viewerId(userId)
            .viewedAt(Instant.now())
            .build(),
            InsertOptions.builder().ttl((int) STORY_TTL.toSeconds()).build()
        );
        
        // Update seen set
        redis.opsForSet().add("seen_stories:" + userId, storyOwnerId.toString());
        redis.expire("seen_stories:" + userId, STORY_TTL);
        
        // Increment view count
        cassandra.update(
            Query.query(Criteria.where("story_id").is(storyId)),
            Update.update("view_count", 1),
            Story.class
        );
    }
}
```

---

## 4. Social Graph Service

### Follow Service

```java
@Service
public class SocialGraphService {
    
    private final FollowRepository followRepository;
    private final RedisTemplate<String, String> redis;
    private final FeedService feedService;
    
    @Transactional
    public void follow(Long followerId, Long followingId) {
        // 1. Create follow relationship
        Follow follow = Follow.builder()
            .followerId(followerId)
            .followingId(followingId)
            .createdAt(Instant.now())
            .build();
        
        followRepository.save(follow);
        
        // 2. Update counts
        userRepository.incrementFollowerCount(followingId);
        userRepository.incrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().add("following:" + followerId, followingId.toString());
        redis.opsForSet().add("followers:" + followingId, followerId.toString());
        
        // 4. Backfill feed with recent posts
        backfillFeed(followerId, followingId);
        
        // 5. Send notification
        notificationService.sendFollowNotification(followerId, followingId);
    }
    
    private void backfillFeed(Long followerId, Long followingId) {
        // Get recent posts from new following
        List<Post> recentPosts = postRepository.findRecentByUser(
            followingId,
            Instant.now().minus(7, ChronoUnit.DAYS),
            20
        );
        
        // Add to follower's feed
        for (Post post : recentPosts) {
            redis.opsForZSet().add(
                "feed:" + followerId,
                post.getId(),
                post.getCreatedAt().toEpochMilli()
            );
        }
    }
    
    @Transactional
    public void unfollow(Long followerId, Long followingId) {
        // 1. Remove follow relationship
        followRepository.delete(followerId, followingId);
        
        // 2. Update counts
        userRepository.decrementFollowerCount(followingId);
        userRepository.decrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().remove("following:" + followerId, followingId.toString());
        redis.opsForSet().remove("followers:" + followingId, followerId.toString());
        
        // 4. Remove posts from feed
        removeFromFeed(followerId, followingId);
    }
    
    private void removeFromFeed(Long followerId, Long unfollowedId) {
        // Get posts by unfollowed user
        List<String> postIds = postRepository.findIdsByUser(unfollowedId);
        
        // Remove from feed
        for (String postId : postIds) {
            redis.opsForZSet().remove("feed:" + followerId, postId);
        }
    }
    
    public boolean isFollowing(Long followerId, Long followingId) {
        // Check cache first
        Boolean cached = redis.opsForSet()
            .isMember("following:" + followerId, followingId.toString());
        
        if (cached != null) {
            return cached;
        }
        
        // Fallback to database
        return followRepository.exists(followerId, followingId);
    }
}
```

---

## 5. Notification System

### Notification Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          NOTIFICATION FLOW                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘

Event (Like, Comment, Follow)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  NOTIFICATION SERVICE                                                                │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  1. Create notification record                                               │   │
│  │     INSERT INTO notifications (user_id, type, actor_id, target_id, ...)     │   │
│  │                                                                              │   │
│  │  2. Check user's notification preferences                                    │   │
│  │     - Push enabled?                                                          │   │
│  │     - Email enabled?                                                         │   │
│  │     - Muted?                                                                 │   │
│  │                                                                              │   │
│  │  3. Aggregate if needed (batch similar notifications)                        │   │
│  │     "user1, user2, and 5 others liked your photo"                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│         │                           │                           │                    │
│         ▼                           ▼                           ▼                    │
│  ┌─────────────┐            ┌─────────────┐            ┌─────────────┐              │
│  │ Push (APNs/ │            │   In-App    │            │   Email     │              │
│  │    FCM)     │            │ (WebSocket) │            │  (SES/SG)   │              │
│  └─────────────┘            └─────────────┘            └─────────────┘              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Notification Service Implementation

```java
@Service
public class NotificationService {
    
    private final NotificationRepository notificationRepository;
    private final PushService pushService;
    private final WebSocketService webSocketService;
    
    public void sendNotification(NotificationEvent event) {
        // 1. Create notification record
        Notification notification = Notification.builder()
            .userId(event.getTargetUserId())
            .type(event.getType())
            .actorId(event.getActorId())
            .targetId(event.getTargetId())
            .createdAt(Instant.now())
            .build();
        
        notificationRepository.save(notification);
        
        // 2. Check preferences
        NotificationPreferences prefs = getPreferences(event.getTargetUserId());
        
        if (prefs.isPushEnabled() && !prefs.isMuted(event.getActorId())) {
            // 3. Send push notification
            pushService.send(event.getTargetUserId(), formatPushMessage(event));
        }
        
        // 4. Send in-app notification via WebSocket
        webSocketService.send(event.getTargetUserId(), notification);
    }
    
    // Aggregate similar notifications
    public List<NotificationGroup> getAggregatedNotifications(Long userId) {
        List<Notification> notifications = notificationRepository.findByUser(userId);
        
        // Group by type and target
        Map<String, List<Notification>> grouped = notifications.stream()
            .collect(Collectors.groupingBy(n -> n.getType() + ":" + n.getTargetId()));
        
        return grouped.entrySet().stream()
            .map(e -> createNotificationGroup(e.getValue()))
            .sorted(Comparator.comparing(NotificationGroup::getLatestAt).reversed())
            .collect(Collectors.toList());
    }
    
    private NotificationGroup createNotificationGroup(List<Notification> notifications) {
        // "user1, user2, and 5 others liked your photo"
        List<User> actors = notifications.stream()
            .map(n -> userService.getUser(n.getActorId()))
            .limit(3)
            .collect(Collectors.toList());
        
        return NotificationGroup.builder()
            .type(notifications.get(0).getType())
            .actors(actors)
            .totalActors(notifications.size())
            .latestAt(notifications.get(0).getCreatedAt())
            .build();
    }
}
```

---

## Summary

| Component           | Technology/Algorithm          | Key Configuration                |
| ------------------- | ----------------------------- | -------------------------------- |
| Feed generation     | Hybrid fan-out                | 10K follower threshold           |
| Feed ranking        | Multi-signal scoring          | Interest, relationship, time     |
| Image processing    | Async pipeline                | 3 sizes + blurhash               |
| Stories             | Cassandra with TTL            | 24-hour expiration               |
| Social graph        | PostgreSQL + Redis cache      | Bidirectional caching            |

