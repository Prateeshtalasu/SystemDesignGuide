# News Feed - Production Deep Dives (Ops)

## Overview

This document covers operational aspects of the News Feed system: scaling & reliability, monitoring & observability, security considerations, end-to-end simulations, and cost analysis.

---

## 1. Scaling & Reliability

### Scaling Strategy

#### Current State (500M DAU)
- 1M peak QPS for feed reads
- 150 feed servers
- 66 Redis nodes (22 shards × 3 replicas)
- 50 fan-out workers
- 20 Kafka brokers

#### Scaling to 2B DAU (4x)

| Component        | Current (500M)  | Scaled (2B)    | Notes                          |
| ---------------- | --------------- | -------------- | ------------------------------ |
| Feed Servers     | 150             | 600            | Linear with users              |
| Redis Nodes      | 66              | 250            | More shards for cache          |
| Kafka Brokers    | 20              | 80             | More partitions                |
| Fan-out Workers  | 50              | 200            | Linear with post volume        |
| PostgreSQL       | 3 (1 primary)   | 12 (4 primary) | Sharded by user_id             |
| Cassandra        | 15              | 60             | Linear with feed storage       |

#### Regional Deployment

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              GLOBAL ARCHITECTURE                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

                         ┌───────────────────────┐
                         │   Global DNS          │
                         │   (GeoDNS routing)    │
                         └───────────┬───────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│    US-EAST      │         │    EU-WEST      │         │     APAC        │
│                 │         │                 │         │                 │
│ - Feed Service  │         │ - Feed Service  │         │ - Feed Service  │
│ - Post Service  │         │ - Post Service  │         │ - Post Service  │
│ - Redis Cluster │         │ - Redis Cluster │         │ - Redis Cluster │
│ - Kafka Cluster │         │ - Kafka Cluster │         │ - Kafka Cluster │
│ - PostgreSQL    │         │ - PostgreSQL    │         │ - PostgreSQL    │
│                 │         │                 │         │                 │
│ Users: 150M     │         │ Users: 100M     │         │ Users: 250M     │
└────────┬────────┘         └────────┬────────┘         └────────┬────────┘
         │                           │                           │
         └───────────────────────────┴───────────────────────────┘
                                     │
                         ┌───────────────────────┐
                         │   Cross-Region Sync   │
                         │   (Celebrity posts,   │
                         │    global users)      │
                         └───────────────────────┘
```

### Reliability Patterns

#### Circuit Breaker for Ranking Service

```java
@Service
public class FeedServiceWithCircuitBreaker {
    
    private final CircuitBreaker rankingCircuitBreaker;
    
    public FeedServiceWithCircuitBreaker() {
        this.rankingCircuitBreaker = CircuitBreaker.ofDefaults("ranking");
        
        rankingCircuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.warn("Ranking circuit breaker state: {}", event.getStateTransition()));
    }
    
    public List<FeedEntry> getFeed(Long userId) {
        List<FeedEntry> feed = feedCache.getFeed(userId);
        
        try {
            // Try to rank with circuit breaker
            return rankingCircuitBreaker.executeSupplier(() -> 
                rankingService.rank(feed, userId)
            );
        } catch (Exception e) {
            log.warn("Ranking service unavailable, falling back to chronological");
            metrics.recordRankingFallback();
            
            // Fallback: Sort by timestamp
            return feed.stream()
                .sorted(Comparator.comparing(FeedEntry::getCreatedAt).reversed())
                .collect(Collectors.toList());
        }
    }
}
```

#### Rate Limiting for Feed Refreshes

```java
@Service
public class FeedRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    // Max 1 refresh per user per 5 seconds
    private static final int REFRESH_INTERVAL_SECONDS = 5;
    
    public boolean allowRefresh(Long userId) {
        String key = "feed_refresh:" + userId;
        
        // Try to set with NX (only if not exists)
        Boolean set = redis.opsForValue().setIfAbsent(
            key, 
            "1", 
            Duration.ofSeconds(REFRESH_INTERVAL_SECONDS)
        );
        
        return Boolean.TRUE.equals(set);
    }
    
    public FeedResponse getFeedWithRateLimit(Long userId, boolean forceRefresh) {
        if (forceRefresh && !allowRefresh(userId)) {
            // Return cached feed
            return FeedResponse.builder()
                .posts(feedCache.getFeed(userId))
                .fromCache(true)
                .build();
        }
        
        // Generate fresh feed
        return generateFreshFeed(userId);
    }
}
```

#### Handling Celebrity "Thundering Herd"

```java
@Service
public class CelebrityPostService {
    
