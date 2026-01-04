# Pastebin - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis. These topics ensure Pastebin operates reliably in production at scale.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Layer 7 (Application) Load Balancing with AWS ALB:**

```
Internet → CloudFront (CDN) → ALB → Paste Service Pods
```

**Why ALB over NLB?**
- Path-based routing (separate `/v1/` API from `/raw/` content)
- Health checks with HTTP endpoints
- SSL/TLS termination
- Request logging and metrics

**Load Balancing Algorithm:** Round-robin with health checks

```yaml
# ALB Target Group Configuration
target_group:
  protocol: HTTP
  port: 8080
  health_check:
    path: /actuator/health
    interval: 30s
    timeout: 5s
    healthy_threshold: 2
    unhealthy_threshold: 3
  stickiness:
    enabled: false  # Stateless service, no need for stickiness
```

### Rate Limiting Implementation

**Algorithm: Sliding Window with Redis**

```java
@Service
public class RateLimiter {
    
    private final StringRedisTemplate redis;
    
    public RateLimitResult checkLimit(String identifier, String action, int limit) {
        String key = String.format("rate:%s:%s", identifier, action);
        String windowKey = key + ":" + (System.currentTimeMillis() / 3600000); // Hour window
        
        Long count = redis.opsForValue().increment(windowKey);
        if (count == 1) {
            redis.expire(windowKey, Duration.ofHours(1));
        }
        
        return RateLimitResult.builder()
            .allowed(count <= limit)
            .current(count.intValue())
            .limit(limit)
            .resetAt(getNextHour())
            .build();
    }
}
```

**Rate Limit Tiers:**

| Tier | Paste Create/hour | Paste View/hour | Storage Limit |
|------|-------------------|-----------------|---------------|
| Anonymous | 10 | 1000 | N/A |
| Free | 100 | 10,000 | 100 MB |
| Pro | 1000 | Unlimited | 10 GB |
| Enterprise | Unlimited | Unlimited | Unlimited |

