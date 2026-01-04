# Instagram / Photo Sharing - Production Deep Dives (Operations)

## Overview

This document covers operational aspects: monitoring & metrics, scaling strategies, failure handling, major event preparation, and cost optimization.

---

## 1. Monitoring & Metrics

### Key Metrics

```java
@Component
public class InstagramMetrics {
    
    private final MeterRegistry registry;
    
    // Upload metrics
    private final Timer uploadLatency;
    private final Counter uploadSuccess;
    private final Counter uploadFailure;
    
    // Feed metrics
    private final Timer feedLatency;
    private final DistributionSummary feedSize;
    
    // Engagement metrics
    private final Counter likes;
    private final Counter comments;
    private final Counter follows;
    
    public InstagramMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.uploadLatency = Timer.builder("instagram.upload.latency")
            .description("Photo upload latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.feedLatency = Timer.builder("instagram.feed.latency")
            .description("Feed load latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public void recordUpload(long durationMs, boolean success) {
        uploadLatency.record(durationMs, TimeUnit.MILLISECONDS);
        if (success) {
            uploadSuccess.increment();
        } else {
            uploadFailure.increment();
        }
    }
}
```

### Alerting Thresholds

| Metric                  | Warning    | Critical   | Action                        |
| ----------------------- | ---------- | ---------- | ----------------------------- |
| Upload latency P99      | > 5s       | > 10s      | Scale image processors        |
| Feed latency P99        | > 500ms    | > 1s       | Check cache hit rate          |
| Image processing queue  | > 10K      | > 50K      | Add processing workers        |
| CDN hit rate            | < 90%      | < 80%      | Check cache configuration     |

### Grafana Dashboard Queries

```promql
# Upload latency P99
histogram_quantile(0.99, sum(rate(instagram_upload_latency_bucket[5m])) by (le))

# Feed latency P99
histogram_quantile(0.99, sum(rate(instagram_feed_latency_bucket[5m])) by (le))

# Image processing queue depth
sum(instagram_processing_queue_depth)

# Cache hit rate
sum(rate(redis_cache_hits_total[5m])) / sum(rate(redis_cache_requests_total[5m]))

# Engagement rate
sum(rate(instagram_likes_total[1h])) / sum(rate(instagram_views_total[1h]))
```

---

## 2. Scaling Strategies

### Major Event Handling (New Year's Eve)

**Expected Traffic:**
- Normal: 50M uploads/day
- New Year's: 250M uploads (5x)
- Peak: 10x normal for 2 hours around midnight

**Preparation:**

1. **Pre-scale Infrastructure:**
   ```
   - Image processors: 3,000 → 15,000
   - API servers: 180 → 900
   - Redis capacity: 3.5 TB → 10 TB
   ```

2. **Graceful Degradation:**
   ```java
   if (systemLoad > 0.8) {
       // Disable non-essential features
       disableExplore();
       disableRecommendations();
       reduceImageQuality();
   }
   
   if (systemLoad > 0.95) {
       // Queue uploads instead of processing immediately
       queueUploadForLater();
       showMessage("Your photo will be ready in a few minutes");
   }
   ```

3. **CDN Preparation:**
   ```
   - Pre-warm popular content
   - Increase edge capacity
   - Extend cache TTLs
   ```

4. **Monitoring:**
   ```
   - Real-time dashboards
   - Auto-scaling triggers
   - On-call team ready
   ```

### Image Processing Backlog

**Problem:** Processing queue grows to 1M images (normally 10K).

```java
@Component
public class ProcessingQueueMonitor {
    
    @Scheduled(fixedRate = 60000)
    public void checkQueueHealth() {
        long queueSize = getQueueSize();
        
        if (queueSize > 100_000) {
            // Scale up workers
            scaleWorkers(currentWorkers * 2);
            
            // Enable fast mode (skip non-essential processing)
            enableFastMode();
        }
        
        if (queueSize > 500_000) {
            // Critical: Temporary measures
            alertOpsTeam();
            
            // Skip blurhash generation
            skipBlurhash = true;
            
            // Reduce image quality
            reduceQuality = true;
        }
    }
    
    private void enableFastMode() {
        // Generate only 2 sizes instead of 3
        // Skip filter if not explicitly requested
        // Use lower quality compression
    }
}
```

### Celebrity Posts Spike

