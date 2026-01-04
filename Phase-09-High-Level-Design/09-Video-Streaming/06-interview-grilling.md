# Video Streaming - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, and level-specific expectations for a video streaming system design.

---

## Trade-off Questions

### Q1: Why use HLS/DASH instead of progressive download?

**Answer:**

| Approach            | Pros                          | Cons                           |
| ------------------- | ----------------------------- | ------------------------------ |
| **Progressive**     | Simple, single file           | No ABR, full download needed   |
| **HLS/DASH**        | ABR, seek anywhere, efficient | Complex, more requests         |

**Why HLS/DASH:**

1. **Adaptive Bitrate**: Quality adjusts to network conditions
2. **Efficient Seeking**: Jump to any point without downloading everything
3. **CDN Friendly**: Small segments cache well
4. **Bandwidth Efficient**: Only download what you watch

**HLS vs DASH:**

| Feature           | HLS                    | DASH                   |
| ----------------- | ---------------------- | ---------------------- |
| Developed by      | Apple                  | MPEG                   |
| Container         | TS (or fMP4)           | fMP4                   |
| Browser support   | Safari native, others via JS | Chrome/Firefox native |
| DRM               | FairPlay               | Widevine, PlayReady    |

**Our Choice**: HLS with fMP4 containers
- Best compatibility
- Good CDN support
- Works with most DRM systems

---

### Q2: Why transcode to multiple resolutions instead of just one?

**Answer:**

**The Problem:**
- Users have different devices (phone, tablet, TV, desktop)
- Users have different network conditions (3G, 4G, WiFi, fiber)
- Single resolution doesn't fit all

**Resolution Strategy:**

| Resolution | Target Device        | Network        | % of Views |
| ---------- | -------------------- | -------------- | ---------- |
| 240p       | Feature phones       | 2G/3G          | 5%         |
| 360p       | Phones (data saving) | 3G             | 10%        |
| 480p       | Phones               | 4G             | 20%        |
| 720p       | Tablets, phones      | 4G/WiFi        | 35%        |
| 1080p      | Desktops, TVs        | WiFi/Fiber     | 25%        |
| 4K         | 4K TVs               | Fiber          | 5%         |

**Cost-Benefit Analysis:**

```
Storage cost for 10-minute video:
- Single 1080p: 450 MB
- All resolutions: 2 GB (~4.5x)

But benefits:
- 30% less bandwidth (users get appropriate quality)
- 50% less rebuffering (can downgrade smoothly)
- Better mobile experience (faster startup)

ROI: Storage cost increase is offset by bandwidth savings and user satisfaction
```

---

### Q3: How do you ensure video is never lost?

**Answer:**

**Durability Strategy:**

1. **Upload Phase:**
   - Chunked upload with checksum per chunk
   - Retry failed chunks automatically
   - Don't acknowledge until all chunks verified

2. **Storage Phase:**
   - S3 with 99.999999999% durability
   - Cross-region replication (3 regions)
   - Versioning enabled (accidental delete protection)

3. **Processing Phase:**
   - Original preserved until all transcodes complete
   - Transcoded files verified with checksum
   - Original kept for 30 days after processing

4. **Serving Phase:**
   - CDN serves copies, origin is source of truth
   - Regular integrity checks
   - Automated re-transcoding if corruption detected

---

### Q4: How do you count views accurately at scale?

**Answer:**

**The Challenge:**
- 2.5 billion views/day = 30,000 views/second
- Need real-time display
- Must prevent fraud
- Must handle duplicates

**Solution: Multi-tier Counting**

```java
public class ViewCountingStrategy {
    
    // Tier 1: Real-time (Redis)
    // - Increment on every valid view
    // - Used for display
    // - May have slight inaccuracy
    
    // Tier 2: Near real-time (Kafka + Flink)
    // - Process view events
    // - Deduplicate
    // - Detect fraud patterns
    
    // Tier 3: Batch (Spark)
    // - Daily reconciliation
    // - Accurate final count
    // - Used for analytics/monetization
}
```

**Fraud Prevention:**

```java
public boolean isValidView(ViewEvent event) {
    // 1. Watch duration check
    if (event.getWatchDuration() < 30 && 
        event.getWatchDuration() < event.getVideoDuration() * 0.5) {
        return false; // Too short
    }
    
    // 2. Rate limiting per user
    String userKey = "view_rate:" + event.getUserId();
    if (redis.incr(userKey) > 100) { // Max 100 views/hour
        return false;
    }
    redis.expire(userKey, 3600);
    
    // 3. IP-based rate limiting
    String ipKey = "view_rate_ip:" + event.getIpAddress();
    if (redis.incr(ipKey) > 1000) { // Max 1000 views/hour per IP
        return false;
    }
    
    // 4. Bot detection (simplified)
    if (event.getUserAgent().contains("bot")) {
        return false;
    }
    
    return true;
}
```

---

### Q5: Why Cassandra for watch history instead of PostgreSQL?

**Answer:**

| Aspect              | PostgreSQL                    | Cassandra                      |
| ------------------- | ----------------------------- | ------------------------------ |
| Write pattern       | Random writes                 | Sequential writes              |
| Read pattern        | Flexible queries              | Partition key required         |
| Scaling             | Vertical + read replicas      | Horizontal (linear)            |
| Query              | "All history for user X"      | Optimized for this pattern     |