**Response Headers:**

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String identifier = getIdentifier(request); // IP or API key
        RateLimitResult result = rateLimiter.check(identifier, "create", getLimit(request));
        
        response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(result.getRemaining()));
        response.setHeader("X-RateLimit-Reset", String.valueOf(result.getResetTime()));
        
        if (!result.isAllowed()) {
            response.setStatus(429);
            response.setHeader("Retry-After", String.valueOf(result.getRetryAfter()));
            return false;
        }
        return true;
    }
}
```

### Circuit Breaker Placement

**Services with Circuit Breakers:**

| Service | Circuit Breaker | Fallback Behavior |
|---------|-----------------|-------------------|
| Redis Cache | Yes | Fall back to S3 directly |
| PostgreSQL | Yes | Return 503, no fallback |
| S3 | Yes | Serve from Redis if cached |
| CDN Purge | Yes | Queue for retry |

**Configuration (Resilience4j):**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      redis:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
      postgresql:
        slidingWindowSize: 50
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      s3:
        slidingWindowSize: 100
        failureRateThreshold: 30
        waitDurationInOpenState: 60s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Redis GET | 50ms | 1 | None |
| PostgreSQL Query | 200ms | 2 | 100ms exponential |
| S3 GetObject | 5s | 3 | 500ms exponential |
| S3 PutObject | 30s | 3 | 1s exponential |
| CDN Purge | 5s | 3 | 1s exponential |

**Jitter Implementation:**

```java
public long calculateBackoff(int attempt, long baseMs) {
    long exponentialDelay = baseMs * (1L << attempt);  // 100, 200, 400, ...
    long jitter = ThreadLocalRandom.current().nextLong(0, exponentialDelay / 2);
    return exponentialDelay + jitter;
}
```

### Replication and Failover

**PostgreSQL:**
- 1 Primary + 2 Read Replicas
- Synchronous replication to 1 replica (zero data loss)
- Asynchronous replication to DR replica
- Automatic failover with Patroni (< 30 seconds)

**Redis:**
- 3-node cluster (3 primary shards)
- Each shard has 1 replica
- Automatic failover per shard

**S3:**
- 11 nines durability (99.999999999%)
- Cross-region replication for DR
- No manual failover needed

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: View count updates
   - Impact: Analytics slightly delayed
   - Trigger: Redis unavailable

2. **Second to drop**: CDN cache invalidation
   - Impact: Deleted pastes may be served briefly
   - Trigger: CDN API unavailable

3. **Third to drop**: Rate limiting
   - Impact: Potential abuse, but service stays up
   - Trigger: Redis unavailable

4. **Fourth to drop**: New paste creation
   - Impact: Read-only mode
   - Trigger: Database at capacity

5. **Last resort**: Serve from CDN only
   - Impact: Only cached pastes work
   - Trigger: Origin completely down

### Disaster Recovery

**Multi-AZ Deployment:**
- Primary region: us-east-1 (3 AZs)
- DR region: us-west-2 (cold standby)

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Availability Zone 1a          Availability Zone 1b          │  │
│  │  ┌─────────────┐              ┌─────────────┐                 │  │
│  │  │ ALB         │              │ ALB         │                 │  │
│  │  └──────┬──────┘              └──────┬──────┘                 │  │
│  │         │                            │                        │  │
│  │  ┌──────┴──────┐              ┌──────┴──────┐                 │  │
│  │  │ Paste Svc   │              │ Paste Svc   │                 │  │
│  │  │ (Pods 1-3)  │              │ (Pods 4-5)  │                 │  │
│  │  └──────┬──────┘              └──────┬──────┘                 │  │
│  │         │                            │                        │  │
│  │  ┌──────┴────────────────────────────┴──────┐                 │  │
│  │  │  PostgreSQL Primary (AZ 1a)              │  │
│  │  │  ──sync repl──> PostgreSQL Replica (1b)  │  │
│  │  │  ──async repl──> PostgreSQL Replica (1c)  │  │
│  │  └──────────────────────────────────────────┘                 │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (3 primary + 3 replica across AZs)        │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  S3 Bucket (us-east-1)                                   │ │  │
│  │  │  ──CRR──> S3 Bucket (us-west-2)                         │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async replication──                            │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: us-west-2                               │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  PostgreSQL Replica (async replication from us-east-1)            │ │
│  │  Redis Replica (async replication from us-east-1)                 │ │
│  │  Paste Service (2 pods, minimal for DR readiness)                 │ │
│  │  ALB (standby, routes traffic only during failover)               │ │
│  │  S3 Bucket (CRR destination, read-only until failover)            │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region (AZ 1b) for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 1 minute for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **S3 Cross-Region Replication (CRR):**
   - **Replication:** Async replication of all paste content to DR region
   - **Replication Lag:** Typically < 5 minutes for S3 CRR
   - **Access:** DR region S3 bucket read-only until failover
   - **Cost:** Additional storage cost in DR region

3. **Cache Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Cache Warm-up:** On failover, cache populated from database
   - **Cache Invalidation:** Cross-region invalidation via pub/sub (optional)

4. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **S3 CRR Lag** | Paste content may not be immediately available in DR | Acceptable for DR (RPO < 5 min), critical pastes can be pre-replicated |
| **Replication Lag** | Stale reads in DR region | Acceptable for DR (RPO < 1 min), use sync replica for reads in primary |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost + S3 replication | DR region uses smaller instances, S3 CRR cost ~$50/month |
| **Data Consistency** | Eventual consistency across regions | Acceptable for DR scenario, strong consistency within region |
| **Cache Coherency** | Stale cache in DR region | Cache rebuilt from database on failover, acceptable for DR |

**Failover Scenarios:**

1. **Single AZ Failure:**
   - Impact: Minimal (other AZs handle traffic)
   - Recovery: Automatic (load balancer routes away from failed AZ)
   - RTO: < 1 minute (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 15 minutes (manual process)
   - RPO: < 5 minutes (S3 CRR lag, database < 1 min)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale data
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $15,000/month (full infrastructure)
- **DR Region:** $2,500/month (minimal infrastructure, 17% of primary)
- **S3 CRR:** $50/month (cross-region replication)
- **Replication Bandwidth:** $200/month (cross-region data transfer)
- **Total Multi-Region Cost:** $17,750/month (18% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** S3 CRR provides additional backup layer

**Trade-offs:**
- **Cost:** 18% increase for DR capability
- **Complexity:** More complex operations (failover procedures, S3 CRR monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)

**RTO/RPO Targets:**
- RPO (Recovery Point Objective): < 1 minute (async replication lag)
- RTO (Recovery Time Objective): < 15 minutes (manual failover)

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 minute
- [ ] S3 cross-region replication (CRR) enabled and verified
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture database replication lag
   - Document active paste count
   - Verify S3 CRR status and lag
   - Check cache hit rates

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in us-east-1 (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote database replica to primary in us-west-2
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-8 min:** Verify S3 content accessible (CRR should be complete)
   - **T+8-12 min:** Warm up Redis cache from database
   - **T+12-14 min:** Verify all services healthy in DR region
   - **T+14-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+30 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check replication lag at failure time
   - Test paste creation: Create 10 test pastes, verify persistence
   - Test paste retrieval: Retrieve 100 test pastes, verify 100% success
   - Verify S3 access: Confirm pastes stored in S3 are accessible
   - Monitor metrics: QPS, latency, error rate return to baseline

5. **Data Integrity Verification:**
   - Compare paste count: Pre-failover vs post-failover (should match)
   - Spot check: Verify 100 random pastes accessible (content + metadata)
   - Check S3: Verify all paste content files accessible
   - Check replication: Verify no gaps in database replication log

6. **Failback Procedure (T+30 to T+45 minutes):**
   - Restore primary region services
   - Reconfigure replication (DR → Primary)
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via database replication lag)
- ✅ Paste creation works: >99% pastes created successfully
- ✅ Paste retrieval works: >99% pastes retrieved successfully
- ✅ No data loss: Paste count matches pre-failover (within 1 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Creates and Views a Paste

**Step-by-step:**

1. **User Action**: Developer pastes code snippet into web form
2. **Request**: `POST /v1/pastes` with `{"content": "def hello(): print('world')", "expires_in": 3600}`
3. **API Gateway**: Validates request, checks rate limit
4. **Paste Service**: 
   - Generates unique paste ID: `abc123xyz`
   - Stores content in S3: `s3://pastes/abc/123/xyz/content.txt`
   - Inserts metadata into PostgreSQL: `INSERT INTO pastes (id, s3_key, expires_at)`
   - Caches metadata in Redis: `SET paste:abc123xyz "{metadata}"` with TTL
