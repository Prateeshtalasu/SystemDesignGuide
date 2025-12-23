# News Feed - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a news feed system design.

---

## Trade-off Questions

### Q1: Why use hybrid fan-out instead of pure fan-out on write or read?

**Answer:**

**Pure Fan-out on Write:**
```
Post created → Write to all followers' feeds
Celebrity with 10M followers → 10M writes per post
```
- **Pros:** Fast reads, simple read path
- **Cons:** Massive write amplification for celebrities, wasted work if followers don't check

**Pure Fan-out on Read:**
```
User opens app → Fetch posts from all 200 followings → Merge and rank
```
- **Pros:** No write amplification, always fresh
- **Cons:** Slow reads (must query 200+ sources), hot spots on popular users

**Hybrid Approach (Chosen):**
```
Regular users (< 10K followers): Fan-out on write
Celebrities (> 10K followers): Fan-out on read
```

| Factor           | Write Only         | Read Only          | Hybrid             |
| ---------------- | ------------------ | ------------------ | ------------------ |
| Write cost       | Very high          | Low                | Medium             |
| Read latency     | Low (~50ms)        | High (~500ms)      | Medium (~150ms)    |
| Celebrity posts  | 10M writes         | Hot spot           | Read-time merge    |
| Storage          | High (duplicated)  | Low                | Medium             |

**Why 10K threshold?**
- Below 10K: Fan-out cost is acceptable (~10K writes)
- Above 10K: Write cost exceeds read benefit
- Facebook uses similar threshold (~1K-10K)

---

### Q2: How do you handle the "thundering herd" when a celebrity posts?

**Answer:**

**The Problem:**
Celebrity with 50M followers posts during Super Bowl. Suddenly 10M people refresh their feeds simultaneously.

**Solutions:**

1. **Fan-out on Read (Already Implemented)**
   - Celebrity posts aren't pre-fanned
   - Each reader fetches from celebrity's timeline
   - But: Celebrity's timeline becomes hot spot

2. **Caching Celebrity Posts**
   ```java
   // Cache celebrity's recent posts
   String key = "celebrity_posts:" + celebrityId;
   List<Post> posts = redis.get(key);
   
   // Refresh cache every 30 seconds
   // All readers hit cache, not database
   ```

3. **Request Coalescing**
   ```java
   // Only one request to backend for same celebrity
   CompletableFuture<List<Post>> future = inFlightRequests.computeIfAbsent(
       celebrityId,
       id -> fetchCelebrityPosts(id)
           .whenComplete((r, e) -> inFlightRequests.remove(id))
   );
   return future;
   ```

4. **Rate Limiting Feed Refreshes**
   ```java
   // Max 1 refresh per user per 5 seconds
   if (rateLimiter.tryAcquire(userId)) {
       return generateFreshFeed(userId);
   } else {
       return getCachedFeed(userId);
   }
   ```

5. **Stale-While-Revalidate**
   - Serve cached feed immediately
   - Refresh in background
   - User sees feed instantly, updates appear shortly

---

### Q3: Why use Redis Sorted Sets for feed cache?

**Answer:**

**Alternatives Considered:**

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

**Example Operations:**
```redis
# Add post to feed with score
ZADD feed:user_123 0.95 post_abc

# Get top 20 posts
ZREVRANGE feed:user_123 0 19 WITHSCORES

# Pagination (posts with score < 0.90)
ZREVRANGEBYSCORE feed:user_123 0.90 -inf LIMIT 0 20

# Remove old posts (keep top 500)
ZREMRANGEBYRANK feed:user_123 0 -501
```

---

### Q4: How do you ensure feed consistency when a user unfollows someone?

**Answer:**

**The Problem:**
User A unfollows User B. User B's posts should disappear from A's feed immediately.

**Challenges:**
1. Posts already in feed cache
2. Posts in flight (being fanned out)
3. User might have scrolled past them

**Solution:**

```java
public void handleUnfollow(Long followerId, Long unfollowedId) {
    // 1. Update graph immediately
    graphService.removeFollow(followerId, unfollowedId);
    
    // 2. Remove posts from feed cache
    List<Long> postIds = postRepository.findPostIdsByAuthor(unfollowedId);
    feedCacheService.removePostsFromFeed(followerId, postIds);
    
    // 3. Add to "hidden authors" set (for in-flight posts)
    redis.opsForSet().add("hidden:" + followerId, unfollowedId.toString());
    redis.expire("hidden:" + followerId, 1, TimeUnit.HOURS);
    
    // 4. Client-side: Send WebSocket message to remove posts
    webSocketService.sendRemovePostsMessage(followerId, postIds);
}

// During feed generation, filter hidden authors
public List<FeedEntry> getFeed(Long userId) {
    List<FeedEntry> feed = feedCache.getFeed(userId);
    Set<Long> hiddenAuthors = getHiddenAuthors(userId);
    
    return feed.stream()
        .filter(entry -> !hiddenAuthors.contains(entry.getAuthorId()))
        .collect(Collectors.toList());
}
```

