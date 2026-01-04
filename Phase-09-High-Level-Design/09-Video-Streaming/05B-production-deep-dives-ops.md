# Video Streaming - Production Deep Dives (Operations)

## Overview

This document covers operational aspects: monitoring & metrics, scaling strategies, failure handling, viral content management, and cost optimization.

---

## 1. Monitoring & Metrics

### Key Metrics

```java
@Component
public class VideoMetrics {
    
    private final MeterRegistry registry;
    
    // Streaming metrics
    private final Timer startupLatency;
    private final Counter rebufferEvents;
    private final DistributionSummary bitrateDistribution;
    
    // Processing metrics
    private final Timer transcodingDuration;
    private final Counter transcodingFailures;
    
    public VideoMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.startupLatency = Timer.builder("video.startup.latency")
            .description("Time to first frame")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.rebufferEvents = Counter.builder("video.rebuffer.count")
            .description("Number of rebuffer events")
            .register(registry);
        
        this.bitrateDistribution = DistributionSummary.builder("video.bitrate")
            .description("Bitrate distribution")
            .publishPercentiles(0.5, 0.95)
            .register(registry);
    }
    
    public void recordPlaybackStart(long startupMs, String resolution) {
        startupLatency.record(startupMs, TimeUnit.MILLISECONDS);
        registry.counter("video.plays", "resolution", resolution).increment();
    }
    
    public void recordRebuffer(String videoId, long durationMs) {
        rebufferEvents.increment();
        registry.timer("video.rebuffer.duration").record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Metric                  | Warning    | Critical   | Action                        |
| ----------------------- | ---------- | ---------- | ----------------------------- |
| Startup P99             | > 3s       | > 5s       | Check CDN/origin              |
| Rebuffer rate           | > 0.5%     | > 2%       | Check bandwidth/CDN           |
| Transcoding queue       | > 1 hour   | > 4 hours  | Scale workers                 |
| CDN hit rate            | < 90%      | < 80%      | Check cache config            |

### Grafana Dashboard Queries

```promql
# Video startup latency P99
histogram_quantile(0.99, sum(rate(video_startup_latency_bucket[5m])) by (le))

# Rebuffer rate
sum(rate(video_rebuffer_count_total[5m])) / sum(rate(video_plays_total[5m]))

# Transcoding queue depth
sum(video_transcoding_queue_depth)

# CDN hit rate
sum(rate(cdn_cache_hits_total[5m])) / sum(rate(cdn_requests_total[5m]))

# Average bitrate by resolution
avg(video_bitrate) by (resolution)
```

---

## 2. Scaling Strategies

### Viral Video Handling

**The Challenge:**
- Normal video: 1,000 views/day
- Viral video: 10,000,000 views/day (10,000x spike)

**Solution: Multi-layer Defense**

```
┌─────────────────────────────────────────────────────────────────┐
│                    VIRAL VIDEO HANDLING                          │
└─────────────────────────────────────────────────────────────────┘

1. CDN Edge (First Line)
   - 85% of requests served from edge
   - Auto-scales with traffic
   - No origin impact for cached content

2. Origin Shield (Second Line)
   - Aggregates cache misses
   - Prevents thundering herd to origin
   - Large cache capacity

3. Origin (Last Resort)
   - Only serves ~1% of requests
   - Auto-scaling enabled
   - Circuit breaker if overwhelmed

4. Proactive Measures:
   - Detect trending early (view velocity)
   - Pre-warm CDN cache
   - Increase TTL for viral content
```

**Implementation:**

```java
@Service
public class ViralDetectionService {
    
    @Scheduled(fixedRate = 60000) // Every minute
    public void detectViralVideos() {
        // Get videos with high view velocity
        List<String> trending = getHighVelocityVideos(threshold: 1000); // 1000 views/min
        
        for (String videoId : trending) {
            // 1. Increase CDN cache TTL
            cdnClient.setCacheTTL(videoId, Duration.ofHours(24));
            
            // 2. Pre-warm all edge locations
            cdnClient.prefetchToAllEdges(videoId);
            
            // 3. Alert ops team
            alertService.sendViralAlert(videoId);
        }
    }
}
```

### Live Event Scaling (100M Concurrent Viewers)

**Pre-event Preparation:**
```
- Scale CDN capacity 3x (buffer)
- Pre-warm edge caches with event page
- Set up dedicated origin servers for event
- Enable aggressive caching (1-second TTL for live)
```

**During Event:**
```
- Live transcoding at multiple qualities
- 2-second segment duration (lower latency)
- Multiple redundant encoders
- Real-time monitoring dashboard
```

**Architecture for Live:**
```
Live Feed → Encoder Farm → Origin → CDN → Viewers
                │
                └── Redundant encoders (N+2)
                └── Multiple ingest points
                └── Automatic failover