5. **Response**: `201 Created` with `{"paste_id": "abc123xyz", "url": "https://pastebin.com/abc123xyz"}`
6. **User Action**: Shares URL with colleague
7. **Colleague clicks link**:
   - CDN cache miss (first view)
   - Load balancer routes to Paste Service
   - Redis: `GET paste:abc123xyz` → HIT (metadata)
   - S3: `GET s3://pastes/abc/123/xyz/content.txt` → Returns content
   - Response: `200 OK` with paste content
8. **CDN caches** the paste for 1 hour
9. **Subsequent views**: Served from CDN edge (< 10ms)

**Total latency: 150ms** (S3 read + metadata lookup)

### Journey 2: Paste Expiration and Cleanup

**Step-by-step:**

1. **Background Job**: Runs every hour, scans for expired pastes
2. **Query**: `SELECT id, s3_key FROM pastes WHERE expires_at < NOW() LIMIT 1000`
3. **For each expired paste**:
   - Delete from S3: `DELETE s3://pastes/{key}`
   - Delete metadata from PostgreSQL: `DELETE FROM pastes WHERE id = ?`
   - Invalidate Redis cache: `DEL paste:{id}`
   - Log deletion event to Kafka for analytics
4. **Async Consumer**: Processes deletion events, updates analytics
5. **Result**: Storage freed, cache cleaned, analytics updated

**Total time: ~5 minutes** for 1000 pastes (batch processing)

### Failure & Recovery Walkthrough

**Scenario: S3 Service Degradation**

**RTO (Recovery Time Objective):** < 5 minutes (automatic retry with exponential backoff)  
**RPO (Recovery Point Objective):** 0 (no data loss, writes queued)

**Timeline:**

```
T+0s:    S3 starts returning 503 errors (throttling)
T+0-10s: Paste creation requests fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Writes queued to local buffer (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   S3 recovers, accepts requests
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Queued writes processed from buffer
T+5min:  All queued writes completed
```

**What degrades:**
- Paste creation fails for 10-30 seconds
- Writes queued in buffer (no data loss)
- Read operations unaffected (S3 GET still works)

**What stays up:**
- Paste viewing (reads from S3)
- Metadata lookups (PostgreSQL, Redis)
- CDN serving cached pastes

**What recovers automatically:**
- Circuit breaker closes after S3 recovery
- Queued writes processed automatically
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
- ✅ RPO < 1 minute: Maximum data loss (verified via replication lag)
- ✅ All active pastes remain accessible: 100% retrieval success rate
- ✅ No data loss: Paste count matches pre-failover
- ✅ S3 content accessible: All paste files retrievable from DR region
- ✅ Cache repopulates correctly: Cache hit rate returns to >85% within 10 minutes

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

### What Breaks First Under Load

**Order of failure at 10x traffic (2,200 QPS):**

1. **S3 request rate** (First)
   - Symptom: 503 errors from S3
   - S3 limit: 5,500 GET/sec per prefix
   - Fix: Randomize key prefixes, increase cache hit rate

2. **Database connections** (Second)
   - Symptom: Connection pool exhausted
   - Fix: Add PgBouncer, increase pool size (see connection pool configuration below)

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 20          # Max connections per application instance
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
- With safety margin: 20 connections per pod
- 10 pods × 20 = 200 max connections to database
- Database max_connections: 300 (20% headroom)
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
   - PgBouncer pool: 100 connections
   - App pools can share PgBouncer connections efficiently

3. **Redis memory** (Third)
   - Symptom: Evictions increase, hit rate drops
   - Fix: Add nodes, increase memory

4. **Application CPU** (Fourth)
   - Symptom: Latency increases
   - Fix: Add more pods

---

## 2. Monitoring & Observability

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Paste View P50 | < 30ms | > 50ms | > 100ms |
| Paste View P95 | < 100ms | > 150ms | > 200ms |
| Paste View P99 | < 200ms | > 300ms | > 500ms |
| Paste Create P50 | < 300ms | > 500ms | > 800ms |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Views/sec | 40 | > 100 | > 200 |
| Creates/sec | 4 | > 10 | > 20 |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Error rate (5xx) | < 0.01% | > 0.1% | > 1% |
| 404 rate | < 10% | > 20% | > 30% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 70% | > 90% |
| Memory | > 80% | > 95% |
| DB Connections | > 80% | > 95% |
| Redis Memory | > 70% | > 85% |
| S3 Request Rate | > 70% of limit | > 90% of limit |