    private final LoadingCache<Long, List<Post>> celebrityPostCache;
    private final Map<Long, CompletableFuture<List<Post>>> inFlightRequests = 
        new ConcurrentHashMap<>();
    
    public CelebrityPostService() {
        this.celebrityPostCache = Caffeine.newBuilder()
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .maximumSize(10_000)
            .build(this::loadCelebrityPosts);
    }
    
    public List<Post> getCelebrityPosts(Long celebrityId) {
        // Request coalescing: only one request to backend
        return inFlightRequests.computeIfAbsent(celebrityId, id -> 
            CompletableFuture.supplyAsync(() -> {
                try {
                    return celebrityPostCache.get(id);
                } finally {
                    inFlightRequests.remove(id);
                }
            })
        ).join();
    }
    
    private List<Post> loadCelebrityPosts(Long celebrityId) {
        return postRepository.findRecentByCelebrity(
            celebrityId,
            Instant.now().minus(24, ChronoUnit.HOURS),
            50
        );
    }
}
```

### Database Connection Pool Configuration

**PostgreSQL Connection Pool (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 30          # Max connections per feed service instance
      minimum-idle: 10                # Minimum idle connections
      
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
For feed services:
- 8 CPU cores per pod
- High read/write ratio (10:1)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for high QPS: 30 connections per pod
- 150 pods × 30 = 4,500 max connections
- Use PgBouncer to multiplex: 200 connections per PostgreSQL shard
```

**Cassandra Connection Pool (Datastax Driver):**

```yaml
datastax-java-driver:
  advanced:
    connection:
      pool:
        local:
          max-requests-per-connection: 32768
          max-connections: 10          # Per node (higher for feed reads)
        remote:
          max-requests-per-connection: 32768
          max-connections: 3           # Remote nodes
    
    # Timeouts
    request:
      timeout: 2s                      # Aggressive timeout for feed reads
      read-timeout: 2s
      write-timeout: 5s                # Slightly higher for fan-out writes
```

**Pool Exhaustion Handling:**

1. **Monitoring:**
   - Alert when PostgreSQL pool usage > 75% (earlier warning for high-traffic system)
   - Track Cassandra connection pool metrics per node
   - Monitor wait time for connections

2. **Circuit Breaker:**
   - If pool exhausted for > 3 seconds, open circuit breaker
   - Fallback to cache-only mode (degraded service)

3. **Connection Pooler (PgBouncer):**
   - Place PgBouncer between feed services and PostgreSQL
   - PgBouncer pool: 200 connections per PostgreSQL shard
   - Feed services can share PgBouncer connections efficiently

### Backup and Recovery

**RPO (Recovery Point Objective):** < 15 minutes
- Maximum acceptable data loss
- Based on database replication lag (15 minutes max)
- Post data replicated to read replicas within 15 minutes
- Feed data in Cassandra with eventual consistency (15 minutes)

**RTO (Recovery Time Objective):** < 10 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 15 minutes
- [ ] Cassandra replication lag < 15 minutes
- [ ] Redis snapshots verified in S3
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture post count and feed generation rate
   - Document active user count
   - Verify cache hit rates
   - Test feed generation (100 sample users)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+10 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-4 min:** Promote database read replica to primary
   - **T+4-5 min:** Update DNS records (Route53 health checks)
   - **T+5-8 min:** Warm up Redis cache from database and Cassandra
   - **T+8-9 min:** Verify all services healthy in DR region
   - **T+9-10 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+10 to T+25 minutes):**
   - Verify RTO < 10 minutes: ✅ PASS/FAIL
   - Verify RPO < 15 minutes: Check replication lag at failure time
   - Test feed generation: Generate feeds for 1,000 test users, verify correctness
   - Test post retrieval: Retrieve 100 random posts, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify feed freshness: Check feeds contain recent posts (within 15 min window)

5. **Data Integrity Verification:**
   - Compare post count: Pre-failover vs post-failover (should match within 15 min)
   - Spot check: Verify 100 random posts accessible
   - Check feed consistency: Verify user feeds match expected content
   - Test edge cases: Empty feeds, celebrity feeds, private posts