```

### Transcoding Optimization

**Current State:**
- 500K uploads/day
- 30 minutes average processing time
- 1,000 transcoding workers

**Optimization Strategies:**

1. **Parallel Transcoding:**
   ```java
   // Instead of sequential:
   // 240p → 360p → 480p → 720p → 1080p (150 minutes)
   
   // Do parallel:
   // All resolutions simultaneously (30 minutes)
   
   CompletableFuture.allOf(
       transcode(video, "240p"),
       transcode(video, "360p"),
       transcode(video, "720p"),
       transcode(video, "1080p")
   ).join();
   ```

2. **GPU Acceleration:**
   ```
   CPU transcoding: 2x real-time
   GPU transcoding: 10x real-time
   
   Cost: GPU instances cost 3x more
   Benefit: 5x faster processing
   ROI: Positive for high-volume
   ```

3. **Priority Queues:**
   ```java
   if (creator.isVerified() || video.isPotentiallyViral()) {
       queue = "high-priority";
   }
   ```

4. **Progressive Availability:**
   ```
   720p ready → Video available (720p only)
   1080p ready → Add 1080p option
   480p ready → Add 480p option
   ```

### Backup and Recovery

**RPO (Recovery Point Objective):** < 1 hour
- Maximum acceptable data loss
- Based on video upload replication frequency (hourly)
- Video metadata replicated to read replicas within 1 hour
- CDN content cached at multiple edge locations (no data loss)

**RTO (Recovery Time Objective):** < 5 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes CDN failover, database promotion, and traffic rerouting

**Backup Strategy:**
- **Video Storage**: S3 with cross-region replication enabled
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 30 days
- **CDN Caching**: Content cached at multiple edge locations globally
- **Transcoding Outputs**: Stored in S3 with versioning enabled

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 25          # Max connections per application instance
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
- With safety margin: 25 connections per pod (higher due to video metadata load)
- 30 pods × 25 = 750 max connections to database
- Database max_connections: 1000 (20% headroom)
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
   - PgBouncer pool: 200 connections
   - App pools can share PgBouncer connections efficiently

**Restore Steps:**
1. Detect primary region failure (health checks) - 30 seconds
2. Promote database read replica to primary - 1 minute
3. Update CDN origin to point to secondary region - 1 minute
4. Verify video streaming service health - 1 minute
5. Resume traffic and verify playback - 1.5 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] S3 cross-region replication lag < 1 hour
- [ ] CDN origin failover configured
- [ ] Database replication lag < 1 hour
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture video playback success rate
   - Document active video count
   - Verify S3 replication lag (< 1 hour)
   - Test video playback (100 sample videos)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote secondary region to primary
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-4 min:** Update CDN origin to point to DR region
   - **T+4-4.5 min:** Verify all services healthy in DR region
   - **T+4.5-5 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+5 to T+15 minutes):**
   - Verify RTO < 5 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 hour: Check S3 replication lag at failure time
   - Test video playback: Play 1,000 test videos, verify >99% success
   - Test video upload: Upload 100 test videos, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify CDN: Check CDN cache hit rates return to normal

5. **Data Integrity Verification:**
   - Compare video count: Pre-failover vs post-failover (should match within 1 hour)
   - Spot check: Verify 100 random videos playable
   - Check video metadata: Verify metadata matches pre-failover
   - Test edge cases: Popular videos, new uploads, transcoding jobs

6. **Failback Procedure (T+15 to T+20 minutes):**
   - Restore primary region services
   - Sync S3 data from DR to primary
   - Verify replication lag < 1 hour
   - Update DNS and CDN to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes: Time from failure to service resumption
- ✅ RPO < 1 hour: Maximum data loss (verified via S3 replication lag)
- ✅ Video playback works: >99% videos playable correctly
- ✅ No data loss: Video count matches pre-failover (within 1 hour window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ CDN functional: Cache hit rates return to normal

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Uploads and Watches Video

**Step-by-step:**

1. **User Action**: User uploads video file via `POST /v1/videos/upload` (chunked upload)
2. **Upload Service**: 
   - Receives chunks, stores in S3: `s3://uploads/video_abc123/chunk_001, chunk_002, ...`
   - After all chunks: Assembles file, validates format
   - Creates video record: `INSERT INTO videos (id, user_id, status, s3_key)`
   - Publishes to Kafka: `{"event": "video_uploaded", "video_id": "video_abc123"}`