### Metrics Collection

```java
@Component
public class PasteMetrics {
    
    private final MeterRegistry registry;
    private final Counter pastesCreated;
    private final Counter pastesViewed;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Timer viewLatency;
    private final Timer createLatency;
    private final DistributionSummary pasteSize;
    
    public PasteMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.pastesCreated = Counter.builder("pastes.created.total")
            .description("Total pastes created")
            .register(registry);
        
        this.pastesViewed = Counter.builder("pastes.viewed.total")
            .description("Total pastes viewed")
            .register(registry);
        
        this.viewLatency = Timer.builder("paste.view.latency")
            .description("Paste view latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.pasteSize = DistributionSummary.builder("paste.size.bytes")
            .description("Paste size distribution")
            .publishPercentiles(0.5, 0.9, 0.99)
            .register(registry);
    }
    
    public void recordCreate(Paste paste) {
        pastesCreated.increment();
        pasteSize.record(paste.getSizeBytes());
    }
    
    public void recordView(String pasteId, long durationMs, boolean cacheHit) {
        pastesViewed.increment();
        viewLatency.record(durationMs, TimeUnit.MILLISECONDS);
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
    }
}
```

### Alerting Thresholds and Escalation

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High Latency P99 | > 300ms for 5 min | Warning | Ticket |
| High Latency P99 | > 500ms for 2 min | Critical | Page on-call |
| Error Rate | > 0.1% for 5 min | Warning | Ticket |
| Error Rate | > 1% for 1 min | Critical | Page on-call |
| Cache Hit Rate | < 70% for 10 min | Warning | Ticket |
| Cache Hit Rate | < 50% for 5 min | Critical | Page on-call |
| S3 Errors | > 0.1% for 5 min | Warning | Ticket |
| S3 Errors | > 1% for 2 min | Critical | Page on-call |
| Cleanup Job Failed | 2 consecutive | Warning | Ticket |
| Cleanup Job Failed | 4 consecutive | Critical | Page on-call |

### Logging Strategy

**Structured Logging Format:**

```java
@Slf4j
@RestController
public class PasteController {
    
    @GetMapping("/v1/pastes/{id}")
    public ResponseEntity<PasteResponse> getPaste(@PathVariable String id, HttpServletRequest request) {
        long startTime = System.currentTimeMillis();
        
        try {
            Paste paste = pasteService.getPaste(id);
            
            log.info("paste.view.success",
                kv("paste_id", id),
                kv("latency_ms", System.currentTimeMillis() - startTime),
                kv("cache_hit", paste.isCacheHit()),
                kv("size_bytes", paste.getSizeBytes()),
                kv("syntax", paste.getSyntax())
            );
            
            return ResponseEntity.ok(PasteResponse.from(paste));
            
        } catch (NotFoundException e) {
            log.warn("paste.view.not_found",
                kv("paste_id", id),
                kv("latency_ms", System.currentTimeMillis() - startTime)
            );
            return ResponseEntity.notFound().build();
        }
    }
}
```

**PII Redaction:**
- IP addresses: Hash before logging
- Paste content: Never log content
- User agents: Log as-is (not PII)
- Titles: Truncate to 50 chars in logs

### Distributed Tracing

**Trace ID Propagation:**

```java
@Component
public class TracingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        String traceId = ((HttpServletRequest) request).getHeader("X-Trace-ID");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        MDC.put("traceId", traceId);
        ((HttpServletResponse) response).setHeader("X-Trace-ID", traceId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("traceId");
        }
    }
}
```

**Span Boundaries:**
1. HTTP Request (root span)
2. Redis Metadata Lookup (child span)
3. Redis Content Lookup (child span)
4. PostgreSQL Query (child span, if cache miss)
5. S3 GetObject (child span, if cache miss)

### Health Checks

```java
@Component
public class PastebinHealthIndicator implements HealthIndicator {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final JdbcTemplate jdbcTemplate;
    private final S3Client s3Client;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean healthy = true;
        
        // Check Redis
        try {
            redisTemplate.opsForValue().get("health-check");
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DOWN: " + e.getMessage());
            healthy = false;
        }
        
        // Check PostgreSQL
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            details.put("postgresql", "UP");
        } catch (Exception e) {
            details.put("postgresql", "DOWN: " + e.getMessage());
            healthy = false;
        }
        
        // Check S3 (non-critical for reads if cached)
        try {
            s3Client.headBucket(HeadBucketRequest.builder()
                .bucket("pastebin-content")
                .build());
            details.put("s3", "UP");
        } catch (Exception e) {
            details.put("s3", "DOWN: " + e.getMessage());
            // S3 down is not critical if cache is working
        }
        
        return healthy ? Health.up().withDetails(details).build()
                       : Health.down().withDetails(details).build();
    }
}
```