6. **Failback Procedure (T+25 to T+35 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 15 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 10 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 10 minutes: Time from failure to service resumption
- ✅ RPO < 15 minutes: Maximum data loss (verified via replication lag)
- ✅ Feed generation works: >95% feeds generated correctly
- ✅ No data loss: Post count matches pre-failover (within 15 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Feed freshness maintained: Feeds contain recent posts

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Creates Post and It Appears in Followers' Feeds

**Step-by-step:**

1. **User Action**: User creates post via `POST /v1/posts {"content": "Hello world!", "user_id": "user123"}`
2. **Post Service**: 
   - Validates content
   - Generates post ID: `post_abc123`
   - Stores post in PostgreSQL: `INSERT INTO posts (id, user_id, content, created_at)`
   - Publishes to Kafka: `{"event": "post_created", "post_id": "post_abc123", "user_id": "user123"}`
3. **Response**: `201 Created` with `{"post_id": "post_abc123"}`
4. **Fan-out Worker** (consumes Kafka):
   - Fetches user's followers: `SELECT follower_id FROM follows WHERE user_id = 'user123'` → 200 followers
   - For each follower (regular user, < 10K followers):
     - Writes to Cassandra: `INSERT INTO feed (user_id, post_id, timestamp) VALUES (follower_id, post_abc123, now)`
     - Updates Redis: `ZADD feed:follower_id timestamp post_abc123`
   - Total: 200 writes to Cassandra, 200 Redis updates
5. **Follower 1 opens app**:
   - Request: `GET /v1/feed?user_id=follower1`
   - Feed Service: Checks Redis `ZRANGE feed:follower1 0 20` → HIT (post_abc123 in top 20)
   - Response: `200 OK` with feed including post_abc123
6. **Result**: Post appears in follower's feed within 2 seconds

**Total latency: ~2 seconds** (Kafka + fan-out + Redis update)

### Journey 2: Celebrity Post (Read-Time Fan-out)

**Step-by-step:**

1. **User Action**: Celebrity (1M followers) creates post via `POST /v1/posts`
2. **Post Service**: 
   - Stores post in PostgreSQL
   - Publishes to Kafka
   - **Skips fan-out** (celebrity has > 10K followers)
3. **Celebrity post stored** in Redis: `ZADD celebrity_posts:user123 timestamp post_xyz789`
4. **Follower opens feed**:
   - Request: `GET /v1/feed?user_id=follower1`
   - Feed Service:
     - Fetches regular feed from Redis (fan-out posts)
     - Checks if following celebrities: `SISMEMBER following_celebrities:follower1 user123` → true
     - Fetches celebrity posts: `ZRANGE celebrity_posts:user123 0 10`
     - Merges feeds (by timestamp)
   - Response: `200 OK` with merged feed
5. **Result**: Celebrity post appears in feed on read (no pre-computation)

**Total latency: ~50ms** (Redis lookups + merge)

### Failure & Recovery Walkthrough

**Scenario: Kafka Broker Failure During Fan-out**

**RTO (Recovery Time Objective):** < 5 minutes (broker failover + consumer rebalance)  
**RPO (Recovery Point Objective):** < 1 minute (Kafka replication lag)

**Timeline:**

```
T+0s:    Kafka broker 3 crashes (handles partitions 10-14)
T+0-10s: Fan-out workers consuming from partitions 10-14 fail
T+10s:   Kafka cluster detects broker failure
T+15s:   Partitions 10-14 reassigned to brokers 1, 2, 4, 5
T+20s:   Consumer group rebalances
T+30s:   Workers resume consumption from new partition assignments
T+1min:  All queued posts processed
T+2min:  Fan-out catches up to real-time
```

**What degrades:**
- Fan-out for posts in partitions 10-14 delayed by 1-2 minutes
- Feeds may show posts out of order temporarily
- No data loss (Kafka replication)

**What stays up:**
- Post creation (writes to Kafka succeed)
- Feed reads (existing feeds still work)
- Fan-out for other partitions (unaffected)

**What recovers automatically:**
- Kafka cluster reassigns partitions
- Consumers rebalance and resume
- Fan-out catches up automatically
- No manual intervention required

**What requires human intervention:**
- Investigate broker crash root cause
- Replace failed broker hardware
- Review Kafka cluster capacity

**Cascading failure prevention:**
- Partition replication (3x) prevents data loss
- Consumer group rebalancing distributes load
- Circuit breakers prevent retry storms
- Dead letter queue for failed fan-outs

**Backup Strategy:**
- **Database Snapshots**: Hourly full snapshots, 15-minute incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to secondary region
- **Cassandra Backups**: Daily full backups, hourly incremental
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Feed        │  │ Feed        │  │ Feed        │           │  │
│  │  │ Service     │  │ Service     │  │ Service     │           │  │
│  │  │ (150 pods)  │  │ (150 pods)  │  │ (150 pods)  │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  PostgreSQL (posts, users) - Sharded            │          │  │
│  │  │  ──async repl──> PostgreSQL Replica (DR)         │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Cassandra (user feeds) - 15 nodes                       │ │  │
│  │  │  ──async repl──> Cassandra Replica (DR)                   │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (feed cache) - 66 nodes                  │ │  │
│  │  │  ──async repl──> Redis Replica (DR)                      │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Kafka (fan-out events) - 20 brokers                     │ │  │
│  │  │  ──async repl──> Kafka Mirror (DR)                       │ │  │
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
│  │  PostgreSQL Replica (async replication from us-east-1)            │ │
│  │  Cassandra Replica (async replication from us-east-1)            │ │
│  │  Redis Replica (async replication + snapshots)                   │ │
│  │  Kafka Mirror (async replication from us-east-1)                  │ │
│  │  Feed Service (50 pods, minimal for DR readiness)                │ │
│  │  Fan-out Workers (10 pods, standby)                               │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 15 minutes for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **Cassandra Replication:**
   - **Cassandra Replication:** Async cross-datacenter replication
   - **Replication Lag:** < 15 minutes (acceptable for feed data)
   - **Consistency:** Eventual consistency across regions

3. **Redis Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Snapshots:** Hourly snapshots to S3 for faster recovery
   - **Cache Warm-up:** On failover, restore from latest snapshot

4. **Kafka Replication:**
   - **Kafka MirrorMaker:** Async replication of fan-out events
   - **Replication Lag:** < 15 minutes (acceptable for fan-out jobs)
   - **Partition Rebalancing:** Automatic on failover

5. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Replication Lag** | Stale feeds in DR region | Acceptable for DR (RPO < 15 min), use sync replica for reads in primary |
| **Cassandra Consistency** | Eventual consistency across regions | Acceptable for feeds (slight staleness), strong consistency within region |
| **Fan-out Lag** | Fan-out events may be delayed | Acceptable for DR, events will process eventually |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Feed Freshness** | Feeds may be slightly stale | Acceptable for DR (15 min staleness), users see recent posts |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle traffic)
   - Recovery: Automatic (Kubernetes restarts pod)
   - RTO: < 2 minutes (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 10 minutes (manual process)
   - RPO: < 15 minutes (async replication lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale data
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $625,500/month (full infrastructure)
- **DR Region:** $200,000/month (reduced capacity, 32% of primary)
- **Replication Bandwidth:** $15,000/month (cross-region data transfer)
- **Total Multi-Region Cost:** $840,500/month (34% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** Multiple replication layers provide backup

**Trade-offs:**
- **Cost:** 34% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)
- **Consistency:** Eventual consistency (acceptable for feeds, slight staleness)

---

## 2. Monitoring & Observability

### Key Metrics

```java
@Component
public class FeedMetrics {
    
    private final MeterRegistry registry;
    
    // Latency
    private final Timer feedLoadLatency;
    private final Timer fanoutLatency;
    private final Timer rankingLatency;
    
    // Throughput
    private final Counter feedRequests;
    private final Counter postsCreated;
    private final Counter fanoutWrites;
    
    // Cache
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    // Business metrics
    private final DistributionSummary feedSize;
    private final Counter rankingFallbacks;
    
    public FeedMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.feedLoadLatency = Timer.builder("feed.load.latency")
            .description("Feed load latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.fanoutLatency = Timer.builder("fanout.latency")
            .description("Fan-out processing latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.cacheHits = Counter.builder("feed.cache.hits")
            .description("Feed cache hits")
            .register(registry);
        
        this.cacheMisses = Counter.builder("feed.cache.misses")
            .description("Feed cache misses")
            .register(registry);
        
        this.feedSize = DistributionSummary.builder("feed.size")
            .description("Number of posts in feed response")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public void recordFeedLoad(long durationMs, boolean cacheHit, int postsReturned) {
        feedLoadLatency.record(durationMs, TimeUnit.MILLISECONDS);
        feedRequests.increment();
        feedSize.record(postsReturned);
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
    }
    
    public void recordFanout(long durationMs, int followersProcessed) {
        fanoutLatency.record(durationMs, TimeUnit.MILLISECONDS);
        fanoutWrites.increment(followersProcessed);
    }
}
```

### Alerting Thresholds

| Metric               | Warning    | Critical   | Action                |
| -------------------- | ---------- | ---------- | --------------------- |
| Feed P99 latency     | > 400ms    | > 800ms    | Scale feed servers    |
| Fan-out lag          | > 30s      | > 60s      | Scale fan-out workers |
| Cache hit rate       | < 80%      | < 60%      | Investigate cache     |
| Error rate           | > 0.1%     | > 1%       | Check dependencies    |
| Ranking fallback rate| > 1%       | > 5%       | Check ranking service |
| WebSocket connections| > 80% cap  | > 95% cap  | Scale WS servers      |

### Grafana Dashboard Queries

```promql
# Feed latency P99
histogram_quantile(0.99, 
  sum(rate(feed_load_latency_seconds_bucket[5m])) by (le)
)

# Cache hit rate
sum(rate(feed_cache_hits_total[5m])) / 
(sum(rate(feed_cache_hits_total[5m])) + sum(rate(feed_cache_misses_total[5m])))

# Fan-out throughput
sum(rate(fanout_writes_total[5m]))

# Kafka consumer lag
kafka_consumer_lag{topic="post-events", consumer_group="fanout-workers"}

# Active WebSocket connections
sum(websocket_connections_active)
```

### Health Check Endpoints

```java
@RestController
@RequestMapping("/health")
public class HealthController {
    
    @GetMapping("/ready")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> health = new HashMap<>();
        boolean healthy = true;
        
        // Check Redis
        try {
            redis.ping();
            health.put("redis", "UP");
        } catch (Exception e) {
            health.put("redis", "DOWN");
            healthy = false;
        }
        
        // Check PostgreSQL
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            health.put("postgresql", "UP");
        } catch (Exception e) {
            health.put("postgresql", "DOWN");
            healthy = false;
        }
        
        // Check Kafka
        try {
            kafkaAdmin.describeCluster();
            health.put("kafka", "UP");
        } catch (Exception e) {
            health.put("kafka", "DOWN");
            healthy = false;
        }
        
        return healthy ? 
            ResponseEntity.ok(health) : 
            ResponseEntity.status(503).body(health);
    }
    
    @GetMapping("/live")
    public ResponseEntity<String> liveness() {
        return ResponseEntity.ok("OK");
    }
}
```

---

## 3. Security Considerations

### Authentication & Authorization

```java
@Service
public class FeedSecurityService {
    
    public void validateFeedAccess(Long requesterId, Long targetUserId) {
        if (!requesterId.equals(targetUserId)) {
            // Only allow viewing own feed
            throw new ForbiddenException("Cannot access other user's feed");
        }
    }
    
    public void validatePostVisibility(Long viewerId, Post post) {
        switch (post.getPrivacy()) {
            case "public":
                return; // Anyone can see
                
            case "friends":
                if (!graphService.areFriends(viewerId, post.getAuthorId())) {
                    throw new ForbiddenException("Post is friends-only");
                }
                break;
                
            case "private":
                if (!viewerId.equals(post.getAuthorId())) {
                    throw new ForbiddenException("Post is private");
                }
                break;
        }
    }
}
```

### Content Moderation

```java
@Service
public class ContentModerationService {
    
    private final ToxicityDetector toxicityDetector;
    private final SpamDetector spamDetector;
    
    public ModerationResult moderatePost(Post post) {
        ModerationResult result = new ModerationResult();
        
        // Check for toxic content
        double toxicityScore = toxicityDetector.analyze(post.getContent());
        if (toxicityScore > 0.8) {
            result.setBlocked(true);
            result.setReason("Toxic content detected");
            return result;
        }
        
        // Check for spam
        if (spamDetector.isSpam(post)) {
            result.setBlocked(true);
            result.setReason("Spam detected");
            return result;
        }
        
        // Check rate limiting (too many posts)
        if (isPostingTooFast(post.getAuthorId())) {
            result.setBlocked(true);
            result.setReason("Posting too frequently");
            return result;
        }
        
        result.setBlocked(false);
        return result;
    }
    
    private boolean isPostingTooFast(Long userId) {
        // Max 10 posts per hour
        String key = "post_rate:" + userId;
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, 1, TimeUnit.HOURS);
        }
        return count > 10;
    }
}
```

### Privacy Controls

```java
@Service
public class PrivacyService {
    
    public List<FeedEntry> filterByPrivacy(List<FeedEntry> entries, Long viewerId) {
        return entries.stream()
            .filter(entry -> canView(viewerId, entry))
            .collect(Collectors.toList());
    }
    
    private boolean canView(Long viewerId, FeedEntry entry) {
        // Check if author has blocked viewer
        if (blockService.isBlocked(entry.getAuthorId(), viewerId)) {
            return false;
        }
        
        // Check if viewer has muted author
        if (muteService.isMuted(viewerId, entry.getAuthorId())) {
            return false;
        }
        
        // Check privacy settings
        Post post = postRepository.findById(entry.getPostId());
        return checkPrivacy(viewerId, post);
    }
    
    public void handleUnfollow(Long followerId, Long unfollowedId) {
        // Remove from graph
        graphService.removeFollow(followerId, unfollowedId);
        
        // Remove posts from feed cache
        List<Long> postIds = postRepository.findPostIdsByAuthor(unfollowedId);
        feedCacheService.removePostsFromFeed(followerId, postIds);
        
        // Add to hidden authors (for in-flight posts)
        redis.opsForSet().add("hidden:" + followerId, unfollowedId.toString());
        redis.expire("hidden:" + followerId, 1, TimeUnit.HOURS);
        
        // Notify client to remove posts
        webSocketService.sendRemovePostsMessage(followerId, postIds);
    }
}
```

---

## 4. End-to-End User Journey Simulations

### Simulation 1: Normal Feed Load (Happy Path)

```
User: Alice (500 following, active daily)
Action: Opens app to view feed
Time: 9:00 AM (peak hour)

┌─────────────────────────────────────────────────────────────┐
│ Step 1: App Launch (0ms)                                     │
│ - Client sends GET /v1/feed                                  │
│ - Headers: Authorization: Bearer token_alice                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: API Gateway (5ms)                                    │
│ - Validate JWT token                                         │
│ - Rate limit check: PASS (first request today)              │
│ - Route to feed-service                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Feed Service (15ms)                                  │
│ - Check Redis cache: feed:alice_123                         │
│ - Cache HIT: 450 posts cached                               │
│ - Get top 40 posts (for ranking buffer)                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Celebrity Posts (10ms)                               │
│ - Alice follows 3 celebrities                               │
│ - Fetch recent posts from celebrity_posts table             │
│ - Found: 8 celebrity posts from last 24h                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Ranking Service (20ms)                               │
│ - Merge: 40 + 8 = 48 posts                                  │
│ - Calculate scores for each                                  │
│ - Apply diversity rules                                      │
│ - Return top 20                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Response (Total: 50ms)                               │
│ {                                                            │
│   "posts": [20 ranked posts],                               │
│   "pagination": {"next_cursor": "...", "has_more": true},  │
│   "new_posts_count": 0                                      │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
```

### Simulation 2: Post Creation with Fan-out

```
User: Bob (8,000 followers, regular user)
Action: Creates a new post with image
Time: 12:00 PM

┌─────────────────────────────────────────────────────────────┐
│ Step 1: Post Request (0ms)                                   │
│ POST /v1/posts                                               │
│ {                                                            │
│   "content": {"text": "Lunch time! 🍕", "media_ids": [...]}│
│   "privacy": "public"                                        │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Content Moderation (50ms)                            │
│ - Toxicity check: PASS (score: 0.02)                        │
│ - Spam check: PASS                                           │
│ - Rate limit: PASS (3rd post today)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Post Service (30ms)                                  │
│ - Generate post_id: post_bob_lunch_123                      │
│ - Store in PostgreSQL                                        │
│ - Publish to Kafka: post-events                             │
│ - Return 201 Created to client                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼ (async)
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Fan-out Worker (500ms total)                         │
│ - Consume from Kafka                                         │
│ - Check: 8,000 < 10,000 (not celebrity)                     │
│ - Fetch follower list: 8,000 followers                      │
│ - Process in batches of 1,000:                              │
│   - Batch 1: 50ms                                            │
│   - Batch 2: 50ms                                            │
│   - ...                                                      │
│   - Batch 8: 50ms                                            │
│ - Total Redis writes: 8,000                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Real-time Notifications                              │
│ - 200 followers currently online                            │
│ - Push WebSocket notification to each                       │
│ - "Bob posted: Lunch time! 🍕"                               │
└─────────────────────────────────────────────────────────────┘

Timeline:
- Client receives response: 80ms
- All followers have post in feed: 580ms
- Online followers notified: 600ms
```

### Simulation 3: Failure Scenario - Redis Partial Failure

```
Scenario: 3 of 66 Redis nodes fail during peak traffic

RTO (Recovery Time Objective): < 30 seconds (automatic failover)
RPO (Recovery Point Objective): 0 (no data loss, feed cache may be cold)
Impact: ~5% of users' feed caches unavailable

┌─────────────────────────────────────────────────────────────┐
│ Detection (T+0)                                              │
│ - Redis cluster health check fails                          │
│ - Alert triggered: "Redis nodes down: node-12, 13, 14"     │
│ - Affected shards: 3 of 22                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Automatic Failover (T+30s)                                   │
│ - Redis Sentinel promotes replicas                          │
│ - node-12-replica → node-12 (new primary)                   │
│ - node-13-replica → node-13 (new primary)                   │
│ - node-14-replica → node-14 (new primary)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ During Failover (T+0 to T+30s)                               │
│ Affected users experience:                                   │
│                                                              │
│ Request: GET /v1/feed (user on affected shard)              │
│ 1. Redis cache miss (node unavailable)                      │
│ 2. Fallback to Cassandra for feed history                   │
│ 3. Regenerate and cache in new primary                      │
│                                                              │
│ Latency: 200ms (vs normal 50ms)                             │
│ Success rate: 100% (graceful degradation)                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Recovery (T+30s onwards)                                     │
│ - New primaries accepting traffic                           │
│ - Cache warming for affected users                          │
│ - Latency returns to normal within 5 minutes               │
│ - Ops team investigates root cause                          │
└─────────────────────────────────────────────────────────────┘
```

### Simulation 4: High-Load / Contention Scenario - Celebrity Post (Thundering Herd)

```
User: Taylor (50M followers, celebrity)
Action: Posts during Super Bowl
Time: Halftime (peak engagement)

┌─────────────────────────────────────────────────────────────┐
│ Step 1: Post Creation                                        │
│ - Taylor posts: "Amazing halftime show! 🎤"                 │
│ - Follower count: 50,000,000 > 10,000 (celebrity)          │
│ - NO fan-out triggered                                       │
│ - Post stored in celebrity_posts table only                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Thundering Herd (T+0 to T+60s)                       │
│ - 10M followers refresh feed simultaneously                 │
│ - All request Taylor's celebrity posts                      │
│                                                              │
│ Without protection:                                          │
│ - 10M database queries                                       │
│ - Database overload, cascading failures                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Protection Mechanisms                                │
│                                                              │
│ 1. Celebrity Post Cache (Caffeine, 30s TTL):                │
│    - First request loads from DB                            │
│    - Next 9,999,999 requests hit cache                     │
│                                                              │
│ 2. Request Coalescing:                                       │
│    - Concurrent requests share single DB query              │
│    - Only 1 query per 30-second window                      │
│                                                              │
│ 3. Feed Refresh Rate Limiting:                              │
│    - Max 1 refresh per user per 5 seconds                   │
│    - Spreads load over time                                  │
│                                                              │
│ Result:                                                      │
│ - DB queries: ~2,000 (vs 10M without protection)           │
│ - Cache hit rate: 99.98%                                    │
│ - All users see post within 30 seconds                      │
└─────────────────────────────────────────────────────────────┘
```

### Simulation 5: Edge Case Scenario - Rapid Unfollow/Refollow

```
User: Charlie (1,000 followers)
Action: Unfollows and refollows user_123 rapidly
Time: During peak traffic

┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Race Condition in Fan-out                        │
│                                                              │
│ T+0ms:   Charlie unfollows user_123                        │
│          - Graph Service updates: DELETE follows table     │
│          - Fan-out worker processing post from user_123    │
│          - Checks graph: Charlie NOT in follower list      │
│          - Post NOT added to Charlie's feed                │
│                                                              │
│ T+100ms: Charlie refollows user_123                        │
│          - Graph Service updates: INSERT follows table     │
│          - Post already processed (missed)                  │
│                                                              │
│ Problem: Charlie's feed missing post from user_123         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Eventual Consistency with Catch-up                │
│                                                              │
│ 1. Graph Service publishes follow/unfollow events to Kafka │
│ 2. Fan-out worker listens to graph-events topic            │
│ 3. On refollow event:                                       │
│    - Check if user_123 has recent posts (last 24h)        │
│    - Backfill: Add missed posts to Charlie's feed          │
│                                                              │
│ Result:                                                      │
│ - Feed consistent within 1-2 seconds                       │
│ - No posts permanently missed                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Alternative: Read Repair on Feed Fetch                      │
│                                                              │
│ When Charlie fetches feed:                                  │
│ 1. Check if following user_123                            │
│ 2. If yes, check if recent posts exist in feed            │
│ 3. If missing, fetch and add (read repair)                 │
│                                                              │
│ Trade-off:                                                   │
│ - Adds ~10-20ms to feed fetch latency                      │
│ - Ensures consistency without async processing             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Infrastructure Costs (500M DAU)

| Component         | Specification              | Count  | Monthly Cost |
| ----------------- | -------------------------- | ------ | ------------ |
| Feed Servers      | c5.4xlarge (16 vCPU, 32GB) | 150    | $225,000     |
| Redis Cluster     | r5.4xlarge (16 vCPU, 128GB)| 66     | $165,000     |
| Kafka Brokers     | m5.4xlarge (16 vCPU, 64GB) | 20     | $48,000      |
| Fan-out Workers   | c5.2xlarge (8 vCPU, 16GB)  | 50     | $37,500      |
| PostgreSQL (RDS)  | db.r5.8xlarge              | 3      | $45,000      |
| Cassandra         | i3.2xlarge (8 vCPU, 61GB)  | 15     | $45,000      |
| WebSocket Servers | c5.2xlarge (8 vCPU, 16GB)  | 20     | $15,000      |
| Load Balancers    | ALB                        | 5      | $5,000       |
| Data Transfer     | 500TB/month                | -      | $40,000      |

**Total Monthly: ~$625,500**

### Cost per User

| Metric              | Value                    |
| ------------------- | ------------------------ |
| Monthly cost        | $625,500                 |
| DAU                 | 500,000,000              |
| Cost per DAU/month  | $0.00125                 |
| Cost per DAU/year   | $0.015                   |

### Cost Optimization Strategies

**Current Monthly Cost:** $625,500
**Target Monthly Cost:** $400,000 (36% reduction)

**Top 3 Cost Drivers:**
1. **Feed Servers (EC2):** $225,000/month (36%) - Largest single component
2. **Redis Cluster:** $165,000/month (26%) - Large cache infrastructure
3. **Data Transfer:** $40,000/month (6%) - High data transfer volume

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand c5.4xlarge instances (150 × $1,500 = $225K)
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $90,000/month (40% of $225K)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Skip Inactive Users in Fan-out (40% savings on fan-out writes):**
   - **Current:** Fan-out to all followers (including inactive)
   - **Optimization:** 
     - Skip fan-out to users inactive > 30 days
     - Regenerate feed on demand when they return
     - Reduces fan-out writes by 40%
   - **Savings:** $15,000/month (40% of fan-out worker costs)
   - **Trade-off:** Slight delay for returning users (acceptable)

3. **Tiered Caching (30% savings on Redis):**
   - **Current:** All active users have full feed cached
   - **Optimization:** 
     - Hot users (daily active): Full feed in Redis (100M users)
     - Warm users (weekly active): Partial feed in Redis (200M users)
     - Cold users (monthly active): Feed regenerated on demand (200M users)
   - **Savings:** $49,500/month (30% of $165K Redis)
   - **Trade-off:** Slight latency for cold users (acceptable)

4. **Spot Instances for Fan-out Workers (70% savings):**
   - **Current:** On-demand instances for fan-out workers
   - **Optimization:** Use Spot instances (70% savings)
   - **Savings:** $26,250/month (70% of $37.5K fan-out workers)
   - **Trade-off:** Possible interruptions (acceptable, fan-out is idempotent)

5. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
     - Use auto-scaling more aggressively
   - **Savings:** $22,500/month (10% of $225K feed servers)
   - **Trade-off:** Need careful monitoring to avoid performance issues

6. **Data Transfer Optimization:**
   - **Current:** High cross-region data transfer costs
   - **Optimization:** 
     - Compress feed data before transfer
     - Use regional endpoints to reduce cross-region transfer
   - **Savings:** $12,000/month (30% of $40K data transfer)
   - **Trade-off:** Slight CPU overhead (negligible)

**Total Potential Savings:** $215,250/month (34% reduction)
**Optimized Monthly Cost:** $410,250/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Feed Request | $0.0000021 | $0.0000014 | 33% |
| Post Creation | $0.00001 | $0.000006 | 40% |
| Fan-out Write | $0.000001 | $0.0000006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Skip inactive users → $105K savings
2. **Phase 2 (Month 2):** Tiered caching, Spot instances → $75.75K savings
3. **Phase 3 (Month 3):** Right-sizing, Data transfer optimization → $34.5K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor feed latency (ensure no degradation)
- Monitor cache hit rates (target >80%)
- Review and adjust quarterly

### Cost Projection at Scale

| DAU        | Monthly Cost | Cost per DAU |
| ---------- | ------------ | ------------ |
| 100M       | $150,000     | $0.0015      |
| 500M       | $625,000     | $0.00125     |
| 1B         | $1,100,000   | $0.0011      |
| 2B         | $2,000,000   | $0.001       |

*Note: Cost per DAU decreases at scale due to infrastructure efficiency gains*

---

## Summary

| Aspect              | Approach                           | Key Metrics                |
| ------------------- | ---------------------------------- | -------------------------- |
| Scaling             | Regional deployment, horizontal    | 4x scale = 4x resources    |
| Reliability         | Circuit breakers, rate limiting    | 99.9% availability target  |
| Monitoring          | Prometheus + Grafana               | P99 < 400ms, cache > 80%   |
| Security            | Content moderation, privacy        | Block toxic content        |
| Cost                | $0.00125/DAU/month                 | Optimize via tiered cache  |