3. **Response**: `201 Created` with `{"video_id": "video_abc123", "status": "processing"}`
4. **Transcoding Worker** (consumes Kafka):
   - Downloads original from S3
   - Transcodes to multiple resolutions: 240p, 360p, 480p, 720p, 1080p
   - Uploads segments to CDN: `s3://cdn/video_abc123/720p/segment_001.ts, ...`
   - Creates HLS playlist: `playlist.m3u8`
   - Updates database: `UPDATE videos SET status = 'ready', cdn_url = '...'`
5. **User watches video** (30 minutes later):
   - Request: `GET /v1/videos/video_abc123/stream`
   - Video Service: Returns HLS playlist URL
   - Client: Requests segments from CDN
   - CDN: Serves segments (cached at edge)
6. **Result**: Video streams smoothly at 720p, < 100ms latency

**Total latency: Upload ~5 min, Transcoding ~10 min, Streaming < 100ms**

### Journey 2: Adaptive Bitrate Streaming

**Step-by-step:**

1. **User Action**: User starts watching video on mobile (variable network)
2. **Client**: Requests master playlist: `GET /v1/videos/video_abc123/playlist.m3u8`
3. **Video Service**: Returns playlist with all resolutions
4. **Client**: Monitors network bandwidth, starts with 480p
5. **Network improves**: Client switches to 720p segment
6. **Network degrades**: Client switches to 360p segment
7. **Result**: Smooth playback with automatic quality adjustment

**Total latency: < 50ms** (CDN edge serving)

### Failure & Recovery Walkthrough

**Scenario: Transcoding Worker Failure During Processing**

**RTO (Recovery Time Objective):** < 10 minutes (worker restart + job re-queue)  
**RPO (Recovery Point Objective):** 0 (job re-queued, no data loss)

**Timeline:**

```
T+0s:    Transcoding worker 5 processing video_xyz789 (50% complete)
T+30s:   Worker 5 crashes (OOM during 4K transcoding)
T+30s:   Kafka consumer group detects worker failure
T+35s:   Job re-queued to Kafka (at-least-once delivery)
T+40s:   Worker 6 picks up re-queued job
T+2min:  Worker 6 resumes transcoding from checkpoint
T+10min: Transcoding completes, video ready
```

**What degrades:**
- Video_xyz789 transcoding delayed by ~10 minutes
- Other videos continue processing normally
- No data loss (job re-queued)

**What stays up:**
- 19 other workers continue transcoding
- Video uploads continue
- Streaming unaffected (existing videos)
- CDN serving continues

**What recovers automatically:**
- Kubernetes restarts failed worker
- Kafka re-queues job
- New worker resumes from checkpoint
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of crash
- Review worker resource allocation
- Check for systemic issues