**Problem:** Multiple celebrities post simultaneously, overwhelming read path.

```java
@Service
public class CelebrityPostHandler {
    
    public void handleCelebrityPost(Post post) {
        // 1. Pre-warm CDN with post images
        cdnClient.prefetch(post.getImageUrls());
        
        // 2. Cache post data aggressively
        redis.opsForValue().set(
            "post:" + post.getId(),
            serialize(post),
            Duration.ofHours(24)
        );
        
        // 3. Send push notifications in batches
        // Don't notify all 500M followers at once
        notificationService.sendBatchedNotifications(
            post.getAuthor().getFollowerIds(),
            batchSize = 100_000,
            delayBetweenBatches = Duration.ofSeconds(30)
        );
    }
}
```

### Backup and Recovery

**RPO (Recovery Point Objective):** < 15 minutes
- Maximum acceptable data loss
- Based on database replication lag (15 minutes max)
- Photo metadata replicated to read replicas within 15 minutes
- Photos stored in S3 with cross-region replication (no data loss)

**RTO (Recovery Time Objective):** < 10 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Disaster Recovery Testing:**
- **Frequency:** Quarterly
- **Process:** 
  1. Simulate primary region failure
  2. Execute failover procedure to secondary region
  3. Verify RTO < 10 minutes, RPO < 15 minutes
  4. Validate photo upload and retrieval functionality
  5. Test feed generation post-failover
- **Last Test:** TBD (to be scheduled)
- **Validation Criteria:**
  - All photos remain accessible
  - No data loss (RPO < 15 minutes verified)
  - Service resumes within RTO target
  - Feed generation works correctly

**Backup Strategy:**
- **Photo Storage**: S3 with cross-region replication enabled
- **Database Snapshots**: Hourly full snapshots, 15-minute incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync
- **CDN Caching**: Photos cached at multiple edge locations globally

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 30          # Max connections per application instance
      minimum-idle: 5                 # Minimum idle connections
      
      # Timeouts
      connection-timeout: 30000       # 30s - Max time to wait for connection from pool
      idle-timeout: 600000            # 10m - Idle connection timeout
      max-lifetime: 1800000           # 30m - Max connection lifetime
      
      # Monitoring
      leak-detection-threshold: 60000 # 60s - Detect connection leaks
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 3000        # 3s - Connection validation timeout
      
      # Register metrics
      register-mbeans: true
```

**Pool Sizing Calculation:**

```
Pool size = ((Core count × 2) + Effective spindle count)

For our service:
- 4 CPU cores per pod
- Spindle count = 1 (single DB instance)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 30 connections per pod (higher due to feed generation load)
- 50 pods × 30 = 1500 max connections to database
- Database max_connections: 2000 (20% headroom)
```

**Pool Exhaustion Mitigation:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 5 seconds, open circuit breaker
   - Fail fast instead of blocking

3. **PgBouncer (Connection Pooler):**
   - Place PgBouncer between app and PostgreSQL
   - PgBouncer pool: 300 connections
   - App pools can share PgBouncer connections efficiently

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database read replica to primary - 3 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 3 minutes
5. Verify photo service health and resume traffic - 2 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] S3 cross-region replication lag < 15 minutes
- [ ] Database replication lag < 15 minutes
- [ ] CDN origin failover configured
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture photo upload/download success rate
   - Document active photo count
   - Verify S3 replication lag (< 15 minutes)
   - Test photo upload/download (100 sample photos)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+10 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-6 min:** Update CDN origin to point to DR region
   - **T+6-8 min:** Warm up Redis cache from database
   - **T+8-9 min:** Verify all services healthy in DR region
   - **T+9-10 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+10 to T+20 minutes):**
   - Verify RTO < 10 minutes: ✅ PASS/FAIL
   - Verify RPO < 15 minutes: Check S3 replication lag at failure time
   - Test photo upload: Upload 1,000 test photos, verify 100% success
   - Test photo download: Download 1,000 test photos, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify feed generation: Check feed generation works correctly

5. **Data Integrity Verification:**
   - Compare photo count: Pre-failover vs post-failover (should match within 15 min)
   - Spot check: Verify 100 random photos accessible
   - Check photo metadata: Verify metadata matches pre-failover
   - Test edge cases: Popular photos, new uploads, story expiration

6. **Failback Procedure (T+20 to T+25 minutes):**
   - Restore primary region services
   - Sync S3 data from DR to primary
   - Verify replication lag < 15 minutes
   - Update DNS and CDN to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 10 minutes: Time from failure to service resumption
- ✅ RPO < 15 minutes: Maximum data loss (verified via S3 replication lag)
- ✅ Photo upload/download works: >99% photos accessible correctly
- ✅ No data loss: Photo count matches pre-failover (within 15 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Feed generation works: Feeds generated correctly

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Uploads Photo and It Appears in Feed

**Step-by-step:**

1. **User Action**: User selects photo, uploads via `POST /v1/posts` (multipart upload)
2. **Photo Service**: 
   - Receives photo, stores original in S3: `s3://uploads/user123/post_abc123/original.jpg`
   - Generates post ID: `post_abc123`
   - Stores metadata in PostgreSQL: `INSERT INTO posts (id, user_id, s3_key, created_at)`
   - Publishes to Kafka: `{"event": "post_created", "post_id": "post_abc123", "user_id": "user123"}`