**Trade-off:** We accept that some posts might briefly appear before being filtered. Perfect consistency would require locking, which hurts performance.

---

### Q5: How do you rank posts for relevance?

**Answer:**

**Ranking Signals (with weights):**

| Signal           | Weight | Why                                    |
| ---------------- | ------ | -------------------------------------- |
| Recency          | 30%    | Fresh content is more relevant         |
| Engagement       | 25%    | Popular posts are likely interesting   |
| Affinity         | 25%    | Posts from close friends matter more   |
| Content match    | 15%    | Match user's interests                 |
| Author quality   | 5%     | Verified, low spam score               |

**Ranking Formula:**
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

**Real-time vs Batch Ranking:**
- **Pre-ranking**: When post is fanned out (base score)
- **Re-ranking**: When feed is fetched (personalized adjustments)
- **Batch update**: Hourly recalculation of engagement scores

---

## Scaling Questions

### Q6: How would you scale from 500M to 2B DAU?

**Answer:**

**Current State (500M DAU):**
- 1M peak QPS
- 150 feed servers
- 66 Redis nodes

**Scaling to 2B DAU (4x):**

1. **Feed Servers: 150 → 600**
   - Linear scaling with users
   - Add more regions for global coverage

2. **Redis Cluster: 66 → 250 nodes**
   - More shards for feed cache
   - Increase memory per node

3. **Kafka: 20 → 80 brokers**
   - More partitions for fan-out events
   - Higher throughput

4. **Architecture Changes:**

   ```
   Current: Single global deployment
   Scaled: Regional deployments with cross-region sync
   
   US users → US cluster
   EU users → EU cluster
   APAC users → APAC cluster
   
   Celebrity posts → Replicated globally
   Regular posts → Regional only (unless cross-region follow)
   ```

5. **Optimization Opportunities:**
   - More aggressive caching (push to CDN edge)
   - Smarter fan-out (skip inactive users)
   - Tiered storage (hot/warm/cold feeds)

**Cost Impact:**
- Servers: 600 → 2,400 (~4x)
- Monthly cost: $1.7M → $6.8M (~4x)
- Cost per DAU stays similar

---

### Q7: What happens if the ranking service goes down?

**Answer:**

**Impact:**
- Feeds would show unranked posts
- User engagement could drop
- But users can still see content

**Graceful Degradation:**

```java
public List<FeedEntry> getFeed(Long userId) {
    List<FeedEntry> feed = feedCache.getFeed(userId);
    
    try {
        // Try to rank
        return rankingService.rank(feed, userId);
    } catch (Exception e) {
        log.warn("Ranking service unavailable, falling back to chronological");
        metrics.recordRankingFallback();
        
        // Fallback: Sort by timestamp (already have this data)
        return feed.stream()
            .sorted(Comparator.comparing(FeedEntry::getCreatedAt).reversed())
            .collect(Collectors.toList());
    }
}
```

**Recovery:**
1. Circuit breaker opens after 5 failures
2. Chronological feed served during outage
3. Circuit breaker half-opens after 30 seconds
4. If ranking recovers, resume normal operation

**Monitoring:**
- Alert on ranking service errors
- Track fallback rate
- Compare engagement metrics during fallback

---

## Failure Scenarios

### Scenario 1: Redis Cluster Partial Failure

**Problem:** 3 of 66 Redis nodes fail, losing some users' cached feeds.

**Solution:**

```java
public List<FeedEntry> getFeedWithFallback(Long userId) {
    try {
        return feedCache.getFeed(userId);
    } catch (RedisException e) {
        log.warn("Redis unavailable for user {}, regenerating feed", userId);
        
        // Regenerate from Cassandra
        List<FeedEntry> feed = cassandraFeedStore.getFeed(userId);
        
        // Async: Try to repopulate Redis cache
        asyncRepopulateCache(userId, feed);
        
        return feed;
    }
}
```

**Prevention:**
- Redis Cluster with 3 replicas per shard
- Automatic failover (< 30 seconds)
- Cassandra as backup feed store

---

### Scenario 2: Fan-out Worker Backlog

**Problem:** Celebrity posts cause massive fan-out backlog, regular users' posts delayed.

**Solution:**

```java
// Separate queues for different post types
@KafkaListener(topics = "regular-post-events")
public void handleRegularPosts(PostEvent event) {
    // Fast lane for regular users
    fanoutService.process(event);
}

@KafkaListener(topics = "celebrity-post-events")
public void handleCelebrityPosts(PostEvent event) {
    // These don't fan out, just store
    celebrityPostStore.save(event.getPost());
}

// Priority-based processing
public void routePost(Post post) {
    if (post.getAuthor().getFollowersCount() > CELEBRITY_THRESHOLD) {
        kafka.send("celebrity-post-events", post);
    } else {
        kafka.send("regular-post-events", post);
    }
}
```

**Monitoring:**
- Consumer lag per topic
- Alert if regular posts delayed > 30 seconds

---

### Scenario 3: Database Overload During Peak

**Problem:** Super Bowl causes 10x traffic spike, PostgreSQL can't keep up.

**Solution:**

