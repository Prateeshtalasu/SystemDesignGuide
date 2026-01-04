# Instagram / Photo Sharing - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, and level-specific expectations for a photo sharing platform design.

---

## Trade-off Questions

### Q1: Why use hybrid fan-out instead of pure fan-out on write?

**Answer:**

**The Celebrity Problem:**
```
Cristiano Ronaldo: 500M followers
Posts 1 photo → 500M feed writes
At 1ms per write → 8+ minutes to propagate
```

**Hybrid Approach:**

| User Type           | Strategy           | Rationale                        |
| ------------------- | ------------------ | -------------------------------- |
| Regular (< 10K)     | Fan-out on write   | Fast reads, manageable writes    |
| Celebrity (> 10K)   | Fan-out on read    | Avoid write amplification        |

**Implementation:**

```java
if (author.getFollowerCount() > 10_000) {
    // Store in celebrity posts, merge at read time
    storeCelebrityPost(post);
} else {
    // Push to all followers' feeds
    fanOutToFollowers(post, followers);
}
```

**Trade-offs:**

| Metric           | Write-only        | Read-only         | Hybrid            |
| ---------------- | ----------------- | ----------------- | ----------------- |
| Write cost       | Very high         | Low               | Medium            |
| Read latency     | Low (~50ms)       | High (~300ms)     | Medium (~100ms)   |
| Storage          | High              | Low               | Medium            |
| Complexity       | Low               | Medium            | High              |

---

### Q2: How do you handle image processing at scale?

**Answer:**

**The Challenge:**
- 50M uploads/day = 600 uploads/second
- Each upload needs: validation, resize (3 sizes), filter, blurhash
- Processing time: ~5 seconds per image

**Solution: Async Pipeline**

```
Upload → Store Original → Queue → Workers → Store Processed → Notify
```

**Why Async?**
1. **User Experience**: Return immediately after upload
2. **Scalability**: Scale workers independently
3. **Reliability**: Retry failed processing

**Progressive Availability:**
```java
// Phase 1: Upload accepted (immediate)
response.setStatus("processing");
response.setBlurhash(generateQuickBlurhash(image));

// Phase 2: Thumbnail ready (~2 seconds)
// Phase 3: Medium ready (~3 seconds)
// Phase 4: Large ready (~5 seconds)
```

**Client shows blurhash → thumbnail → full image**

---

### Q3: Why store stories in Cassandra instead of PostgreSQL?

**Answer:**

**Stories Characteristics:**
- Write-heavy (100M stories/day)
- Time-series data
- Auto-expire after 24 hours
- Read pattern: by user, recent first

**Why Cassandra:**

| Feature           | PostgreSQL        | Cassandra         |
| ----------------- | ----------------- | ----------------- |
| TTL support       | Manual cleanup    | Native TTL        |
| Write throughput  | Limited           | Excellent         |
| Time-series       | Requires indexes  | Natural fit       |
| Horizontal scale  | Complex           | Easy              |

**Cassandra Schema:**
```sql
CREATE TABLE stories (
    user_id TEXT,
    story_id TIMEUUID,
    ...
    PRIMARY KEY (user_id, story_id)
) WITH CLUSTERING ORDER BY (story_id DESC)
  AND default_time_to_live = 86400;  -- Auto-delete after 24h
```

**Trade-off:**
- Cassandra: No joins, eventual consistency
- Acceptable for stories (not critical data)

---

### Q4: How do you ensure photos are never lost?

**Answer:**

**Durability Strategy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHOTO DURABILITY                              │
└─────────────────────────────────────────────────────────────────┘

1. Upload Phase:
   - Client calculates MD5 before upload
   - Server verifies MD5 after upload
   - Don't acknowledge until verified

2. Storage Phase:
   - S3 with 11 9's durability
   - Cross-region replication (3 regions)
   - Versioning enabled

3. Processing Phase:
   - Original preserved until all sizes generated
   - Each size verified with checksum
   - Original kept indefinitely

4. Serving Phase:
   - CDN serves copies
   - Origin is source of truth
   - Regular integrity checks