3. **Response**: `201 Created` with `{"post_id": "post_abc123", "status": "processing"}`
4. **Image Processing Worker** (consumes Kafka):
   - Downloads original from S3
   - Generates sizes: Large (1080px), Medium (640px), Thumbnail (150px)
   - Uploads to CDN: `s3://cdn/post_abc123/large.jpg, medium.jpg, thumb.jpg`
   - Updates database: `UPDATE posts SET status = 'ready', cdn_urls = '...'`
5. **Fan-out Worker** (consumes Kafka):
   - Fetches user's followers: `SELECT follower_id FROM follows WHERE user_id = 'user123'` → 500 followers
   - For regular users (< 10K followers): Writes to feed cache
   - For celebrities: Skips fan-out (handled on read)
6. **Follower views feed** (5 minutes later):
   - Request: `GET /v1/feed?cursor=...`
   - Feed Service: Returns feed including post_abc123
   - Client: Requests thumbnail from CDN
   - CDN: Serves cached thumbnail (< 10ms)
7. **Result**: Photo appears in followers' feeds within 2 minutes

**Total latency: Upload ~5s, Processing ~30s, Fan-out ~10s, Total ~45s**

### Journey 2: Celebrity Post (Read-Time Fan-out)

**Step-by-step:**

1. **User Action**: Celebrity (1M followers) uploads photo
2. **Photo Service**: 
   - Stores photo, processes sizes
   - Publishes to Kafka
   - **Skips fan-out** (celebrity has > 10K followers)
3. **Celebrity post stored** in Redis: `ZADD celebrity_posts:user123 timestamp post_xyz789`
4. **Follower views feed**:
   - Request: `GET /v1/feed?cursor=...`
   - Feed Service:
     - Fetches regular feed from Redis (fan-out posts)
     - Checks if following celebrities: `SISMEMBER following_celebrities:follower1 user123` → true
     - Fetches celebrity posts: `ZRANGE celebrity_posts:user123 0 20`
     - Merges feeds (by timestamp)
   - Response: `200 OK` with merged feed
5. **Result**: Celebrity post appears in feed on read (no pre-computation)

**Total latency: ~50ms** (Redis lookups + merge)

### Failure & Recovery Walkthrough

**Scenario: S3 Service Degradation During Photo Upload**

**RTO (Recovery Time Objective):** < 5 minutes (automatic retry with exponential backoff)  
**RPO (Recovery Point Objective):** 0 (uploads queued, no data loss)

**Timeline:**

```
T+0s:    S3 starts returning 503 errors (throttling)
T+0-10s: Photo uploads fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Uploads queued to local buffer (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   S3 recovers, accepts requests
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Queued uploads processed from buffer
T+5min:  All queued uploads completed
```

**What degrades:**
- Photo uploads fail for 10-30 seconds
- Uploads queued in buffer (no data loss)
- Feed viewing unaffected (reads from S3/CDN)

**What stays up:**
- Photo viewing (reads from S3/CDN)
- Feed requests (Redis cache)
- User authentication
- All read operations

**What recovers automatically:**
- Circuit breaker closes after S3 recovery
- Queued uploads processed automatically
- No manual intervention required