**Why Cassandra for Watch History:**
1. **Time-series data**: Natural fit for watch events
2. **User-partitioned**: All user's history on same node
3. **High write volume**: Millions of watch events/second
4. **Horizontal scale**: Add nodes as users grow

**Trade-off:**
- No ad-hoc queries (design tables for access patterns)
- Eventual consistency (acceptable for watch history)

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand basic video pipeline
- Know about CDN concept
- Simple upload/storage flow

**Sample Answer Quality:**

> "Users upload videos to our servers. We store them in S3 and use a CDN to serve them globally. The CDN caches videos at edge locations close to users for faster delivery."

**Red Flags:**
- No mention of transcoding
- Doesn't understand adaptive streaming
- No scale considerations

---

### L5 (Mid-Level)

**What's Expected:**
- Transcoding pipeline with multiple resolutions
- HLS/DASH adaptive streaming
- CDN architecture with origin shield
- View counting strategy

**Sample Answer Quality:**

> "When a user uploads a video, we store the original in S3 and queue it for transcoding. Our transcoding workers convert it to multiple resolutions (240p to 4K) and generate HLS manifests. Each resolution is segmented into 10-second chunks.

> For delivery, we use a multi-tier CDN: edge servers close to users, regional PoPs, and origin shield protecting our origin. The client uses adaptive bitrate streaming - it measures bandwidth and switches quality accordingly.

> For view counting, we use Redis for real-time counts and Kafka for event processing. We batch-aggregate to PostgreSQL hourly for accuracy."

**Red Flags:**
- Can't explain adaptive bitrate
- No understanding of CDN tiers
- Missing view counting complexity

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- Live streaming considerations
- Cost optimization strategies
- Viral content handling

**Sample Answer Quality:**

> "Let me walk through the evolution. For MVP, we'd use a simple pipeline: upload to S3, transcode with FFmpeg workers, serve via CloudFront. This works for thousands of videos but doesn't scale.

> For scale, we need to optimize at each layer. Transcoding becomes the bottleneck first - we parallelize across resolutions and use GPU acceleration for 5x speedup. The cost increase is offset by faster time-to-publish.

> CDN architecture is critical. We use a three-tier approach: edge (85% hit rate), regional PoP (10%), origin shield (4%), origin (1%). The origin shield is key - it aggregates cache misses so a viral video doesn't overwhelm our origin.

> For viral content, we detect high view velocity early and proactively warm CDN caches. We also increase cache TTL from 1 hour to 24 hours for trending content.

> View counting is tricky at scale. We can't write to a database on every view - that's 30K writes/second. Instead, we use Redis for real-time display (eventually consistent), Kafka for processing, and batch reconciliation for accuracy. For monetization, we use the batch-reconciled numbers.

> For live streaming, the architecture changes significantly. We need real-time transcoding, 2-second segments instead of 10, and we accept higher latency (10-30 seconds) for better scalability. For a 100M viewer event, we'd pre-scale CDN 3x, use dedicated encoder farms, and have automated failover."

**Red Flags:**
- Can't discuss trade-offs
- No cost awareness
- Missing live streaming considerations

---

## Common Interviewer Pushbacks

### "Your CDN costs will be enormous"

**Response:**
"You're right - CDN bandwidth is the largest cost at scale. We optimize in several ways:

1. **Codec efficiency**: H.265/VP9 reduces bitrate by 30-40% vs H.264
2. **Tiered storage**: Only popular videos on fast CDN, long-tail on cheaper storage
3. **Quality capping**: Default to 720p on mobile, users can opt for higher
4. **Regional pricing**: Negotiate better rates in high-traffic regions

At YouTube scale, CDN costs are ~$0.01 per video view. The revenue from ads (~$0.05 per view) more than covers it."

### "How do you handle copyright infringement?"

**Response:**
"While content moderation is out of scope for this design, the architecture supports it:

1. **Upload time**: Hash uploaded video, check against known infringing content database
2. **Audio fingerprinting**: Detect copyrighted music (like YouTube's Content ID)
3. **Manual review queue**: Flag suspicious content for human review
4. **Takedown API**: Allow copyright holders to report and remove content

The key is doing this asynchronously - we don't block upload, but we may block publishing until checks pass."

### "What about DRM for premium content?"

**Response:**
"For premium/paid content, we need Digital Rights Management:

1. **Encryption**: Encrypt video segments with AES-128 or AES-256
2. **Key delivery**: License server provides decryption keys to authorized clients
3. **Multi-DRM**: Support Widevine (Android/Chrome), FairPlay (Apple), PlayReady (Microsoft)
4. **Hardware security**: Use hardware-backed DRM for highest security tier

Trade-off: DRM adds latency (key request) and complexity. We only use it for premium content, not free ad-supported videos."

### "How do you handle users in regions with poor connectivity?"

**Response:**
"We optimize for low-bandwidth regions:

1. **Lower quality default**: Start at 360p instead of 720p
2. **Aggressive ABR**: Switch down faster when buffer is low
3. **Smaller segments**: 4-second segments for faster quality switching
4. **Preloading**: Download next segment while watching current
5. **Offline mode**: Allow downloads for later viewing (with DRM)

We also deploy edge servers in these regions to reduce latency, even if bandwidth is limited."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | HLS vs progressive, resolution strategy            |
| Scaling             | CDN tiers, viral handling, live events             |
| Failures            | Edge failover, origin redundancy, queue backup     |
| Cost                | CDN optimization, codec efficiency                 |
| Quality             | ABR algorithm, startup time, rebuffering           |