**Kubernetes Probes:**

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

### Dashboards

**Golden Signals Dashboard:**
- Request rate (by endpoint: create, view, delete)
- Error rate (by status code)
- Latency percentiles (P50, P95, P99)
- Saturation (CPU, memory, connections)

**Business KPIs Dashboard:**
- Pastes created per hour
- Pastes viewed per hour
- Storage growth rate
- Top 10 pastes by views
- Syntax language distribution

---

## 3. Security Considerations

### Authentication Mechanism

**API Key Authentication:**

```java
@Service
public class ApiKeyService {
    
    public String generateApiKey() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        String key = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
        return "pb_" + key;  // e.g., pb_xK9mN2pQ...
    }
    
    public void storeApiKey(Long userId, String apiKey) {
        String hash = DigestUtils.sha256Hex(apiKey);
        String prefix = apiKey.substring(0, 10);
        
        ApiKeyEntity entity = ApiKeyEntity.builder()
            .userId(userId)
            .keyHash(hash)
            .keyPrefix(prefix)
            .createdAt(Instant.now())
            .build();
        
        apiKeyRepository.save(entity);
    }
    
    public Optional<ApiKeyEntity> validateApiKey(String apiKey) {
        String hash = DigestUtils.sha256Hex(apiKey);
        return apiKeyRepository.findByKeyHash(hash);
    }
}
```

### Authorization and Access Control

**Permission Model:**

| Role | Create Paste | View Paste | Delete Paste | View Private |
|------|--------------|------------|--------------|--------------|
| Anonymous | Yes (public/unlisted) | Public/Unlisted | No | No |
| User | Yes (all types) | Public/Unlisted/Own | Own only | Own only |
| Admin | Yes | All | All | All |

### Data Encryption

**At Rest:**
- PostgreSQL: AWS RDS encryption (AES-256)
- S3: SSE-S3 encryption (AES-256)
- Redis: Not encrypted (in-memory, acceptable risk)

**In Transit:**
- All external traffic: TLS 1.3
- Internal traffic: TLS within VPC

### Input Validation and Sanitization

```java
@Service
public class ContentValidator {
    
    private static final long MAX_SIZE = 10 * 1024 * 1024; // 10 MB
    
    public ValidationResult validate(String content) {
        // 1. Size check
        if (content.getBytes(StandardCharsets.UTF_8).length > MAX_SIZE) {
            return ValidationResult.invalid("Content exceeds 10 MB limit");
        }
        
        // 2. Empty check
        if (content.isBlank()) {
            return ValidationResult.invalid("Content cannot be empty");
        }
        
        // 3. Binary content check
        if (containsBinaryData(content)) {
            return ValidationResult.invalid("Binary content not allowed");
        }
        
        // 4. Malware pattern check (optional)
        if (containsMalwarePatterns(content)) {
            return ValidationResult.flagged("Content flagged for review");
        }
        
        return ValidationResult.valid();
    }
    
    private boolean containsBinaryData(String content) {
        // Check for null bytes or control characters
        return content.chars().anyMatch(c -> c == 0 || (c < 32 && c != 9 && c != 10 && c != 13));
    }
}
```

### XSS Prevention

```java
@Component
public class ContentSanitizer {
    
    public String sanitizeForDisplay(String content) {
        // HTML encode for safe display
        return HtmlUtils.htmlEscape(content);
    }
    
    public boolean containsMaliciousPatterns(String content) {
        // Check for common attack patterns
        List<Pattern> patterns = List.of(
            Pattern.compile("<script[^>]*>", Pattern.CASE_INSENSITIVE),
            Pattern.compile("javascript:", Pattern.CASE_INSENSITIVE),
            Pattern.compile("on\\w+\\s*=", Pattern.CASE_INSENSITIVE)
        );
        
        return patterns.stream()
            .anyMatch(p -> p.matcher(content).find());
    }
}
```

### PII Handling and Compliance

**GDPR Compliance:**

1. **Right to Deletion**: Users can delete their pastes and account
2. **Data Minimization**: Only collect necessary data
3. **IP Hashing**: Store SHA-256 hash of IP in logs, not raw IP
4. **Retention Policy**: Pastes deleted after expiration

**Data Retention:**

| Data Type | Retention | Deletion Method |
|-----------|-----------|-----------------|
| Pastes | Until expiration | Cleanup worker |
| User accounts | Until deletion request | Hard delete |
| Access logs | 90 days | Log rotation |
| Metrics | 1 year | Prometheus retention |

