# URL Shortener - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis. These topics ensure the URL shortener operates reliably in production at scale.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Layer 7 (Application) Load Balancing with AWS ALB:**

```
Internet → CloudFront (CDN) → ALB → URL Service Pods
```

**Why ALB over NLB?**
- Path-based routing (separate `/v1/` API from redirect)
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

    private final RedisTemplate<String, String> redisTemplate;

    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String redisKey = "ratelimit:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000L);

        // Use Redis sorted set for sliding window
        List<Object> results = redisTemplate.execute(new SessionCallback<List<Object>>() {
            @Override
            public List<Object> execute(RedisOperations operations) {
                operations.multi();
                operations.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
                operations.opsForZSet().add(redisKey, String.valueOf(now), now);
                operations.opsForZSet().count(redisKey, windowStart, now);
                operations.expire(redisKey, windowSeconds, TimeUnit.SECONDS);
                return operations.exec();
            }
        });

        Long count = (Long) results.get(2);
        return count != null && count <= limit;
    }
}
```

**Rate Limit Tiers:**

| Tier | URL Creation | Analytics API | Redirects |
|------|--------------|---------------|-----------|
| Anonymous | 10/hour | N/A | Unlimited |
| Free | 100/hour | 1000/hour | Unlimited |
| Pro | 10,000/hour | 100,000/hour | Unlimited |
| Enterprise | Unlimited | Unlimited | Unlimited |

**Response Headers:**

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String apiKey = request.getHeader("Authorization");
        RateLimitResult result = rateLimiter.check(apiKey);

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
| Redis Cache | Yes | Fall back to PostgreSQL |
| PostgreSQL | Yes | Return 503, no fallback |
| Kafka | Yes | Log locally, drop event |
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
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Redis GET | 50ms | 1 | None |
| PostgreSQL Query | 200ms | 2 | 100ms exponential |
| Kafka Produce | 100ms | 3 | 100ms exponential |
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
- 6-node cluster (3 primary + 3 replica)
- Automatic failover per slot
- No data loss for acknowledged writes

**Kafka:**
- 3 brokers
- Replication factor 3
- min.insync.replicas = 2

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: Analytics (Kafka events)
   - Impact: No click tracking, redirects still work
   - Trigger: Kafka unavailable or lag > 100,000

2. **Second to drop**: Rate limiting
   - Impact: Potential abuse, but service stays up
   - Trigger: Redis unavailable

3. **Third to drop**: Non-critical API endpoints
   - Impact: No URL creation, only redirects
   - Trigger: Database at capacity

4. **Last resort**: Serve from CDN only
   - Impact: Only cached redirects work
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
│  │  │ URL Service │              │ URL Service │                 │  │
│  │  │ (Pods 1-5)  │              │ (Pods 6-10) │                 │  │
│  │  └──────┬──────┘              └──────┬──────┘                 │  │
│  │         │                            │                        │  │
│  │  ┌──────┴────────────────────────────┴──────┐                 │  │
│  │  │  PostgreSQL Primary (AZ 1a)              │  │
│  │  │  ──sync repl──> PostgreSQL Replica (1b)  │  │
│  │  │  ──async repl──> PostgreSQL Replica (1c)  │  │
│  │  └──────────────────────────────────────────┘                 │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (3 primary + 3 replica across AZs)      │ │  │
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
│  │  PostgreSQL Replica (async replication from us-east-1)         │ │
│  │  Redis Replica (async replication from us-east-1)              │ │
│  │  URL Service (2 pods, minimal for DR readiness)                 │ │
│  │  ALB (standby, routes traffic only during failover)             │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region (AZ 1b) for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 1 minute for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **Cache Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Cache Warm-up:** On failover, cache populated from database
   - **Cache Invalidation:** Cross-region invalidation via pub/sub (optional)

3. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Replication Lag** | Stale reads in DR region | Acceptable for DR (RPO < 1 min), use sync replica for reads in primary |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
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
   - RPO: < 1 minute (async replication lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale data
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $10,000/month (full infrastructure)
- **DR Region:** $2,000/month (minimal infrastructure, 20% of primary)
- **Replication Bandwidth:** $200/month (cross-region data transfer)
- **Total Multi-Region Cost:** $12,200/month (22% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy

**Trade-offs:**
- **Cost:** 22% increase for DR capability
- **Complexity:** More complex operations (failover procedures, monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)

**Active-Passive Configuration:**

```
us-east-1 (Active)          us-west-2 (Passive)
┌─────────────────┐         ┌─────────────────┐
│ ALB             │         │ ALB (standby)   │
│ URL Service x10 │         │ URL Service x2  │
│ Redis Cluster   │   ───>  │ Redis (replica) │
│ PostgreSQL      │  async  │ PostgreSQL (DR) │
│ Kafka           │         │ Kafka (mirror)  │
└─────────────────┘         └─────────────────┘
```

**RTO/RPO Targets:**
- RPO (Recovery Point Objective): < 1 minute (async replication lag)
- RTO (Recovery Time Objective): < 15 minutes (manual failover)

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture database replication lag
   - Document active URL count
   - Verify cache hit rates

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in us-east-1 (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-2 min:** Promote database replica to primary in us-west-2
   - **T+2-3 min:** Update DNS records (Route53 health checks)
   - **T+3-10 min:** Warm up Redis cache from database
   - **T+10-12 min:** Verify all services healthy in DR region
   - **T+12-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+30 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check replication lag at failure time
   - Test URL creation: Create 10 test URLs, verify persistence
   - Test URL redirect: Redirect 100 test URLs, verify 100% success
   - Verify analytics: Check click events are being recorded
   - Monitor metrics: QPS, latency, error rate return to baseline

5. **Data Integrity Verification:**
   - Compare URL count: Pre-failover vs post-failover (should match)
   - Spot check: Verify 100 random URLs accessible
   - Check replication: Verify no gaps in database replication log

6. **Failback Procedure (T+30 to T+45 minutes):**
   - Restore primary region services
   - Reconfigure replication (DR → Primary)
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via replication lag)
- ✅ All active URLs remain accessible: 100% redirect success rate
- ✅ No data loss: URL count matches pre-failover
- ✅ Cache repopulates correctly: Cache hit rate returns to >90% within 10 minutes
- ✅ Analytics continue: Click events recorded without gaps

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

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

**Pool Exhaustion Handling:**

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        
        // Custom connection pool metrics
        config.setMetricRegistry(metricRegistry);
        
        // Pool exhaustion handling
        config.addDataSourceProperty("poolName", "url-service-pool");
        config.setInitializationFailTimeout(-1); // Fail fast on init errors
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public ConnectionPoolMetrics connectionPoolMetrics(DataSource dataSource) {
        return new ConnectionPoolMetrics((HikariDataSource) dataSource);
    }
}
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

### What Breaks First Under Load

**Order of failure at 10x traffic (120,000 QPS):**

1. **Database connections** (First)
   - Symptom: Connection pool exhausted
   - Fix: Add PgBouncer, increase pool size (see configuration above)

2. **Redis memory** (Second)
   - Symptom: Evictions increase, hit rate drops
   - Fix: Add nodes, increase memory

3. **Kafka consumer lag** (Third)
   - Symptom: Analytics delayed by hours
   - Fix: Add consumers, increase partitions

4. **Application CPU** (Fourth)
   - Symptom: Latency increases
   - Fix: Add more pods

---

## 2. Monitoring & Observability

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Redirect P50 | < 20ms | > 30ms | > 50ms |
| Redirect P95 | < 50ms | > 80ms | > 100ms |
| Redirect P99 | < 100ms | > 150ms | > 200ms |
| URL Create P50 | < 100ms | > 150ms | > 200ms |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Redirects/sec | 4,000 | > 10,000 | > 15,000 |
| URL Creates/sec | 40 | > 100 | > 150 |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Error rate (5xx) | < 0.01% | > 0.1% | > 1% |
| 404 rate | < 5% | > 10% | > 20% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 70% | > 90% |
| Memory | > 80% | > 95% |
| DB Connections | > 80% | > 95% |
| Redis Memory | > 70% | > 85% |

### Metrics Collection

```java
@Component
public class MetricsCollector {