```

**Checksum Verification:**
```java
public void verifyUpload(String key, String expectedMd5) {
    S3Object object = s3Client.getObject(key);
    String actualMd5 = calculateMd5(object.getContent());
    
    if (!expectedMd5.equals(actualMd5)) {
        throw new IntegrityException("MD5 mismatch");
    }
}
```

---

### Q5: How do you rank the feed?

**Answer:**

**Ranking Signals:**

| Signal           | Weight | Description                          |
| ---------------- | ------ | ------------------------------------ |
| Interest         | 40%    | User's interest in content type      |
| Relationship     | 30%    | Interaction history with author      |
| Timeliness       | 20%    | How recent is the post               |
| Engagement       | 10%    | Post's overall performance           |

**Interest Score:**
```java
// Based on user's past interactions with similar content
double interestScore = 0.0;
interestScore += getUserCategoryAffinity(user, post.getCategory()) * 0.5;
interestScore += getUserHashtagAffinity(user, post.getHashtags()) * 0.3;
interestScore += getUserAuthorTypeAffinity(user, post.getAuthor()) * 0.2;
```

**Relationship Score:**
```java
// Based on interaction history
InteractionStats stats = getInteractionStats(viewer, author);
double relationshipScore = 
    Math.min(stats.getLikes() / 50.0, 0.3) +
    Math.min(stats.getComments() / 20.0, 0.3) +
    Math.min(stats.getProfileVisits() / 10.0, 0.2) +
    (stats.hasDMs() ? 0.2 : 0.0);
```

**Timeliness Score:**
```java
// Exponential decay
long ageHours = getAgeInHours(post);
double timelinessScore = Math.exp(-ageHours / 6.0);  // 6-hour half-life
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Basic upload/feed flow
- Simple database schema
- Understanding of CDN

**Sample Answer Quality:**

> "Users upload photos to our servers. We store them in S3 and serve via CDN. For the feed, we query the database for posts from users they follow, sorted by time."

**Red Flags:**
- No mention of image processing
- Doesn't understand fan-out
- No caching strategy

---

### L5 (Mid-Level)

**What's Expected:**
- Image processing pipeline
- Fan-out strategies
- Feed ranking basics
- Stories architecture

**Sample Answer Quality:**

> "When a user uploads a photo, we store the original in S3 and queue it for processing. Workers generate multiple sizes (thumbnail, medium, large) and apply filters. We also generate a blurhash for progressive loading.

> For the feed, we use hybrid fan-out. Regular users' posts are fanned out to followers' feeds in Redis. Celebrity posts (>10K followers) are stored separately and merged at read time to avoid write amplification.

> Stories use Cassandra with 24-hour TTL. The partition key is user_id, so we can efficiently query all stories for a user. The TTL handles automatic cleanup.

> Feed ranking considers recency, engagement, and relationship strength. We calculate interaction history between the viewer and author to boost posts from close connections."

**Red Flags:**
- Can't explain fan-out trade-offs
- No understanding of image processing
- Missing stories architecture

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- Scaling for major events
- Cost optimization
- ML integration for ranking

**Sample Answer Quality:**

> "Let me walk through the evolution. For MVP, we'd use PostgreSQL for everything and process images synchronously. This works for 100K users but doesn't scale.

> For scale, we separate concerns. Image processing becomes async with Kafka and dedicated workers. We add Redis for feed caching with hybrid fan-out. Stories move to Cassandra for TTL support.

> The 10K follower threshold for fan-out is based on analysis: below 10K, write cost is acceptable. Above 10K, read-time merging is more efficient. We continuously tune this based on metrics.

> For ranking, we start with simple time-decay, then add engagement signals. At scale, we introduce ML models trained on user behavior. The model predicts probability of engagement and we rank by expected value.

> For major events like New Year's, we pre-scale 5x, enable graceful degradation (disable explore, reduce quality), and have playbooks for different scenarios. The key is accepting some degradation to maintain core functionality.

> Cost optimization: We use tiered storage (hot/warm/cold), compress aggressively, and serve lower quality to users on slow connections. CDN costs dominate, so we optimize cache hit rates."

**Red Flags:**
- Can't discuss trade-offs
- No cost awareness
- Missing ML ranking discussion

---

## Common Interviewer Pushbacks

### "Your feed will be stale"

**Response:**
"There's a trade-off between freshness and cost. Our hybrid approach ensures:

1. Regular users' posts appear immediately (fan-out on write)
2. Celebrity posts are merged at read time (always fresh)
3. We can tune the threshold based on acceptable staleness

For real-time scenarios, we use WebSocket to push new posts to active users. The feed is refreshed on pull-to-refresh, and we show a 'New posts' indicator."

### "What about privacy?"

**Response:**
"Privacy is handled at multiple levels:

1. **Private accounts**: Posts only visible to approved followers. We check follow status before showing posts.

2. **Blocked users**: Blocked users can't see your content. We maintain a block list and filter at query time.

3. **Stories close friends**: Separate visibility list stored with each story. Checked at view time.

4. **Feed**: We never show posts from users you've blocked or who've blocked you.

Implementation: We cache visibility rules and check them before serving any content."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | Fan-out strategy, Cassandra vs PostgreSQL          |
| Scaling             | Major events, celebrity problem                    |
| Failures            | Cache miss, processing backlog                     |
| Quality             | Image processing, progressive loading              |
| Ranking             | Multi-signal scoring, ML integration               |