### Security Headers

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy("default-src 'self'; script-src 'self' https://cdnjs.cloudflare.com")
                .xssProtection(xss -> xss.block(true))
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)
                )
                .contentTypeOptions(contentType -> {})
            )
            .requiresChannel(channel -> channel
                .anyRequest().requiresSecure()
            );
        
        return http.build();
    }
}
```

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Enumeration** | Rate limiting, non-sequential IDs, honeypot pastes |
| **Abuse (spam/malware)** | Content scanning, user reports, rate limits |
| **XSS** | Content sanitization, CSP headers, output encoding |
| **Injection** | Parameterized queries, input validation |
| **DDoS** | CDN, rate limiting, auto-scaling |
| **Data exfiltration** | Encryption at rest, access controls, audit logs |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Developer Creates and Shares Code Snippet

**Step-by-step:**

1. **User Action**: Developer pastes Python code into web form
2. **Request**: `POST /v1/pastes` with content, syntax="python", visibility="unlisted"
3. **API Gateway**: Validates request, checks rate limit (anonymous: 10/hour)
4. **Paste Service**:
   - Validates content (size < 10MB, not binary)
   - Generates unique ID: `abc12345`
   - Computes SHA-256 hash for deduplication
   - Uploads compressed content to S3: `pastes/2024/01/15/abc12345.txt.gz`
   - Inserts metadata into PostgreSQL
   - Caches metadata in Redis
5. **Response**: `201 Created` with `{"url": "https://pastebin.com/abc12345"}`
6. **User Action**: Shares URL on Slack
7. **Team member clicks link**:
   - CDN cache miss (first view)
   - Redis metadata hit
   - Redis content miss
   - S3 fetch, cache in Redis
   - Return content with syntax highlighting
8. **Subsequent views**: Served from CDN or Redis cache

### Journey 2: User Views Paste with Password Protection

**Step-by-step:**

1. **User Action**: Clicks `https://pastebin.com/xyz78901`
2. **CDN**: Cache miss (password-protected pastes not cached at CDN)
3. **Paste Service**:
   - Redis: `GET meta:xyz78901` → HIT
   - Metadata shows `password_hash` is set
   - Returns 401 with message "Password required"
4. **User Action**: Enters password in prompt
5. **Request**: `GET /v1/pastes/xyz78901?password=secret123`
6. **Paste Service**:
   - Verifies bcrypt hash matches
   - Fetches content from Redis or S3
   - Returns content
7. **Response**: 200 OK with paste content

### Failure & Recovery Walkthrough

#### Failure Scenario 1: S3 Partial Outage

**RTO (Recovery Time Objective):** < 5 minutes  
**RPO (Recovery Point Objective):** < 1 minute (no data loss, reads may fail temporarily)

**Detection:**
- **How to detect**: Monitor S3 error rate, alert threshold: > 1% for 2 minutes
- **Alert thresholds**: 
  - Warning: S3 error rate > 0.5% for 5 minutes
  - Critical: S3 error rate > 1% for 2 minutes
- **Monitoring**: CloudWatch S3 metrics, application error logs
- **Symptoms**: 
  - 503 errors from S3 API
  - Paste view failures
  - High latency on paste creation

**Impact:**
- **Affected services**: Paste Service (read operations), Cleanup Worker
- **User impact**: Users cannot view pastes, new pastes may fail to upload
- **Degradation**: 
  - Read operations: 100% failure rate
  - Write operations: May succeed if S3 write path unaffected
  - Cache: Existing cached pastes still accessible

**Recovery Steps:**
1. **Verify S3 outage** (30 seconds)
   - Check AWS Service Health Dashboard
   - Verify S3 error rate in CloudWatch
   - Confirm affected S3 buckets/regions

2. **Activate circuit breaker** (10 seconds)
   - Circuit breaker should auto-activate after 50% failure rate
   - Verify circuit breaker state in monitoring
   - If not activated, manually trigger via API

3. **Enable fallback to Redis cache** (20 seconds)
   - Verify Redis cache contains popular pastes
   - Monitor cache hit rate (should increase)
   - Serve cached pastes for read operations

4. **Route new uploads to alternative region** (1 minute)
   - Update S3 client configuration to use DR region bucket
   - Verify write operations succeed
   - Monitor write success rate

5. **Warm up cache from database** (2 minutes)
   - Query database for popular paste metadata
   - Pre-populate Redis cache with metadata
   - Note: Content still unavailable until S3 recovers

6. **Monitor recovery** (ongoing)
   - Watch S3 error rate decrease
   - Verify circuit breaker closes when S3 recovers
   - Gradually resume normal S3 operations

**RTO:** < 5 minutes (circuit breaker + fallback activation)  
**RPO:** < 1 minute (S3 replication lag, no data loss)

**Prevention:**
- Enable S3 cross-region replication for all buckets
- Implement aggressive caching for popular pastes
- Use S3 Transfer Acceleration for better reliability
- Monitor S3 request rate limits and scale accordingly

**Timeline:**

```
T+0s:    S3 starts returning 503 errors for some requests
T+0-30s: Cache hits continue working normally
T+30s:   Cache misses start failing
T+60s:   Circuit breaker opens for S3
T+60s:   Service returns cached content only
T+5min:  S3 recovers
T+5min:  Circuit breaker half-opens
T+6min:  Successful S3 requests, circuit closes
T+10min: Normal operation restored
```