```java
// Read-through cache with circuit breaker
public Post getPost(Long postId) {
    // 1. Try cache
    Post cached = redis.get("post:" + postId);
    if (cached != null) {
        return cached;
    }
    
    // 2. Check circuit breaker
    if (dbCircuitBreaker.isOpen()) {
        throw new ServiceUnavailableException("Database overloaded");
    }
    
    // 3. Query database with timeout
    try {
        Post post = postRepository.findById(postId)
            .orTimeout(100, TimeUnit.MILLISECONDS)
            .get();
        
        // Cache for future requests
        redis.setex("post:" + postId, 3600, post);
        
        dbCircuitBreaker.recordSuccess();
        return post;
        
    } catch (TimeoutException e) {
        dbCircuitBreaker.recordFailure();
        throw new ServiceUnavailableException("Database timeout");
    }
}
```

**Prevention:**
- Read replicas for read-heavy workloads
- Connection pooling with queue limits
- Rate limiting on feed refreshes

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand basic feed generation
- Know difference between push and pull
- Simple caching strategy
- Basic database schema

**Sample Answer Quality:**

> "When a user posts, we store it in a database and add it to their followers' feeds. We'd use a cache like Redis to store each user's feed so reads are fast. For the database, we'd have a posts table and a follows table."

**Red Flags:**
- No mention of fan-out strategies
- Ignores scale implications
- No discussion of ranking

---

### L5 (Mid-Level)

**What's Expected:**
- Hybrid fan-out with celebrity handling
- Multi-signal ranking algorithm
- Caching layers and invalidation
- Failure handling and degradation

**Sample Answer Quality:**

> "I'd use a hybrid fan-out approach. For regular users with under 10K followers, we fan-out on write - when they post, we push to all followers' feed caches in Redis. For celebrities, we fan-out on read - their posts are stored separately and merged at read time.

> For ranking, I'd combine recency (exponential decay), engagement (log-normalized likes/comments/shares), and affinity (how often the viewer interacts with the author). The ranking service runs at read time for personalization.

> If the ranking service fails, we fall back to chronological ordering. The feed is still usable, just not personalized."

**Red Flags:**
- Can't explain why hybrid approach
- No understanding of ranking signals
- Missing failure handling

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- Complex edge cases (viral posts, unfollows)
- Cost optimization at scale
- Real-world examples

**Sample Answer Quality:**

> "Let me walk through how this evolves. For MVP with 1M users, pure fan-out on write works fine - it's simple and reads are fast. As we scale to 100M users, we hit the celebrity problem. A single celebrity post could trigger 10M writes, which is unsustainable.

> The hybrid approach solves this, but introduces complexity. At read time, we need to merge precomputed feeds with celebrity posts. This is where ranking becomes critical - we can't just sort by time because celebrity posts would dominate.

> For ranking, Facebook uses a combination of affinity (EdgeRank), time decay, and engagement signals. The key insight is that ranking happens in two phases: pre-ranking during fan-out (base score) and re-ranking at read time (personalized). This balances computation cost with relevance.

> The hardest edge case is the 'unfollow' problem. When you unfollow someone, their posts should disappear immediately. We handle this by maintaining a 'hidden authors' set per user that's checked at read time. It's eventually consistent but provides good UX.

> For cost optimization at 500M DAU, the biggest lever is reducing fan-out for inactive users. If someone hasn't opened the app in 30 days, we don't fan out to them - we regenerate their feed on demand. This can reduce fan-out volume by 40%."

**Red Flags:**
- Can't discuss evolution
- No awareness of real-world solutions
- Missing cost considerations

---

## Common Interviewer Pushbacks

### "Your feed will be stale"

**Response:**
"You're right that there's a trade-off between freshness and cost. For regular users, the fan-out on write ensures posts appear within seconds. For celebrity posts, we merge at read time, so there's no staleness. The main staleness risk is if a user has their app open for hours without refreshing - we handle this with WebSocket push notifications for new posts from close friends, and a 'New Posts' indicator that encourages refresh."

### "What about privacy settings?"

**Response:**
"Privacy adds complexity to fan-out. When a user posts with 'Friends Only' privacy, we need to filter the follower list to only include friends, not just followers. We maintain a separate 'friends' relationship that's bidirectional. During fan-out, we check the privacy setting and filter accordingly. For 'Only Me' posts, we skip fan-out entirely."

### "How do you handle deleted posts?"

**Response:**
"Post deletion needs to be fast and complete. When a user deletes a post: (1) We mark it deleted in the database immediately, (2) We remove it from the author's timeline cache, (3) We publish a 'post_deleted' event to Kafka, (4) Fan-out workers remove it from all followers' feed caches. We also send a WebSocket message to connected clients to remove it from their UI. The deletion propagates within seconds."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | Hybrid fan-out, ranking signals, consistency       |
| Scaling             | Regional deployment, caching layers                |
| Failures            | Graceful degradation, circuit breakers             |
| Edge cases          | Celebrities, unfollows, deletions                  |
| Evolution           | MVP → Scale → Optimization phases                  |