    private final MeterRegistry registry;
    private final Counter urlsCreated;
    private final Counter redirectsTotal;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Timer redirectLatency;

    public MetricsCollector(MeterRegistry registry) {
        this.registry = registry;

        this.urlsCreated = Counter.builder("urls.created.total")
            .description("Total URLs created")
            .register(registry);

        this.redirectsTotal = Counter.builder("redirects.total")
            .description("Total redirects served")
            .tag("status", "success")
            .register(registry);

        this.redirectLatency = Timer.builder("redirect.latency")
            .description("Redirect latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordRedirect(String shortCode, long durationMs, boolean cacheHit) {
        redirectsTotal.increment();
        redirectLatency.record(durationMs, TimeUnit.MILLISECONDS);

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
| High Latency P99 | > 150ms for 5 min | Warning | Ticket |
| High Latency P99 | > 200ms for 2 min | Critical | Page on-call |
| Error Rate | > 0.1% for 5 min | Warning | Ticket |
| Error Rate | > 1% for 1 min | Critical | Page on-call |
| Cache Hit Rate | < 85% for 10 min | Warning | Ticket |
| Cache Hit Rate | < 70% for 5 min | Critical | Page on-call |
| Kafka Lag | > 10,000 for 10 min | Warning | Ticket |
| Kafka Lag | > 100,000 for 5 min | Critical | Page on-call |

### Logging Strategy

**Structured Logging Format:**

```java
@Slf4j
@RestController
public class RedirectController {

    @GetMapping("/{shortCode}")
    public ResponseEntity<Void> redirect(@PathVariable String shortCode, HttpServletRequest request) {
        long startTime = System.currentTimeMillis();
        
        try {
            UrlMapping mapping = urlService.getUrl(shortCode);
            
            log.info("redirect.success",
                kv("short_code", shortCode),
                kv("latency_ms", System.currentTimeMillis() - startTime),
                kv("cache_hit", mapping.isCacheHit()),
                kv("referrer", request.getHeader("Referer")),
                kv("user_agent", request.getHeader("User-Agent"))
            );
            
            return ResponseEntity.status(301)
                .header("Location", mapping.getOriginalUrl())
                .build();
                
        } catch (NotFoundException e) {
            log.warn("redirect.not_found",
                kv("short_code", shortCode),
                kv("latency_ms", System.currentTimeMillis() - startTime)
            );
            return ResponseEntity.notFound().build();
        }
    }
}
```

**PII Redaction:**
- IP addresses: Hash before logging
- User agents: Log as-is (not PII)
- Original URLs: Truncate to domain only in logs

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
2. Redis Lookup (child span)
3. PostgreSQL Query (child span, if cache miss)
4. Kafka Publish (child span)

### Health Checks

```java
@Component
public class SystemHealthIndicator implements HealthIndicator {

    private final RedisTemplate<String, String> redisTemplate;
    private final JdbcTemplate jdbcTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

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

        // Check Kafka (non-critical)
        try {
            kafkaTemplate.send("health-check", "ping").get(5, TimeUnit.SECONDS);
            details.put("kafka", "UP");
        } catch (Exception e) {
            details.put("kafka", "DOWN: " + e.getMessage());
            // Kafka down is not critical for redirects
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
- Request rate (by endpoint)
- Error rate (by status code)
- Latency percentiles (P50, P95, P99)
- Saturation (CPU, memory, connections)

**Business KPIs Dashboard:**
- URLs created per hour
- Redirects per hour
- Top 10 URLs by clicks
- Geographic distribution

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
        return "turl_" + key;  // e.g., turl_xK9mN2pQ...
    }

    public void storeApiKey(Long userId, String apiKey) {
        String hash = DigestUtils.sha256Hex(apiKey);
        String prefix = apiKey.substring(0, 12);

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

| Role | Create URL | Read URL | Delete URL | Analytics |
|------|------------|----------|------------|-----------|
| Anonymous | Yes (limited) | Yes (public) | No | No |
| User | Yes | Own only | Own only | Own only |
| Admin | Yes | All | All | All |

### Data Encryption

**At Rest:**
- PostgreSQL: AWS RDS encryption (AES-256)
- Redis: Not encrypted (in-memory, acceptable risk)
- Kafka: AWS MSK encryption

**In Transit:**
- All external traffic: TLS 1.3
- Internal traffic: mTLS in service mesh (optional)

### Input Validation and Sanitization

```java
@Service
public class UrlValidator {

    private static final Pattern URL_PATTERN = Pattern.compile(
        "^https?://[\\w.-]+(?:\\.[\\w.-]+)+[\\w\\-._~:/?#\\[\\]@!$&'()*+,;=]*$"
    );

    private static final Set<String> BLOCKED_DOMAINS = Set.of(
        "malware.com", "phishing.net"
    );

    public ValidationResult validate(String url) {
        // 1. Format validation
        if (!URL_PATTERN.matcher(url).matches()) {
            return ValidationResult.invalid("Invalid URL format");
        }

        // 2. Length check
        if (url.length() > 2048) {
            return ValidationResult.invalid("URL too long (max 2048 characters)");
        }

        // 3. Scheme check
        if (!url.startsWith("http://") && !url.startsWith("https://")) {
            return ValidationResult.invalid("Only HTTP and HTTPS URLs allowed");
        }

        // 4. Domain blocklist check
        String domain = extractDomain(url);
        if (BLOCKED_DOMAINS.contains(domain)) {
            return ValidationResult.invalid("Domain is blocked");
        }

        // 5. Safe browsing API check (async)
        safeBrowsingService.checkAsync(url);

        return ValidationResult.valid();
    }
}
```

### PII Handling and Compliance

**GDPR Compliance:**

1. **Right to Deletion**: Users can delete their URLs and associated analytics
2. **Data Minimization**: Only collect necessary data (no names, emails in analytics)
3. **IP Hashing**: Store SHA-256 hash of IP, not raw IP
4. **Retention Policy**: Analytics data deleted after 90 days

**Data Retention:**

| Data Type | Retention | Deletion Method |
|-----------|-----------|-----------------|
| URLs | Until expiration | Soft delete |
| Raw clicks | 90 days | Partition drop |
| Aggregated stats | 2 years | Archive to cold storage |
| User accounts | Until deletion request | Hard delete |

### Security Headers

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy("default-src 'self'")
                .xssProtection(xss -> xss.block(true))
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)
                )
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
| **Enumeration** | Rate limiting, non-sequential codes, honeypot URLs |
| **Abuse (spam/phishing)** | Domain blocklist, Safe Browsing API, user reports |
| **Injection (XSS)** | Input validation, CSP headers, output encoding |
| **Replay attacks** | Idempotency keys, timestamp validation |
| **DDoS** | CDN, rate limiting, auto-scaling |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Creates and Shares a URL

**Step-by-step:**

1. **User Action**: Marketing manager pastes long URL into web form
2. **Request**: `POST /v1/urls` with `{"original_url": "https://example.com/promo"}`
3. **API Gateway**: Validates API key, checks rate limit
4. **URL Service**: 
   - Validates URL format
   - Generates short code: `abc123`
   - Inserts into PostgreSQL
   - Caches in Redis: `SET url:abc123 "https://example.com/promo|301|0"`
5. **Response**: `201 Created` with `{"short_url": "https://tiny.url/abc123"}`
6. **User Action**: Shares `https://tiny.url/abc123` on Twitter
7. **Followers click link**:
   - CDN cache miss (first click)
   - Redis cache hit
   - 301 redirect returned
   - Click event published to Kafka
8. **CDN caches** the redirect for 1 hour
9. **Subsequent clicks**: Served from CDN edge (< 10ms)

### Journey 2: Redirect with Analytics

**Step-by-step:**

1. **User Action**: Clicks `https://tiny.url/abc123`
2. **CDN**: Cache miss, forwards to origin
3. **Load Balancer**: Routes to healthy URL Service pod
4. **URL Service**:
   - Redis: `GET url:abc123` → HIT
   - Parses cached value
   - Publishes click event to Kafka (async, non-blocking)
   - Returns 301 redirect
5. **Kafka Producer**: Sends to partition `hash(abc123) % 12`
6. **Response**: 301 with `Location: https://example.com/promo`
7. **Browser**: Follows redirect to destination
8. **[Async] Consumer**:
   - Polls batch of events
   - Enriches with GeoIP and user agent parsing
   - Batch inserts to ClickHouse
   - Increments Redis counter: `INCR clicks:abc123`
9. **[Async] Aggregation job**: Updates daily_stats table

### Failure & Recovery Walkthrough

**Scenario: Redis Cluster Partial Failure**

**RTO (Recovery Time Objective):** < 20 seconds (automatic failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, cache may be cold)

**Timeline:**

```
T+0s:    Redis node 2 crashes (handles slots 5461-10922)
T+0-5s:  Requests to affected slots fail
T+5s:    Redis cluster detects failure
T+10s:   Replica promoted to primary for affected slots
T+15s:   Cluster topology updated
T+20s:   All requests succeeding again
```

**What degrades:**
- Redirects for ~1/3 of URLs fail for 15-20 seconds
- Circuit breaker opens, falls back to database
- Database load increases 3x temporarily

**What stays up:**
- Redirects for unaffected slots
- URL creation (writes to different nodes)
- Analytics (Kafka independent)

**What recovers automatically:**
- Redis cluster self-heals
- Circuit breaker closes after success threshold
- Cache repopulates on demand

**What requires human intervention:**
- Investigate root cause
- Replace failed node
- Review capacity planning

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Timeouts prevent thread exhaustion
- Load shedding drops analytics before redirects

### High-Load / Contention Scenario: Viral URL Traffic Spike

```
Scenario: Celebrity tweets a short URL, causing 1M redirects in 5 minutes
Time: Peak traffic event

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Celebrity tweets short URL "abc123"                   │
│ - URL was created weeks ago (cold cache case)               │
│ - Expected: ~100 redirects/day normal traffic              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-30s: Traffic Spike Begins                               │
│ - 10,000 requests/second hit CDN                           │
│ - CDN cache: MISS (URL not popular before)                 │
│ - All requests forward to origin                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Origin Load Handling:                                        │
│                                                              │
│ 1. Load Balancer:                                            │
│    - Distributes across 10 URL Service pods                │
│    - Each pod: ~1,000 requests/second                      │
│                                                              │
│ 2. URL Service (per pod):                                   │
│    - Redis: GET url:abc123                                  │
│    - Cache HIT (warm cache from initial requests)          │
│    - Latency: ~2ms per request                             │
│    - Publish to Kafka (async, < 1ms)                       │
│                                                              │
│ 3. CDN Caching:                                             │
│    - First 100 requests miss, forward to origin            │
│    - Origin responds with 301 redirect                     │
│    - CDN caches redirect (1 hour TTL)                      │
│    - Next 999,900 requests: CDN HIT (< 10ms latency)      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Analytics Backpressure:                               │
│                                                              │
│ - 1M events published to Kafka                             │
│ - Partition: hash(abc123) % 12 = partition 5               │
│ - Consumer lag builds: 500K events queued                  │
│                                                              │
│ Protection:                                                  │
│ - Kafka buffers (7-day retention)                          │
│ - Consumers auto-scale (K8s HPA)                           │
│ - Analytics delayed but not lost                           │
│                                                              │
│ Result:                                                      │
│ - All 1M redirects succeed (< 50ms p99)                    │
│ - 99.99% served from CDN after warm-up                     │
│ - Analytics processes within 10 minutes                    │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Expired URL Access Attempt

```
Scenario: User tries to access expired URL with race condition
Edge Case: URL expires between cache check and redirect

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User clicks expired URL "xyz789"                    │
│ - URL expired 1 second ago                                  │
│ - Redis cache still has entry (TTL not expired yet)        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: CDN Cache Check                                     │
│ - CDN: Cache MISS (expired URLs not cached)                │
│ - Request forwarded to origin                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Redis Cache Check                                   │
│ - GET url:xyz789 → Returns cached value                    │
│ - Cache entry exists (hasn't expired in Redis yet)         │
│ - Service parses: "https://example.com/old|301|expired"    │
│ - Checks expiration timestamp: EXPIRED                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Database Validation (Double-Check)                 │
│ - SELECT * FROM urls WHERE short_code='xyz789'            │
│ - Database confirms: expired_at < NOW()                    │
│ - Record marked as expired                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Response Handling                                   │
│                                                              │
│ Option A: Soft Delete (Recommended)                         │
│ - Return 301 to landing page: "/expired?url=xyz789"       │
│ - Shows "This link has expired" message                    │
│ - User experience: Graceful degradation                     │
│                                                              │
│ Option B: Hard Delete                                       │
│ - Return 404 Not Found                                     │
│ - Cache entry deleted from Redis                           │
│ - User experience: Broken link                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Cache Cleanup:                                              │
│ - Async job: DELETE url:xyz789 from Redis                  │
│ - Prevents stale cache hits                                │
│ - Expiration job runs every 5 minutes                     │
└─────────────────────────────────────────────────────────────┘

**Race Condition Handling:**
- Database is source of truth for expiration
- Cache is best-effort optimization
- Double-check on expiration prevents stale redirects
- Async cleanup prevents cache pollution
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EKS) | $3,000 | 30% |
| Database (RDS) | $2,500 | 25% |
| Cache (ElastiCache) | $1,500 | 15% |
| CDN (CloudFront) | $1,500 | 15% |
| Kafka (MSK) | $1,000 | 10% |
| Storage (S3, EBS) | $300 | 3% |
| Network (data transfer) | $200 | 2% |
| **Total** | **$10,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $10,000
**Target Monthly Cost:** $7,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Database (RDS):** $2,500/month (25%) - Largest single component
2. **Compute (EKS):** $3,000/month (30%) - Multiple pods
3. **CDN (CloudFront):** $1,500/month (15%) - High traffic volume

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for compute and database
   - **Savings:** $2,200/month (40% of $5,500)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads
   - **Implementation:** Convert 80% of instances to Reserved (keep 20% on-demand for flexibility)

2. **Database Optimization:**
   - **Current:** r5.2xlarge (8 vCPU, 64 GB RAM) × 3 instances
   - **Optimization:** 
     - Right-size based on actual usage (likely r5.xlarge sufficient)
     - Use Aurora Serverless v2 for variable workloads
     - Enable compression for analytics data
   - **Savings:** $800/month (32% reduction)
   - **Trade-off:** Monitor performance, may need to scale back up

3. **CDN Caching Enhancement:**
   - **Current:** 1-hour TTL, 60% cache hit rate
   - **Optimization:** 
     - Increase TTL to 24 hours for popular URLs
     - Implement cache warming for top 10K URLs
     - Use CloudFront price classes (reduce data transfer costs)
   - **Savings:** $600/month (40% reduction)
   - **Trade-off:** Slightly stale data (acceptable for redirects)

4. **Spot Instances for Analytics:**
   - **Current:** On-demand instances for analytics consumers
   - **Optimization:** Use Spot instances (70% savings)
   - **Savings:** $300/month
   - **Trade-off:** Possible interruptions (acceptable for analytics, not critical path)

5. **Data Tiering:**
   - **Current:** All analytics data in hot storage
   - **Optimization:** 
     - Move data > 90 days to S3 Glacier (80% savings)
     - Keep only last 30 days in ClickHouse
   - **Savings:** $200/month
   - **Trade-off:** Slower access to old data (acceptable for analytics)

6. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
     - Use auto-scaling more aggressively
   - **Savings:** $200/month
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $4,300/month (43% reduction)
**Optimized Monthly Cost:** $5,700/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| URL Creation | $0.0001 | $0.00006 | 40% |
| URL Redirect | $0.00001 | $0.000006 | 40% |
| Analytics Query | $0.00005 | $0.00003 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Database optimization → $3,000 savings
2. **Phase 2 (Month 2):** CDN optimization, Spot instances → $900 savings
3. **Phase 3 (Month 3):** Data tiering, Right-sizing → $400 savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor performance metrics (ensure no degradation)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| URL Service | Build | Core business logic |
| Database | Buy (RDS) | Managed, reliable, less ops |
| Cache | Buy (ElastiCache) | Managed Redis, auto-failover |
| CDN | Buy (CloudFront) | Global edge network |
| Kafka | Buy (MSK) | Managed, less ops burden |
| Monitoring | Buy (DataDog/CloudWatch) | Feature-rich, integrations |

### Scale vs Cost

| Scale | QPS | Monthly Cost | Cost per 1M requests |
|-------|-----|--------------|----------------------|
| Current | 4,000 | $10,000 | $0.10 |
| 10x | 40,000 | $50,000 | $0.05 |
| 100x | 400,000 | $200,000 | $0.02 |

**Economies of scale**: Cost per request decreases as volume increases due to:
- Better CDN cache hit rates
- More efficient resource utilization
- Volume discounts

### Cost per User Estimate

**Assumptions:**
- 1M active users
- Average 10 URLs created per user per month
- Average 100 redirects per URL per month

**Calculation:**
- Total redirects: 1M × 10 × 100 = 1B/month
- Cost: $10,000/month
- Cost per user: $0.01/month
- Cost per redirect: $0.00001

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | AWS ALB, round-robin | < 1ms added latency |
| Rate Limiting | Redis sliding window | 100 req/hour free tier |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 100ms |
| Security | API keys, input validation | Zero injection vulnerabilities |
| DR | Active-passive, us-west-2 | RTO < 15 min, RPO < 1 min |
| Cost | $10,000/month | $0.01 per user |