**What degrades:**
- New paste creation fails (can't upload to S3)
- Cache misses return 503
- Only cached pastes accessible

**What stays up:**
- Cached paste views (Redis + CDN)
- Paste metadata queries
- Rate limiting

**What recovers automatically:**
- S3 self-heals
- Circuit breaker closes after success threshold
- Cache repopulates on demand

**What requires human intervention:**
- Investigate S3 issue
- Review failed paste creations
- Potentially retry failed uploads from queue

**Cascading failure prevention:**
- Circuit breaker prevents retry storms to S3
- Timeouts prevent thread exhaustion
- Graceful degradation serves cached content

### High-Load / Contention Scenario: Viral Paste Traffic Spike

```
Scenario: Popular GitHub repo links to paste, causing 100K views in 1 hour
Time: Peak traffic event

┌─────────────────────────────────────────────────────────────┐
│ T+0s: GitHub issue links paste URL "def45678"              │
│ - Paste was created yesterday (warm cache)                  │
│ - Expected: ~100 views/day normal traffic                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-5min: Traffic Spike Begins                             │
│ - 5,000 requests/minute hit CDN                           │
│ - CDN cache: HIT (paste cached, 1 hour TTL)               │
│ - 99% of requests served from CDN edge                    │
│ - Origin receives only cache misses (~50 requests/minute)  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Origin Load Handling:                                        │
│                                                              │
│ 1. Cache Misses (~50/minute):                              │
│    - Redis metadata: GET meta:def45678 → HIT              │
│    - Redis content: GET content:def45678 → MISS           │
│    - S3 fetch: GetObject (paste content)                   │
│    - Cache in Redis (15min TTL)                           │
│    - Return to CDN, CDN caches (1 hour TTL)               │
│                                                              │
│ 2. CDN Cache Hits (5,000/minute):                         │
│    - Served from edge (< 10ms latency)                    │
│    - No origin load                                        │
│                                                              │
│ 3. Database Load:                                          │
│    - Only for cache misses (minimal)                      │
│    - No view count updates (async, not in path)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ CDN Optimization:                                            │
│                                                              │
│ - CDN cache hit rate: 99%+                                 │
│ - Origin requests: ~1% of total                            │
│ - S3 requests: ~50/hour (only cache misses)                │
│ - Redis memory: Stable (single paste cached)               │
│                                                              │
│ Result:                                                      │
│ - All 100K views succeed (< 50ms p99)                      │
│ - 99% served from CDN (< 10ms)                             │
│ - Origin infrastructure unaffected                         │
│ - S3 costs minimal (50 requests vs 100K)                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Concurrent Paste Deletion and View

```
Scenario: User deletes paste while someone else is viewing it
Edge Case: Race condition between deletion and cache refresh

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User A requests deletion of paste "ghi90123"        │
│ - DELETE /v1/pastes/ghi90123                               │
│ - Service marks as deleted in database                     │
│ - Service deletes from Redis cache                         │
│ - Response: 204 No Content                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+10ms: User B requests view of same paste (concurrent)     │
│ - GET /v1/pastes/ghi90123                                  │
│ - Request arrives before deletion completes                 │
│ - Cache check: GET meta:ghi90123 → Cache still has entry  │
│ - Metadata returned (race condition window)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Database Validation (Double-Check)                 │
│ - SELECT * FROM pastes WHERE id='ghi90123'                │
│ - Database confirms: deleted_at IS NOT NULL                │
│ - Record marked as deleted                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Response Handling                                   │
│                                                              │
│ Option A: Return 404 (Recommended)                          │
│ - Service returns 404 Not Found                            │
│ - Cache entry invalidated (DELETE meta:ghi90123)          │
│ - User experience: "Paste not found"                       │
│                                                              │
│ Option B: Return 410 Gone                                   │
│ - Service returns 410 Gone (permanent deletion)           │
│ - Clearer signal that paste was deleted                    │
│ - Cache entry invalidated                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Race Condition Handling:                                    │
│                                                              │
│ 1. Database is source of truth for deletion                │
│ 2. Cache is best-effort optimization                        │
│ 3. Double-check on view prevents serving deleted content   │
│ 4. Cache invalidation ensures consistency                  │
│                                                              │
│ Trade-off:                                                   │
│ - Adds one database query to view path                     │
│ - Ensures consistency (worth the latency cost)             │
└─────────────────────────────────────────────────────────────┘

**Alternative: Optimistic Cache Invalidation**
- Delete from cache immediately
- If deletion fails, cache repopulates on next read
- Faster but requires cache refresh logic
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| S3 Storage | $140 | 1% |
| S3 Transfer | $4,500 | 30% |
| CloudFront CDN | $8,500 | 57% |
| EC2 (App Servers) | $750 | 5% |
| RDS (PostgreSQL) | $450 | 3% |
| ElastiCache (Redis) | $450 | 3% |
| Other (monitoring, etc.) | $210 | 1% |
| **Total** | **$15,000** | 100% |

**Key Insight**: Data transfer is the dominant cost (87%). CDN optimization is critical.

### Cost Optimization Strategies

**Current Monthly Cost:** $15,000
**Target Monthly Cost:** $10,000 (33% reduction)

**Top 3 Cost Drivers:**
1. **CDN (CloudFront):** $8,500/month (57%) - Largest single component
2. **S3 Transfer:** $4,500/month (30%) - High data transfer volume
3. **S3 Storage:** $140/month (1%) - Relatively small

**Optimization Strategies (Ranked by Impact):**

1. **CDN Optimization (40% savings on CDN costs):**
   - **Current:** 1-hour TTL, 70% cache hit rate
   - **Optimization:** 
     - Increase TTL to 24 hours for popular pastes
     - Implement cache warming for top 10K pastes
     - Use CloudFront price classes (reduce data transfer costs by 20%)
     - Enable compression (Gzip) to reduce transfer size by 60%
   - **Savings:** $3,400/month (40% of $8,500)
   - **Trade-off:** Slightly stale data (acceptable for pastes)

2. **S3 Transfer Optimization (50% savings):**
   - **Current:** High origin requests due to low cache hit rate
   - **Optimization:** 
     - Increase cache hit rate from 70% to 90% (reduces origin requests by 67%)
     - Use S3 Transfer Acceleration only for critical uploads
     - Implement regional storage (store in cheapest region)
   - **Savings:** $2,250/month (50% of $4,500)
   - **Trade-off:** Need to monitor cache effectiveness

3. **Compression Enhancement:**
   - **Current:** Basic Gzip compression
   - **Optimization:** 
     - Enable Brotli compression (better than Gzip, 10-20% more savings)
     - Compress at origin (reduce transfer by additional 10%)
   - **Savings:** $850/month (10% of $8,500 CDN)
   - **Trade-off:** Slight CPU overhead (negligible)

4. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for compute and database
   - **Savings:** $480/month (40% of $1,200 compute+DB)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

5. **S3 Lifecycle Policies:**
   - **Current:** All pastes in standard storage
   - **Optimization:** 
     - Move pastes > 30 days to S3 Infrequent Access (50% cheaper)
     - Move pastes > 90 days to S3 Glacier (80% cheaper)
   - **Savings:** $70/month (50% of $140)
   - **Trade-off:** Slower access to old pastes (acceptable)

6. **Spot Instances for Cleanup Workers:**
   - **Current:** On-demand instances for cleanup jobs
   - **Optimization:** Use Spot instances (70% savings)
   - **Savings:** $50/month
   - **Trade-off:** Possible interruptions (acceptable for background jobs)

**Total Potential Savings:** $7,100/month (47% reduction)
**Optimized Monthly Cost:** $7,900/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Paste Creation | $0.0015 | $0.0008 | 47% |
| Paste View | $0.00015 | $0.00008 | 47% |
| Storage (per GB/month) | $0.023 | $0.012 | 48% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** CDN optimization, Compression → $4,250 savings
2. **Phase 2 (Month 2):** S3 transfer optimization, Reserved instances → $2,730 savings
3. **Phase 3 (Month 3):** Lifecycle policies, Spot instances → $120 savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor cache hit rates (target >90%)
- Monitor performance metrics (ensure no degradation)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Paste Service | Build | Core business logic |
| Database | Buy (RDS) | Managed, reliable, less ops |
| Cache | Buy (ElastiCache) | Managed Redis, auto-failover |
| Storage | Buy (S3) | Infinite scale, high durability |
| CDN | Buy (CloudFront) | Global edge network |
| Monitoring | Buy (DataDog/CloudWatch) | Feature-rich, integrations |

### At What Scale Does This Become Expensive?

| Scale | Monthly Cost | Cost per Paste | Notes |
|-------|--------------|----------------|-------|
| 10M pastes/month | $15,000 | $0.0015 | Current |
| 50M pastes/month | $50,000 | $0.0010 | Economies of scale |
| 100M pastes/month | $80,000 | $0.0008 | Need CDN optimization |
| 500M pastes/month | $300,000 | $0.0006 | Consider multi-CDN |

**Cost triggers:**
- > $50K/month: Review CDN strategy
- > $100K/month: Consider own edge servers
- > $200K/month: Multi-region optimization

### Cost per User Estimate

**Assumptions:**
- 1M active users
- Average 10 pastes created per user per month
- Average 10 views per paste

**Calculation:**
- Total pastes: 1M × 10 = 10M/month
- Total views: 10M × 10 = 100M/month
- Cost: $15,000/month
- Cost per user: $0.015/month
- Cost per paste: $0.0015
- Cost per view: $0.00015

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | AWS ALB, round-robin | < 1ms added latency |
| Rate Limiting | Redis sliding window | 10 req/hour anonymous |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 200ms |
| Security | API keys, input validation | Zero injection vulnerabilities |
| DR | Active-passive, us-west-2 | RTO < 15 min, RPO < 1 min |
| Cost | $15,000/month | $0.015 per user |