**Cascading failure prevention:**
- Job checkpointing prevents work loss
- Worker isolation (one crash doesn't affect others)
- Circuit breakers prevent retry storms
- Dead letter queue for failed jobs

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Upload     │  │ Transcoding │  │ API         │           │  │
│  │  │ Service    │  │ Service     │  │ Service     │           │  │
│  │  │ (20 pods)  │  │ (50 pods)   │  │ (30 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  S3 Bucket (video storage) - Primary              │          │  │
│  │  │  ──CRR──> S3 Bucket (DR region)                  │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  PostgreSQL (video metadata) - Sharded                    │ │  │
│  │  │  ──async repl──> PostgreSQL Replica (DR)                 │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  CDN (CloudFront) - Global edge network                   │ │  │
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
│  │  S3 Bucket (CRR destination, read-only until failover)          │ │
│  │  PostgreSQL Replica (async replication from us-east-1)          │ │
│  │  Upload Service (5 pods, minimal for DR readiness)              │ │
│  │  Transcoding Service (10 pods, minimal for DR readiness)        │ │
│  │  API Service (5 pods, minimal for DR readiness)                │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **S3 Cross-Region Replication (CRR):**
   - **Primary:** us-east-1 maintains video storage
   - **Replication:** Async replication to DR region (us-west-2)
   - **Replication Lag:** < 1 hour (acceptable for videos, RPO < 1 hour)
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 1 hour for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

3. **CDN Origin Failover:**
   - **Primary:** CloudFront origin points to us-east-1 S3
   - **Failover:** On primary failure, CloudFront automatically fails over to us-west-2 S3
   - **RTO:** < 30 seconds (automatic CDN failover)

4. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **S3 CRR Lag** | Stale videos in DR region | Acceptable for DR (RPO < 1 hour), popular videos cached in CDN |
| **CDN Cache** | CDN cache may be cold after failover | Acceptable for DR (cache warms up quickly), popular videos already cached |
| **Transcoding** | Transcoding jobs may be delayed | Acceptable for DR, jobs will process eventually |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Data Consistency** | Eventual consistency across regions | Acceptable for videos (slight staleness), strong consistency within region |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle traffic)
   - Recovery: Automatic (Kubernetes restarts pod)
   - RTO: < 2 minutes (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 5 minutes (manual process)
   - RPO: < 1 hour (S3 CRR lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale videos
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $2,000,000/month (full infrastructure)
- **DR Region:** $600,000/month (reduced capacity, 30% of primary)
- **S3 CRR:** $10,000/month (cross-region replication)
- **Replication Bandwidth:** $20,000/month (cross-region data transfer)
- **Total Multi-Region Cost:** $2,630,000/month (32% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** S3 CRR provides additional backup layer

**Trade-offs:**
- **Cost:** 32% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)
- **Consistency:** Eventual consistency (acceptable for videos, slight staleness)

---

## 3. Failure Handling

### Scenario 1: CDN Edge Failure

**RTO (Recovery Time Objective):** < 30 seconds (automatic DNS failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, content cached at other edges)

**Detection:**
- How to detect: Grafana alert "CDN edge location down"
- Alert thresholds: Edge location health check failures > 3
- Monitoring: Check CDN edge status dashboard

**Impact:**
- Affected services: Video playback for users in NYC region
- User impact: Automatic failover to DC edge (+20ms latency)
- Degradation: Slightly higher latency, no service interruption

**Recovery Steps:**
1. **T+0s**: Alert received, NYC edge location down
2. **T+5s**: Verify edge location status (CDN provider dashboard)
3. **T+10s**: Anycast routing automatically routes to DC edge
4. **T+20s**: Traffic flows to DC edge, users reconnect
5. **T+30s**: Normal operations resume (DC edge serving NYC traffic)

**RTO:** < 30 seconds
**RPO:** 0 (no data loss, content cached at other edges)

**Prevention:**
- Multiple edge locations per region
- Automatic anycast failover
- Health checks every 10 seconds

**Problem:** Major edge location (NYC) goes offline during prime time.

```java
// DNS-based failover
// Before: Users in NYC → NYC edge
// After: Users in NYC → DC edge (next closest)

// Anycast routing handles this automatically:
// - NYC edge stops announcing routes
// - Traffic routes to next closest edge
// - Failover time: < 30 seconds

// Impact mitigation:
// - Slightly higher latency (NYC → DC: +20ms)
// - No service interruption
// - Quality may temporarily dip
```

### Scenario 2: Origin Storage Outage

**RTO (Recovery Time Objective):** < 1 minute (automatic failover to replica region)

**Detection:**
- How to detect: Grafana alert "S3 origin storage errors"
- Alert thresholds: > 5% error rate for 30 seconds
- Monitoring: Check S3 bucket health and availability

**Impact:**
- Affected services: Video uploads, new video playback
- User impact: Existing cached videos continue playing
- Degradation: New uploads fail, uncached videos unavailable

**Recovery Steps:**
1. **T+0s**: Alert received, S3 origin storage errors detected
2. **T+10s**: Verify S3 bucket status (AWS console)
3. **T+20s**: Confirm primary region S3 outage
4. **T+30s**: Update CDN origin to point to secondary region S3
5. **T+45s**: Verify CDN can fetch from secondary region
6. **T+60s**: Normal operations resume (secondary region serving)

**RTO:** < 1 minute
**RPO:** < 1 hour (S3 cross-region replication lag)

**Prevention:**
- S3 cross-region replication enabled
- Multiple CDN origins configured
- Health checks every 10 seconds  
**RPO (Recovery Point Objective):** < 5 minutes (async replication lag)

**Problem:** S3 us-east-1 has partial outage.

```java
public String getVideoSegment(String videoId, String resolution, int segment) {
    String primaryKey = "encoded/" + videoId + "/" + resolution + "/segment_" + segment + ".ts";
    
    try {
        // Try primary region
        return s3Client.getObject(primaryKey);
    } catch (S3Exception e) {
        // Fallback to replica region
        log.warn("Primary S3 failed, trying replica");
        return s3ClientReplica.getObject(primaryKey);
    }
}

// CDN configuration:
// - Origin group with failover
// - Primary: s3.us-east-1
// - Secondary: s3.us-west-2
// - Automatic failover on 5xx errors
```

### Scenario 3: Transcoding Pipeline Backup

**Problem:** Processing queue grows to 100K videos (normally 10K).

```java
@Component
public class TranscodingScaler {
    
    @Scheduled(fixedRate = 60000)
    public void checkAndScale() {
        long queueSize = getQueueSize();
        int currentWorkers = getWorkerCount();
        
        // Target: Process queue in 1 hour
        int targetWorkers = (int) (queueSize / 2); // 2 videos/worker/hour
        
        if (targetWorkers > currentWorkers * 1.5) {
            // Scale up
            int toAdd = Math.min(targetWorkers - currentWorkers, 500); // Max 500 at once
            scaleWorkers(currentWorkers + toAdd);
            alertService.sendScaleAlert("Scaling up transcoding workers");
        }
        
        // Also: Prioritize processing
        if (queueSize > 50000) {
            // Temporarily skip lower resolutions
            disableResolution("240p");
            disableResolution("360p");
            alertService.sendAlert("Temporarily disabled low resolutions");
        }
    }
}
```

## 3.5. Simulation (End-to-End User Journeys)

### Journey 1: User Watches Video (Happy Path)

**Step-by-step:**

1. **User Action**: Clicks play on video
2. **Client**: Requests manifest file (HLS/DASH)
3. **CDN**: Serves manifest from edge cache (< 10ms)
4. **Client**: Requests first video segment
5. **CDN**: Serves segment from edge cache (99% hit rate)
6. **Playback**: Video plays smoothly, client requests next segments
7. **Adaptive Streaming**: Client adjusts quality based on bandwidth

### Journey 2: Video Upload and Processing

**Step-by-step:**

1. **User Action**: Uploads 500MB video file
2. **Upload Service**: 
   - Chunks file into 10MB chunks
   - Uploads to S3 (multipart upload)
   - Stores metadata in database
3. **Transcoding Service**: 
   - Consumes upload event from Kafka
   - Generates multiple resolutions (240p, 480p, 720p, 1080p)
   - Stores encoded segments in S3
4. **CDN**: 
   - Cache warm-up for popular videos
   - Video available for streaming within 30 minutes

### High-Load / Contention Scenario: Viral Video Launch

```
Scenario: Popular video released, millions watch simultaneously
Time: Video launch event (peak traffic)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Popular video released to 10M subscribers            │
│ - Video ID: video_abc123                                   │
│ - 5M users start watching simultaneously                   │
│ - Expected: ~10K concurrent viewers normal traffic        │
│ - 500x traffic spike                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-10min: CDN Request Storm                              │
│ - 5M concurrent viewers × 5 segments/minute = 25M req/min │
│ - CDN edge locations: 200+ globally                       │
│ - Each location: ~125K requests/minute                    │
│ - Request rate: ~2K requests/second per location          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. CDN Caching:                                           │
│    - Pre-warmed: Video cached at all edge locations       │
│    - Cache hit rate: 99.9% (served from edge)            │
│    - Origin requests: 0.1% (25K requests/minute)          │
│    - Edge latency: < 50ms (excellent)                     │
│                                                              │
│ 2. Origin Load (for cache misses):                       │
│    - S3: Handles 25K requests/minute (well within limits)│
│    - S3 can handle 3,500 requests/second (210K/min)      │
│    - Utilization: 12% (safe)                             │
│                                                              │
│ 3. Adaptive Streaming:                                    │
│    - Clients automatically adjust quality                 │
│    - High bandwidth: 1080p (larger segments)             │
│    - Low bandwidth: 480p (smaller segments)              │
│    - Prevents overwhelming user connections               │
│                                                              │
│ 4. Bandwidth Distribution:                                │
│    - Total bandwidth: 5M viewers × 5 Mbps = 25 Tbps      │
│    - CDN edge network: Handles 100+ Tbps (safe)          │
│    - No congestion or throttling                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 5M viewers stream successfully                      │
│ - 99.9% requests served from CDN (< 50ms latency)       │
│ - P99 latency: 80ms (excellent)                          │
│ - Video quality adapts to user bandwidth                 │
│ - No service degradation                                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Video Segment Corruption During Playback

```
Scenario: Video segment becomes corrupted while user is watching
Edge Case: User experiences playback error mid-stream

┌─────────────────────────────────────────────────────────────┐
│ T+0s: User watching video, segment 15 plays correctly     │
│ - Segment 15: Downloaded and decoded successfully         │
│ - Playback: Smooth, no errors                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+10s: User requests segment 16                          │
│ - Request: GET /videos/video_abc123/720p/segment_16.ts  │
│ - CDN: Serves segment from cache                          │
│ - Client: Downloads segment                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+11s: Segment Corruption Detected                       │
│ - Client: Attempts to decode segment 16                  │
│ - Error: Decode failure (corrupted data)                 │
│ - Playback: Stalls, error shown to user                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Corruption Handling                           │
│                                                              │
│ Problem: Segment 16 corrupted in CDN cache               │
│ - Corrupted segment served to all viewers                │
│ - Multiple users experience playback errors              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Multi-Layer Corruption Handling                │
│                                                              │
│ 1. Client-Side Retry:                                    │
│    - Client detects decode error                         │
│    - Retry: Request segment again from CDN              │
│    - If still corrupted: Request from origin (S3)        │
│    - If origin also corrupted: Request lower quality     │
│                                                              │
│ 2. Server-Side Detection:                                │
│    - Monitor decode error rates per segment             │
│    - If error rate > 1%: Flag segment for verification  │
│    - Background job: Verify segment integrity            │
│    - If corrupted: Re-encode and replace in CDN         │
│                                                              │
│ 3. Quality Fallback:                                     │
│    - If 720p corrupted: Try 480p segment                │
│    - If 480p corrupted: Try 240p segment                │
│    - Ensures playback continues even with corruption     │
│                                                              │
│ 4. CDN Cache Invalidation:                               │
│    - If segment verified corrupted: Invalidate CDN cache│
│    - Force CDN to fetch from origin                     │
│    - Origin serves correct segment (from backup)         │
│    - CDN re-caches correct version                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - User playback continues (quality fallback)            │
│ - Corrupted segments automatically replaced              │
│ - Multiple users don't all experience error              │
│ - System self-heals from corruption                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Video Durability

### Durability Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    VIDEO DURABILITY                              │
└─────────────────────────────────────────────────────────────────┘

1. Upload Phase:
   - Chunked upload with checksum per chunk
   - Retry failed chunks automatically
   - Don't acknowledge until all chunks verified

2. Storage Phase:
   - S3 with 99.999999999% durability
   - Cross-region replication (3 regions)
   - Versioning enabled (accidental delete protection)

3. Processing Phase:
   - Original preserved until all transcodes complete
   - Transcoded files verified with checksum
   - Original kept for 30 days after processing

4. Serving Phase:
   - CDN serves copies, origin is source of truth
   - Regular integrity checks
   - Automated re-transcoding if corruption detected
```

### Integrity Verification

```java
@Service
public class VideoIntegrityService {
    
    @Scheduled(cron = "0 0 3 * * *") // Daily at 3 AM
    public void verifyVideoIntegrity() {
        List<String> videos = videoRepository.getRandomSample(1000);
        
        for (String videoId : videos) {
            // Verify original exists
            if (!s3Client.doesObjectExist("originals/" + videoId)) {
                alertService.sendCriticalAlert("Original missing: " + videoId);
                continue;
            }
            
            // Verify all resolutions exist
            for (String resolution : RESOLUTIONS) {
                String key = "encoded/" + videoId + "/" + resolution + "/playlist.m3u8";
                if (!s3Client.doesObjectExist(key)) {
                    // Re-trigger transcoding
                    transcodingService.retranscode(videoId, resolution);
                }
            }
        }
    }
}
```

---

## 5. Cost Optimization

**Current Monthly Cost:** $2,000,000
**Target Monthly Cost:** $1,400,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **CDN Bandwidth:** $800,000/month (40%) - Largest single component
2. **Storage (S3):** $600,000/month (30%) - Large video storage
3. **Transcoding (GPU):** $300,000/month (15%) - Video encoding costs

**Optimization Strategies (Ranked by Impact):**

1. **Codec Efficiency (40% savings on CDN bandwidth):**
   - **Current:** H.264 baseline (100% bandwidth)
   - **Optimization:** 
     - Use H.265/HEVC for mobile (50% bandwidth)
     - Use VP9 for web (50% bandwidth)
     - Use AV1 for future (35% bandwidth)
   - **Savings:** $320,000/month (40% of $800K CDN)
   - **Trade-off:** Slight CPU overhead for encoding/decoding (acceptable)

2. **Tiered Storage (50% savings on old content):**
   - **Current:** All videos in S3 Standard
   - **Optimization:** 
     - Move videos not watched > 90 days to S3 Glacier (80% cheaper)
     - Move videos not watched > 30 days to S3 Infrequent Access (50% cheaper)
   - **Savings:** $180,000/month (30% of $600K storage)
   - **Trade-off:** Slower access to old videos (acceptable for archive)

3. **Spot Instances for Transcoding (70% savings):**
   - **Current:** On-demand instances for transcoding jobs
   - **Optimization:** Use Spot instances with checkpointing
   - **Savings:** $210,000/month (70% of $300K transcoding)
   - **Trade-off:** Possible interruptions (acceptable, transcoding is batch job)

4. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for stable workloads
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $80,000/month (40% of $200K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

5. **CDN Optimization (20% savings):**
   - **Current:** High CDN bandwidth costs
   - **Optimization:** 
     - Origin shield (reduce origin requests by 50%)
     - Aggressive caching (increase cache hit rate to 95%)
   - **Savings:** $160,000/month (20% of $800K CDN)
   - **Trade-off:** Slight complexity increase (acceptable)

6. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $20,000/month (10% of $200K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $970,000/month (49% reduction)
**Optimized Monthly Cost:** $1,030,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Video Playback (per GB) | $0.08 | $0.041 | 49% |
| Video Storage (per GB/month) | $0.023 | $0.011 | 52% |
| Video Transcoding (per hour) | $10 | $3 | 70% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Codec Efficiency, Tiered Storage → $500K savings
2. **Phase 2 (Month 2):** Spot instances, Reserved Instances → $290K savings
3. **Phase 3 (Month 3):** CDN Optimization, Right-sizing → $180K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor video playback quality (ensure no degradation)
- Monitor storage costs (target < $420K/month)
- Review and adjust quarterly

### Infrastructure Cost Breakdown

| Component        | Cost Driver                | Optimization Strategy                |
| ---------------- | -------------------------- | ------------------------------------ |
| CDN bandwidth    | GB transferred             | Codec efficiency (H.265/VP9)         |
| Storage          | GB stored                  | Tiered storage, TTL for old content  |
| Transcoding      | GPU hours                  | Parallel processing, spot instances  |
| Origin servers   | Requests                   | Origin shield, aggressive caching    |

### Codec Efficiency

```
H.264 baseline: 100% bandwidth
H.264 high:     80% bandwidth
H.265/HEVC:     50% bandwidth
VP9:            50% bandwidth
AV1:            35% bandwidth

Trade-off: Newer codecs need more CPU for encoding/decoding
Strategy: Use H.265 for mobile, H.264 for older devices
```

### Tiered Storage

```java
@Service
public class StorageTieringService {
    
    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void tierOldContent() {
        // Move old, unpopular videos to cheaper storage
        List<String> coldVideos = videoRepository.getVideosNotWatched(
            Duration.ofDays(90)
        );
        
        for (String videoId : coldVideos) {
            // Move from S3 Standard to S3 Glacier
            s3Client.copyObject(
                "videos-hot", videoId,
                "videos-cold", videoId,
                StorageClass.GLACIER
            );
            
            // Delete from hot storage
            s3Client.deleteObject("videos-hot", videoId);
            
            // Update metadata
            videoRepository.setStorageTier(videoId, "cold");
        }
    }
    
    // On-demand retrieval from cold storage
    public void warmUpVideo(String videoId) {
        if (videoRepository.getStorageTier(videoId).equals("cold")) {
            // Initiate restore from Glacier (takes 3-5 hours)
            s3Client.restoreObject("videos-cold", videoId, 
                RestoreTier.STANDARD, Duration.ofDays(7));
            
            // Notify user
            notificationService.sendProcessingNotification(videoId, 
                "Video will be available in 3-5 hours");
        }
    }
}
```

### Cost Calculation Example

```
At YouTube scale:
- 500M daily active users
- 1B video views/day
- Average view: 5 minutes at 720p (3 Mbps)

CDN bandwidth: 
- 1B views × 5 min × 3 Mbps = 1.875 PB/day
- At $0.02/GB = $37.5M/day (!)

Optimizations:
- H.265 codec: 50% reduction → $18.75M/day
- 85% cache hit rate → already factored in
- Regional pricing negotiation: 30% reduction → $13M/day

Revenue:
- 1B views × $0.05 CPM = $50M/day
- Net positive after CDN costs
```

---

## 6. Capacity Planning

### Calculation Example

**Assumptions:**
- 500K video uploads/day
- Average video: 10 minutes, 500MB original
- 7 output resolutions

**Storage Growth:**
```
Daily uploads: 500K × 500MB = 250TB originals
Transcoded (all resolutions): 250TB × 4 = 1PB/day
With replication (3x): 3PB/day
Monthly: 90PB
Yearly: 1EB (before cleanup)
```

**Transcoding Capacity:**
```
Videos/day: 500K
Processing time: 30 min average
Worker capacity: 2 videos/hour = 48 videos/day
Workers needed: 500K / 48 = 10,417 workers
With headroom (50%): 15,625 workers
```

**CDN Capacity:**
```
Peak concurrent viewers: 50M
Average bitrate: 4 Mbps
Peak bandwidth: 50M × 4 Mbps = 200 Tbps
CDN capacity needed: 200 Tbps + 50% headroom = 300 Tbps
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
            .s3(checkS3())
            .postgres(checkPostgres())
            .redis(checkRedis())
            .kafka(checkKafka())
            .cdn(checkCdn())
            .build();
        
        if (status.isHealthy()) {
            return ResponseEntity.ok(status);
        }
        return ResponseEntity.status(503).body(status);
    }
    
    private ComponentHealth checkS3() {
        try {
            s3Client.headBucket("video-uploads");
            return ComponentHealth.healthy();
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
    
    private ComponentHealth checkCdn() {
        try {
            // Check CDN health endpoint
            HttpResponse response = httpClient.send(
                HttpRequest.newBuilder()
                    .uri(URI.create("https://cdn.example.com/health"))
                    .timeout(Duration.ofSeconds(5))
                    .build(),
                HttpResponse.BodyHandlers.ofString()
            );
            return response.statusCode() == 200 ? 
                ComponentHealth.healthy() : 
                ComponentHealth.unhealthy("CDN returned " + response.statusCode());
        } catch (Exception e) {
            return ComponentHealth.unhealthy(e.getMessage());
        }
    }
}
```

---

## Summary

| Aspect              | Strategy                                              |
| ------------------- | ----------------------------------------------------- |
| Monitoring          | Prometheus + Grafana, video-specific metrics          |
| Scaling             | CDN auto-scale, viral detection, priority queues      |
| Failure handling    | Multi-region, origin failover, auto re-transcoding    |
| Durability          | Cross-region replication, integrity checks            |
| Cost optimization   | Codec efficiency, tiered storage, regional pricing    |
| Capacity planning   | 15K transcoding workers, 300 Tbps CDN capacity        |