**What requires human intervention:**
- Investigate S3 throttling root cause
- Review capacity planning
- Consider S3 request rate increases

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Request queuing prevents data loss
- Exponential backoff reduces S3 load
- Timeouts prevent thread exhaustion

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Photo       │  │ Feed        │  │ API         │           │  │
│  │  │ Service     │  │ Service     │  │ Service     │           │  │
│  │  │ (30 pods)   │  │ (50 pods)   │  │ (40 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  S3 Bucket (photo storage) - Primary             │          │  │
│  │  │  ──CRR──> S3 Bucket (DR region)                 │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  PostgreSQL (photo metadata, feeds) - Sharded              │ │  │
│  │  │  ──async repl──> PostgreSQL Replica (DR)                  │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (feed cache) - 50 nodes                    │ │  │
│  │  │  ──async repl──> Redis Replica (DR)                      │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  CDN (CloudFront) - Global edge network                    │ │  │
│  │  │  Origin: us-east-1 S3 (primary)                           │ │  │
│  │  │  Failover: us-west-2 S3 (DR)                             │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async replication──                            │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: us-west-2                                │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  S3 Bucket (CRR destination, read-only until failover)            │ │
│  │  PostgreSQL Replica (async replication from us-east-1)            │ │
│  │  Redis Replica (async replication + snapshots)                   │ │
│  │  Photo Service (5 pods, minimal for DR readiness)                  │ │
│  │  Feed Service (10 pods, minimal for DR readiness)                 │ │
│  │  API Service (5 pods, minimal for DR readiness)                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **S3 Cross-Region Replication (CRR):**
   - **Primary:** us-east-1 maintains photo storage
   - **Replication:** Async replication to DR region (us-west-2)
   - **Replication Lag:** < 15 minutes (acceptable for photos, RPO < 15 min)
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 15 minutes for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

3. **Redis Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Snapshots:** Every 5 minutes to S3 for faster recovery
   - **Cache Warm-up:** On failover, restore from latest snapshot

4. **CDN Origin Failover:**
   - **Primary:** CloudFront origin points to us-east-1 S3
   - **Failover:** On primary failure, CloudFront automatically fails over to us-west-2 S3
   - **RTO:** < 30 seconds (automatic CDN failover)

5. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **S3 CRR Lag** | Stale photos in DR region | Acceptable for DR (RPO < 15 min), popular photos cached in CDN |
| **CDN Cache** | CDN cache may be cold after failover | Acceptable for DR (cache warms up quickly), popular photos already cached |
| **Feed Generation** | Feeds may be slightly stale | Acceptable for DR (15 min staleness), feeds regenerate on access |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Data Consistency** | Eventual consistency across regions | Acceptable for photos (slight staleness), strong consistency within region |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle traffic)
   - Recovery: Automatic (Kubernetes restarts pod)
   - RTO: < 2 minutes (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 10 minutes (manual process)
   - RPO: < 15 minutes (S3 CRR lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale photos
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $800,000/month (full infrastructure)
- **DR Region:** $240,000/month (reduced capacity, 30% of primary)
- **S3 CRR:** $4,000/month (cross-region replication)
- **Replication Bandwidth:** $8,000/month (cross-region data transfer)
- **Total Multi-Region Cost:** $1,052,000/month (32% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** S3 CRR provides additional backup layer

**Trade-offs:**
- **Cost:** 32% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)
- **Consistency:** Eventual consistency (acceptable for photos, slight staleness)

---

## 3. Failure Handling

### Scenario 1: Redis (Feed Cache) Down

**RTO (Recovery Time Objective):** < 30 seconds (automatic failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, cache may be cold)

**Detection:**
- How to detect: Grafana alert "Redis cluster down"
- Alert thresholds: 3 consecutive health check failures
- Monitoring: Check Redis cluster status endpoint

**Impact:**
- Affected services: Feed generation service (cache miss rate 100%)
- User impact: Feed loads slower (must regenerate from database)
- Degradation: System continues operating, slower responses

**Recovery Steps:**
1. **T+0s**: Alert received, Redis cluster down detected
2. **T+5s**: Verify Redis cluster status via AWS console
3. **T+10s**: Check Redis primary node health
4. **T+15s**: If primary down, promote replica to primary (automatic)
5. **T+20s**: Verify Redis cluster healthy
6. **T+25s**: Feed service reconnects to Redis
7. **T+30s**: Normal operations resume

**RTO:** < 30 seconds
**RPO:** 0 (no data loss, cache rebuilds from database)

**Prevention:**
- Redis cluster with automatic failover
- Health checks every 10 seconds
- Circuit breaker falls back to database on Redis failure

**Mitigation:**

1. **Redis Cluster with Replicas:**
   ```
   - 3 replicas per shard
   - Automatic failover (< 30 seconds)
   - Sentinel for monitoring
   ```

2. **Fallback to Database:**
   ```java
   public List<Post> getFeed(Long userId) {
       try {
           return getFeedFromRedis(userId);
       } catch (RedisException e) {
           log.warn("Redis unavailable, falling back to DB");
           return regenerateFeedFromDB(userId);
       }
   }
   
   private List<Post> regenerateFeedFromDB(Long userId) {
       // Get following list
       List<Long> following = followRepository.getFollowing(userId);
       
       // Get recent posts from all following
       return postRepository.findRecentByAuthors(following, 50);
   }
   ```

3. **Local Cache:**
   ```java
   // Cache recent feed in memory (short TTL)
   @Cacheable(value = "feed", key = "#userId", ttl = 60)
   public List<Post> getFeed(Long userId) { ... }
   ```

**Recovery:**
- Redis recovers → Warm cache gradually
- Don't thundering herd the database

### Scenario 2: Feed Cache Miss Storm

**Problem:** Cache expires for many users simultaneously.

```java
@Service
public class FeedCacheService {
    
    // Add jitter to TTL to prevent synchronized expiration
    public void cacheFeed(Long userId, List<String> postIds) {
        long baseTtl = 86400; // 24 hours
        long jitter = ThreadLocalRandom.current().nextLong(0, 3600); // 0-1 hour
        
        redis.opsForZSet().addAll("feed:" + userId, postIds);
        redis.expire("feed:" + userId, baseTtl + jitter, TimeUnit.SECONDS);
    }
    
    // Proactive cache warming
    @Scheduled(cron = "0 0 * * * *") // Every hour
    public void warmCachesForActiveUsers() {
        List<Long> activeUsers = getRecentlyActiveUsers();
        
        for (Long userId : activeUsers) {
            if (!redis.hasKey("feed:" + userId)) {
                // Warm cache before user requests
                regenerateAndCacheFeed(userId);
            }
        }
    }
}
```

### Scenario 3: S3 Outage

**Problem:** Image storage unavailable.

```java
@Service
public class ImageStorageService {
    
    private final S3Client primaryS3;
    private final S3Client backupS3;
    
    public String uploadImage(String key, byte[] data) {
        try {
            // Try primary
            primaryS3.putObject(key, data);
            return getPrimaryUrl(key);
        } catch (S3Exception e) {
            log.warn("Primary S3 failed, using backup");
            
            // Fallback to backup region
            backupS3.putObject(key, data);
            return getBackupUrl(key);
        }
    }
    
    public byte[] getImage(String key) {
        try {
            return primaryS3.getObject(key);
        } catch (S3Exception e) {
            // Try backup
            return backupS3.getObject(key);
        }
    }
}
```

---

## 4. Photo Durability

### Durability Strategy

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

### Checksum Verification

```java
@Service
public class ImageIntegrityService {
    
    public void verifyUpload(String key, String expectedMd5) {
        S3Object object = s3Client.getObject(key);
        String actualMd5 = calculateMd5(object.getContent());
        
        if (!expectedMd5.equals(actualMd5)) {
            throw new IntegrityException("MD5 mismatch");
        }
    }
    
    @Scheduled(cron = "0 0 3 * * *") // Daily at 3 AM
    public void verifyRandomSample() {
        List<String> sampleKeys = getRandomImageKeys(1000);
        
        for (String key : sampleKeys) {
            ImageMetadata metadata = getMetadata(key);
            String actualMd5 = calculateMd5(s3Client.getObject(key));
            
            if (!metadata.getMd5().equals(actualMd5)) {
                alertService.sendCriticalAlert("Image corruption: " + key);
                restoreFromBackup(key);
            }
        }
    }
}
```

## 3.5. Simulation (End-to-End User Journeys)

### Journey 1: User Uploads Photo (Happy Path)

**Step-by-step:**

1. **User Action**: Selects photo, taps upload
2. **Upload Service**: 
   - Receives photo (5MB JPEG)
   - Generates multiple sizes (thumb, medium, large)
   - Stores in S3 with CDN distribution
   - Stores metadata in database
3. **Feed Service**: 
   - Updates user's feed
   - Notifies followers (fan-out)
4. **Response**: Photo visible in feed within 2 seconds

### Journey 2: User Views Feed

**Step-by-step:**

1. **User Action**: Opens app, views home feed
2. **Feed Service**: 
   - Fetches feed from Redis cache (if cached)
   - If cache miss: Regenerates from database
   - Returns list of post IDs
3. **Photo Service**: 
   - Fetches photo URLs for posts
   - CDN serves images from edge cache
4. **Response**: Feed loads with images in < 500ms

### High-Load / Contention Scenario: Celebrity Photo Goes Viral

```
Scenario: Celebrity posts photo, millions view simultaneously
Time: Breaking news event (peak traffic)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Celebrity posts photo (50M followers)                │
│ - Photo ID: photo_xyz789                                   │
│ - 10M users view photo within 5 minutes                   │
│ - Expected: ~1K views/minute normal traffic               │
│ - 10,000x traffic spike                                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-5min: Image Request Storm                             │
│ - 10M users request photo (3 sizes each)                  │
│ - Total requests: 30M image requests                     │
│ - Request rate: 6M requests/minute                       │
│ - CDN edge locations: 200+ globally                      │
│ - Each location: ~30K requests/minute                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. CDN Caching:                                           │
│    - Pre-warmed: Photo cached at all edge locations       │
│    - Cache hit rate: 99.9% (served from edge)            │
│    - Origin requests: 0.1% (30K requests total)           │
│    - Edge latency: < 20ms (excellent)                     │
│                                                              │
│ 2. Origin Load (for cache misses):                       │
│    - S3: Handles 30K requests (well within limits)        │
│    - S3 can handle 3,500 requests/second (210K/min)      │
│    - Utilization: 0.01% (negligible)                     │
│                                                              │
│ 3. Feed Cache:                                            │
│    - 50M followers' feeds need update                    │
│    - Fan-out: Updates 50M feed caches in Redis           │
│    - Batch processing: 10K feeds per batch                │
│    - Total time: 5,000 batches × 50ms = 4 minutes        │
│                                                              │
│ 4. Database Load:                                         │
│    - Single photo record write (no issue)                │
│    - Feed reads: Mostly from Redis cache (95% hit)        │
│    - Database: Handles 5% cache misses only              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 10M users view photo successfully                  │
│ - 99.9% requests served from CDN (< 20ms latency)       │
│ - P99 latency: 50ms (excellent)                          │
│ - All 50M followers see photo in feed within 5 minutes  │
│ - No service degradation                                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Photo Deletion During Active Viewing

```
Scenario: User deletes photo while others are viewing it
Edge Case: Photo deleted but CDN cache still serving it

┌─────────────────────────────────────────────────────────────┐
│ T+0s: User deletes photo                                  │
│ - Photo ID: photo_abc123                                   │
│ - DELETE /photos/photo_abc123                            │
│ - Service marks as deleted in database                     │
│ - Status: ACTIVE → DELETED                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+1s: Another user views feed (concurrent)               │
│ - Feed contains photo_abc123 (not yet updated)           │
│ - Request: GET /photos/photo_abc123/large.jpg           │
│ - CDN: Photo still cached (not invalidated)              │
│ - Photo served from CDN cache                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Stale CDN Cache                               │
│                                                              │
│ Problem: Photo deleted but CDN cache not invalidated     │
│ - Photo accessible via direct URL (CDN cache)            │
│ - Photo visible in feeds (cached feed data)              │
│ - User thinks photo is deleted but it's still accessible │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Multi-Layer Invalidation                        │
│                                                              │
│ 1. Database Mark as Deleted:                             │
│    - Photo record: status = DELETED                      │
│    - Soft delete: Photo not removed immediately          │
│    - Grace period: 30 days (allows CDN invalidation)     │
│                                                              │
│ 2. CDN Cache Invalidation:                               │
│    - On delete: Invalidate CDN cache for all sizes      │
│    - CDN API: Purge cache for photo URLs                │
│    - Invalidation time: 1-5 minutes                     │
│    - After invalidation: 404 Not Found                   │
│                                                              │
│ 3. Feed Cache Invalidation:                              │
│    - On delete: Remove photo from all feeds             │
│    - Redis: Remove photo_id from feed:user_id sorted set│
│    - Immediate: Feeds updated in real-time              │
│                                                              │
│ 4. Hard Delete (after grace period):                     │
│    - Background job: Delete photos marked DELETED        │
│    - Condition: deleted_at > 30 days                    │
│    - Action: Delete from S3, remove from database        │
│    - Cleanup: Permanent removal                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Photo removed from feeds immediately                   │
│ - CDN cache invalidated within 5 minutes                │
│ - Photo inaccessible via direct URL after invalidation  │
│ - Hard deleted after 30-day grace period                │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Optimization

**Current Monthly Cost:** $800,000
**Target Monthly Cost:** $560,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **CDN Bandwidth:** $320,000/month (40%) - Largest single component
2. **Storage (S3):** $240,000/month (30%) - Large photo storage
3. **Image Processing (EC2):** $120,000/month (15%) - Image encoding costs

**Optimization Strategies (Ranked by Impact):**

1. **Image Format Optimization (40% savings on CDN bandwidth):**
   - **Current:** JPEG format (100% bandwidth)
   - **Optimization:** 
     - Use WebP format (30-40% smaller than JPEG)
     - Use AVIF format for modern browsers (50% smaller than JPEG)
     - Serve optimal format based on user agent
   - **Savings:** $128,000/month (40% of $320K CDN)
   - **Trade-off:** Slight CPU overhead for encoding (negligible)

2. **Tiered Storage (50% savings on old content):**
   - **Current:** All photos in S3 Standard
   - **Optimization:** 
     - Move photos not accessed > 180 days to S3 Glacier (80% cheaper)
     - Move photos not accessed > 30 days to S3 Infrequent Access (50% cheaper)
   - **Savings:** $72,000/month (30% of $240K storage)
   - **Trade-off:** Slower access to old photos (acceptable for archive)

3. **Spot Instances for Image Processing (70% savings):**
   - **Current:** On-demand instances for image processing jobs
   - **Optimization:** Use Spot instances with checkpointing
   - **Savings:** $84,000/month (70% of $120K image processing)
   - **Trade-off:** Possible interruptions (acceptable, processing is batch job)

4. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for stable workloads
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $32,000/month (40% of $80K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

5. **CDN Optimization (20% savings):**
   - **Current:** High CDN bandwidth costs
   - **Optimization:** 
     - Aggressive caching (increase cache hit rate to 95%)
     - Origin shield (reduce origin requests by 50%)
   - **Savings:** $64,000/month (20% of $320K CDN)
   - **Trade-off:** Slight complexity increase (acceptable)

6. **Image Compression:**
   - **Current:** Basic compression
   - **Optimization:** 
     - Strip EXIF data (reduce size by 10%)
     - Optimize compression settings
   - **Savings:** $24,000/month (10% of $240K storage)
   - **Trade-off:** Slight quality reduction (acceptable, users won't notice)

**Total Potential Savings:** $404,000/month (51% reduction)
**Optimized Monthly Cost:** $396,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Photo Upload | $0.000016 | $0.000008 | 50% |
| Photo Storage (per GB/month) | $0.023 | $0.011 | 52% |
| Photo Download (per GB) | $0.08 | $0.041 | 49% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Image Format Optimization, Tiered Storage → $200K savings
2. **Phase 2 (Month 2):** Spot instances, Reserved Instances → $116K savings
3. **Phase 3 (Month 3):** CDN Optimization, Image Compression → $88K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor photo quality (ensure no degradation)
- Monitor storage costs (target < $168K/month)
- Review and adjust quarterly

### Infrastructure Cost Breakdown

| Component        | Cost Driver                | Optimization Strategy                |
| ---------------- | -------------------------- | ------------------------------------ |
| CDN bandwidth    | GB transferred             | Aggressive caching, WebP format      |
| Storage          | GB stored                  | Tiered storage, image compression    |
| Image processing | CPU hours                  | Spot instances, batch processing     |
| Database         | IOPS, storage              | Read replicas, connection pooling    |

### Image Optimization

```java
@Service
public class ImageOptimizationService {
    
    public byte[] optimizeForStorage(byte[] original) {
        // 1. Convert to WebP (30-40% smaller than JPEG)
        byte[] webp = convertToWebP(original, quality = 85);
        
        // 2. Strip unnecessary metadata
        webp = stripExifData(webp);
        
        // 3. Apply compression
        return compress(webp);
    }
    
    public String getOptimalFormat(String userAgent) {
        if (supportsWebP(userAgent)) {
            return "webp";
        } else if (supportsAVIF(userAgent)) {
            return "avif";
        }
        return "jpeg";
    }
}
```

### Tiered Storage

```java
@Service
public class StorageTieringService {
    
    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void tierOldContent() {
        // Move old, unpopular images to cheaper storage
        List<String> coldImages = imageRepository.getImagesNotAccessed(
            Duration.ofDays(180)
        );
        
        for (String imageKey : coldImages) {
            // Move from S3 Standard to S3 Glacier
            s3Client.copyObject(
                "images-hot", imageKey,
                "images-cold", imageKey,
                StorageClass.GLACIER
            );
            
            // Delete from hot storage
            s3Client.deleteObject("images-hot", imageKey);
            
            // Update metadata
            imageRepository.setStorageTier(imageKey, "cold");
        }
    }
}
```

---

## 6. Capacity Planning

### Calculation Example

**Assumptions:**
- 500M daily active users
- 50M uploads/day
- Average image: 2MB original, 500KB processed (all sizes)

**Storage Growth:**
```
Daily uploads: 50M × 2MB = 100TB originals
Processed (all sizes): 50M × 500KB = 25TB/day
With replication (3x): 375TB/day
Monthly: 11.25PB
Yearly: 135PB
```

**Image Processing Capacity:**
```
Uploads/day: 50M
Processing time: 5 seconds average
Worker capacity: 720 images/hour = 17,280 images/day
Workers needed: 50M / 17,280 = 2,894 workers
With headroom (50%): 4,341 workers
```

**Redis Capacity:**
```
Active users: 500M
Feed per user: 500 posts × 20 bytes = 10KB
Total feed cache: 500M × 10KB = 5TB
With overhead: 7.5TB cluster
```

---

## 7. Health Checks

### Service Health Endpoints

```java
@RestController
@RequestMapping("/health")
public class HealthController {
    
    @GetMapping("/live")
    public ResponseEntity<String> liveness() {
        return ResponseEntity.ok("OK");
    }
    
    @GetMapping("/ready")
    public ResponseEntity<HealthStatus> readiness() {
        HealthStatus status = HealthStatus.builder()
            .postgres(checkPostgres())
            .redis(checkRedis())
            .cassandra(checkCassandra())
            .s3(checkS3())
            .kafka(checkKafka())
            .build();
        
        if (status.isHealthy()) {
            return ResponseEntity.ok(status);
        }
        return ResponseEntity.status(503).body(status);
    }
    
    private ComponentHealth checkRedis() {
        try {
            redis.ping();
            return ComponentHealth.healthy();
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
    
    private ComponentHealth checkS3() {
        try {
            s3Client.headBucket("images");
            return ComponentHealth.healthy();
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
}
```

### Feed Health Monitoring

```java
@Component
public class FeedHealthMonitor {
    
    @Scheduled(fixedRate = 60000)
    public void checkFeedHealth() {
        // Sample random users and check feed freshness
        List<Long> sampleUsers = getRandomActiveUsers(100);
        
        int staleFeeds = 0;
        for (Long userId : sampleUsers) {
            List<Post> feed = feedService.getFeed(userId, null, 10);
            
            if (feed.isEmpty() || isStale(feed.get(0))) {
                staleFeeds++;
            }
        }
        
        double staleRate = staleFeeds / 100.0;
        metrics.gauge("feed.stale_rate", staleRate);
        
        if (staleRate > 0.1) {
            alertService.sendAlert("High feed stale rate: " + staleRate);
        }
    }
    
    private boolean isStale(Post post) {
        // Feed is stale if newest post is > 24 hours old
        return post.getCreatedAt().isBefore(Instant.now().minus(24, ChronoUnit.HOURS));
    }
}
```

---

## Summary

| Aspect              | Strategy                                              |
| ------------------- | ----------------------------------------------------- |
| Monitoring          | Prometheus + Grafana, custom Instagram metrics        |
| Scaling             | Pre-scale for events, graceful degradation            |
| Failure handling    | Redis replicas, S3 cross-region, DB fallback          |
| Durability          | MD5 verification, cross-region replication            |
| Cost optimization   | WebP format, tiered storage, spot instances           |
| Capacity planning   | 4.3K image workers, 7.5TB Redis cluster               |

