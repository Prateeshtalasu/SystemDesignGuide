# Ride Hailing - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis. These topics ensure the ride-hailing system operates reliably in production at scale.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Multi-Layer Load Balancing:**

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│  AWS Global Accelerator / Route 53  │  ← Geographic routing
└─────────────────────────────────────┘
    │
    ├─── US-West Region ───────────────┐
    │                                  │
    ▼                                  ▼
┌─────────────────┐          ┌─────────────────┐
│  ALB (REST API) │          │  NLB (WebSocket)│
│  Layer 7        │          │  Layer 4        │
│  Path routing   │          │  Sticky sessions│
└────────┬────────┘          └────────┬────────┘
         │                            │
         ▼                            ▼
    App Servers               WebSocket Servers
```

**Why ALB for REST, NLB for WebSocket?**

| Load Balancer | Use Case | Reason |
|---------------|----------|--------|
| ALB (Layer 7) | REST API | Path-based routing, health checks |
| NLB (Layer 4) | WebSocket | Lower latency, connection persistence |

**WebSocket Sticky Sessions:**

```yaml
# NLB Target Group
target_group:
  protocol: TCP
  port: 8080
  stickiness:
    enabled: true
    type: source_ip
  health_check:
    protocol: TCP
    interval: 10s
```

### Rate Limiting Implementation

**Multi-Tier Rate Limiting:**

```java
@Service
public class RateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    // Sliding window rate limiter
    public RateLimitResult checkLimit(String userId, String endpoint, int limit, int windowSec) {
        String key = String.format("ratelimit:%s:%s", userId, endpoint);
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSec * 1000L);
        
        List<Object> results = redis.executePipelined((RedisCallback<Object>) conn -> {
            conn.zRemRangeByScore(key.getBytes(), 0, windowStart);
            conn.zAdd(key.getBytes(), now, String.valueOf(now).getBytes());
            conn.zCard(key.getBytes());
            conn.expire(key.getBytes(), windowSec);
            return null;
        });
        
        long count = (Long) results.get(2);
        boolean allowed = count <= limit;
        
        return RateLimitResult.builder()
            .allowed(allowed)
            .limit(limit)
            .remaining(Math.max(0, limit - (int) count))
            .resetTime(now + (windowSec * 1000L))
            .build();
    }
}
```

**Rate Limit Tiers:**

| Endpoint | Anonymous | Authenticated | Premium |
|----------|-----------|---------------|---------|
| Ride request | 5/min | 10/min | 30/min |
| Location update | N/A | 20/sec | 20/sec |
| Fare estimate | 20/min | 60/min | 200/min |
| Profile update | 5/min | 10/min | 30/min |

**Response Headers:**

```http
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1705312860
Retry-After: 45
```

### Circuit Breaker Placement

**Services with Circuit Breakers:**

| Service | Dependency | Fallback |
|---------|------------|----------|
| Matching Service | Redis | PostGIS query |
| Trip Service | PostgreSQL | Return error |
| Payment Service | Stripe | Queue for retry |
| Notification | FCM/APNS | Queue for retry |
| Maps Service | Google Maps | Cached routes |

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
      payment:
        slidingWindowSize: 20
        failureRateThreshold: 30
        waitDurationInOpenState: 120s
      maps:
        slidingWindowSize: 50
        failureRateThreshold: 40
        waitDurationInOpenState: 60s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Redis GET | 50ms | 1 | None |
| PostgreSQL Query | 500ms | 2 | 100ms exp |
| Kafka Produce | 100ms | 3 | 100ms exp |
| Payment Auth | 10s | 2 | 1s exp |
| Maps API | 2s | 2 | 500ms exp |
| Push Notification | 5s | 3 | 1s exp |

**Jitter Implementation:**

```java
public long calculateBackoff(int attempt, long baseMs, double jitterFactor) {
    long exponentialDelay = baseMs * (1L << attempt);
    long jitter = (long) (exponentialDelay * jitterFactor * Math.random());
    return exponentialDelay + jitter;
}
```

### Replication and Failover

**PostgreSQL:**
- 1 Primary + 2 Read Replicas (same region)
- 1 Async Replica (DR region)
- Automatic failover with Patroni (< 30 seconds)
- Sync replication to 1 replica for zero data loss

**Redis:**
- 6-node cluster (3 primary + 3 replica)
- Automatic failover per slot
- Cross-slot operations avoided

**Kafka:**
- 6 brokers across 3 AZs
- Replication factor 3
- min.insync.replicas = 2
- Unclean leader election disabled

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: Analytics events
   - Impact: Surge pricing delayed, metrics incomplete
   - Trigger: Kafka lag > 100,000 or broker down

2. **Second to drop**: Push notifications
   - Impact: Users don't get updates, must poll
   - Trigger: FCM/APNS errors > 50%

3. **Third to drop**: Surge pricing
   - Impact: Fixed pricing, potential driver shortage
   - Trigger: Surge calculator overwhelmed

4. **Fourth to drop**: Non-critical API endpoints
   - Impact: No history, no profile updates
   - Trigger: Database at capacity

5. **Last resort**: Matching only mode
   - Impact: Only basic ride matching works
   - Trigger: Multiple critical failures

### Backup and Recovery

**RPO (Recovery Point Objective):** < 1 minute
- Maximum acceptable data loss
- Critical for ride bookings - must minimize data loss
- Database replication with synchronous replica (RPO = 0 for critical data)
- Ride state changes replicated in real-time

**RTO (Recovery Time Objective):** < 15 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Synchronous replication for critical data (rides)
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync
- **Kafka Mirroring**: Real-time mirroring to DR region

**Restore Steps:**
1. Detect primary region failure (health checks) - 30 seconds
2. Promote database synchronous replica to primary - 1 minute
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 5 minutes
5. Verify ride service health and resume traffic - 7.5 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 minute (synchronous for critical data)
- [ ] Redis replication lag < 30 seconds
- [ ] Kafka replication lag < 1 minute
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] No active rides during test window (or minimal active rides)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture active ride count
   - Document active driver/rider count
   - Verify database replication lag (< 1 minute)
   - Test ride matching (100 sample requests)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-8 min:** Warm up Redis cache from database
   - **T+8-10 min:** Rebalance Kafka partitions
   - **T+10-12 min:** Verify all services healthy in DR region
   - **T+12-14 min:** Resume traffic to DR region
   - **T+14-15 min:** Verify active rides continue without interruption

4. **Post-Failover Validation (T+15 to T+30 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check database replication lag at failure time
   - Test ride matching: Match 1,000 test ride requests, verify >99% success
   - Test active rides: Verify all active rides continue without data loss
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify location updates: Check driver/rider location updates working

5. **Data Integrity Verification:**
   - Compare ride count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random rides accessible
   - Check active rides: Verify no active rides lost (RPO=0 for critical data)
   - Test edge cases: Surge pricing, driver matching, payment processing

6. **Failback Procedure (T+30 to T+40 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 10 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via database replication lag)
- ✅ No active rides lost: All active rides continue without interruption
- ✅ Ride matching works: >99% ride requests matched successfully
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Location updates functional: Driver/rider locations update correctly

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Requests Ride and Completes Trip

**Step-by-step:**

1. **User Action**: Rider opens app, requests ride via `POST /v1/rides` with `{"pickup": {"lat": 37.7749, "lng": -122.4194}, "dropoff": {"lat": 37.7849, "lng": -122.4294}}`
2. **Ride Service**: 
   - Validates request
   - Generates ride ID: `ride_abc123`
   - Queries driver matching service: Finds nearby drivers (within 2km)
   - Sends notifications to top 5 drivers via WebSocket
3. **Response**: `201 Created` with `{"ride_id": "ride_abc123", "status": "matching", "estimated_wait": 5}`
4. **Driver accepts** (30 seconds later):
   - Driver sends: `PUT /v1/rides/ride_abc123/accept`
   - Ride Service: Updates status to "matched", assigns driver
   - Notifies rider via WebSocket: `{"ride_id": "ride_abc123", "status": "matched", "driver": {...}}`
5. **Driver arrives** (5 minutes later):
   - Driver sends: `PUT /v1/rides/ride_abc123/arrived`
   - Ride Service: Updates status to "arrived"
   - Notifies rider
6. **Trip starts**:
   - Driver sends: `PUT /v1/rides/ride_abc123/start`
   - Ride Service: Updates status to "in_progress", starts trip timer
   - Location updates stream via WebSocket every 4 seconds
7. **Trip completes** (15 minutes later):
   - Driver sends: `PUT /v1/rides/ride_abc123/complete`
   - Ride Service: Calculates fare, processes payment, updates status to "completed"
   - Notifies rider with receipt
8. **Result**: Ride completed, payment processed, rating prompt sent

**Total time: ~20 minutes** (matching 30s + wait 5min + trip 15min)

### Journey 2: Driver Goes Online and Receives Requests

**Step-by-step:**

1. **Driver Action**: Driver opens app, goes online via `PUT /v1/drivers/me/status {"status": "online"}`
2. **Driver Service**: 
   - Updates driver status in Redis: `SET driver:driver123:status "online"`
   - Adds to geospatial index: `GEOADD drivers:active 37.7749 -122.4194 driver123`
   - Subscribes to ride request notifications
3. **Ride Request Arrives**:
   - Matching service queries: `GEORADIUS drivers:active 37.7749 -122.4194 2 km`
   - Finds driver123 within 2km
   - Sends notification via WebSocket: `{"type": "ride_request", "ride_id": "ride_xyz789", "pickup": {...}}`
4. **Driver accepts**:
   - Sends: `POST /v1/rides/ride_xyz789/accept`
   - Matching service: Removes driver from available pool
   - Assigns driver to ride
5. **Result**: Driver receives and accepts ride within 30 seconds

**Total latency: Matching ~100ms, Notification ~50ms, Total ~150ms**

### Failure & Recovery Walkthrough

**Scenario: Redis Cluster Failure During Matching**

**RTO (Recovery Time Objective):** < 2 minutes (cluster failover + driver re-indexing)  
**RPO (Recovery Point Objective):** < 30 seconds (driver location updates lost during failover)

**Timeline:**

```
T+0s:    Redis cluster primary node crashes (handles driver locations)
T+0-5s:  Driver location updates fail
T+5s:    Cluster detects failure
T+10s:   Replica promoted to primary
T+15s:   Cluster topology updated
T+20s:   Driver location updates resume
T+30s:   Geospatial index rebuilt from database
T+60s:   All matching queries succeeding again
```

**What degrades:**
- Driver matching unavailable for 30-60 seconds
- Ride requests queued (no data loss)
- Location updates lost during failover window

**What stays up:**
- Ride status updates (PostgreSQL)
- Payment processing
- User authentication
- WebSocket connections

**What recovers automatically:**
- Redis cluster promotes replica
- Driver locations re-indexed from database
- Matching service resumes
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of cluster failure
- Replace failed node
- Review cluster capacity

**Cascading failure prevention:**
- Cluster replication prevents data loss
- Circuit breakers prevent retry storms
- Request queuing prevents ride loss
- Fallback to database for driver locations

### Disaster Recovery

**Multi-Region Setup:**

```
US-West-2 (Primary)              US-East-1 (DR)
┌─────────────────────┐         ┌─────────────────────┐
│ Active Traffic      │         │ Standby             │
│                     │         │                     │
│ ALB + NLB           │         │ ALB + NLB (warm)    │
│ App Servers (60)    │   ───>  │ App Servers (10)    │
│ Redis Cluster       │  async  │ Redis (replica)     │
│ PostgreSQL Primary  │  repl   │ PostgreSQL (async)  │
│ Kafka Cluster       │         │ Kafka (mirror)      │
└─────────────────────┘         └─────────────────────┘
```

**RTO/RPO Targets:**

| Scenario | RPO | RTO |
|----------|-----|-----|
| Single AZ failure | 0 | < 1 minute |
| Region failure | < 1 minute | < 15 minutes |
| Database failure | 0 (sync replica) | < 30 seconds |
| Redis failure | < 30 seconds | < 1 minute |

**Failover Procedure:**

1. Detect: Health checks fail for > 30 seconds
2. Decide: Automated for AZ, manual for region
3. Execute: DNS failover, promote replicas
4. Verify: Smoke tests, traffic ramp-up
5. Communicate: Status page, push notifications

### What Breaks First Under Load

**Order of failure at 10x traffic (1.1M QPS):**

1. **WebSocket connections** (First)
   - Symptom: Connection refused, timeouts
   - Cause: Server memory exhaustion
   - Fix: Add WebSocket servers, increase limits

2. **Redis memory**
   - Symptom: Evictions, slow queries
   - Cause: Too many active drivers/rides
   - Fix: Add cluster nodes, increase memory

3. **Database connections**
   - Symptom: Connection pool exhausted
   - Cause: Too many concurrent queries
   - Fix: PgBouncer, increase pool size (see connection pool configuration below)

### Database Connection Pool Configuration

**PostgreSQL Connection Pool (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 30          # Max connections per application instance
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
For ride-hailing services:
- 8 CPU cores per pod
- High transaction rate (ride booking requires transactions)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for concurrent rides: 30 connections per pod
- 100 pods × 30 = 3,000 max connections
- Use PgBouncer: 200 connections per PostgreSQL instance
```

**PostGIS Connection Pool (for Geospatial Queries):**

```yaml
# Separate pool for geospatial queries (read-heavy)
geospatial:
  datasource:
    hikari:
      maximum-pool-size: 20          # Lower pool for read replicas
      minimum-idle: 5
      connection-timeout: 30000
      read-only: true                 # Use read replicas
```

**Pool Exhaustion Handling:**

1. **Monitoring:**
   - Alert when pool usage > 80%
   - Track wait time for connections (critical for ride booking)
   - Monitor connection leak detection

2. **Circuit Breaker:**
   - If pool exhausted for > 3 seconds, open circuit breaker
   - Return 503 for non-critical operations
   - Critical operations (ride booking) wait with timeout

3. **Connection Pooler (PgBouncer):**
   - Place PgBouncer between app servers and PostgreSQL
   - Transaction pooling mode (efficient for short transactions)
   - PgBouncer pool: 200 connections per PostgreSQL instance

**Redis Connection Pool:**

```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 50               # Max connections per instance
        max-idle: 20                 # Max idle connections
        min-idle: 10                 # Min idle connections
      shutdown-timeout: 200ms
```

4. **Kafka consumer lag**
   - Symptom: Events delayed by minutes
   - Cause: Consumers can't keep up
   - Fix: Add consumers, increase partitions

5. **Maps API quota**
   - Symptom: 429 errors from Google
   - Cause: Exceeded daily quota
   - Fix: Caching, quota increase, fallback

---

## 2. Monitoring & Observability

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Matching P50 | < 5s | > 8s | > 15s |
| Matching P99 | < 15s | > 20s | > 30s |
| Location update P50 | < 100ms | > 200ms | > 500ms |
| API P50 | < 200ms | > 500ms | > 1s |
| WebSocket message | < 500ms | > 1s | > 2s |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Location updates/sec | 60K | > 100K | > 150K |
| Ride requests/min | 300 | > 500 | > 800 |
| Active WebSocket conns | 500K | > 700K | > 900K |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Matching failure rate | < 5% | > 10% | > 20% |
| Payment failure rate | < 1% | > 2% | > 5% |
| API error rate (5xx) | < 0.1% | > 0.5% | > 1% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU (app servers) | > 70% | > 90% |
| Memory (app servers) | > 80% | > 95% |
| Redis memory | > 70% | > 85% |
| DB connections | > 80% | > 95% |
| Kafka consumer lag | > 10K | > 100K |

### Metrics Collection

```java
@Component
public class RideMetrics {
    
    private final MeterRegistry registry;
    
    private final Counter ridesRequested;
    private final Counter ridesCompleted;
    private final Counter matchingFailures;
    private final Timer matchingLatency;
    private final Gauge activeRides;
    private final Gauge onlineDrivers;
    
    public RideMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.ridesRequested = Counter.builder("rides.requested.total")
            .description("Total ride requests")
            .register(registry);
            
        this.matchingLatency = Timer.builder("matching.latency")
            .description("Time to match rider with driver")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
            
        this.activeRides = Gauge.builder("rides.active", this, 
            m -> rideRepository.countActive())
            .description("Currently active rides")
            .register(registry);
            
        this.onlineDrivers = Gauge.builder("drivers.online", this,
            m -> driverService.countOnline())
            .description("Currently online drivers")
            .register(registry);
    }
    
    public void recordMatchingAttempt(String vehicleType, boolean success, long durationMs) {
        ridesRequested.increment();
        matchingLatency.record(durationMs, TimeUnit.MILLISECONDS);
        
        if (!success) {
            matchingFailures.increment();
        }
        
        registry.counter("matching.attempts", 
            "vehicle_type", vehicleType,
            "success", String.valueOf(success))
            .increment();
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Matching latency P99 | > 20s for 5 min | Critical | Page on-call |
| Matching failure rate | > 15% for 3 min | Critical | Page on-call |
| No drivers online | < 100 for 5 min | Critical | Page on-call |
| Redis memory | > 85% for 10 min | Warning | Ticket |
| Kafka lag | > 50K for 10 min | Warning | Ticket |
| Payment failures | > 3% for 5 min | Critical | Page on-call |
| WebSocket disconnects | > 10K/min | Warning | Ticket |

### Logging Strategy

**Structured Logging:**

```java
@Slf4j
@Service
public class RideService {
    
    public Ride createRide(RideRequest request) {
        String rideId = generateRideId();
        
        log.info("ride.requested",
            kv("ride_id", rideId),
            kv("rider_id", request.getRiderId()),
            kv("pickup_lat", request.getPickup().getLat()),
            kv("pickup_lng", request.getPickup().getLng()),
            kv("vehicle_type", request.getVehicleType())
        );
        
        try {
            Ride ride = matchAndCreate(request, rideId);
            
            log.info("ride.matched",
                kv("ride_id", rideId),
                kv("driver_id", ride.getDriverId()),
                kv("matching_time_ms", ride.getMatchingTimeMs()),
                kv("eta_minutes", ride.getEtaMinutes())
            );
            
            return ride;
        } catch (NoDriversException e) {
            log.warn("ride.no_drivers",
                kv("ride_id", rideId),
                kv("pickup_lat", request.getPickup().getLat()),
                kv("pickup_lng", request.getPickup().getLng()),
                kv("search_radius_km", e.getSearchRadius())
            );
            throw e;
        }
    }
}
```

**PII Redaction:**

```java
@Component
public class PiiRedactor {
    
    public String redactPhone(String phone) {
        if (phone == null || phone.length() < 4) return "***";
        return "***-***-" + phone.substring(phone.length() - 4);
    }
    
    public String redactEmail(String email) {
        if (email == null) return "***";
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) return "***";
        return email.charAt(0) + "***" + email.substring(atIndex);
    }
    
    public String hashIp(String ip) {
        return DigestUtils.sha256Hex(ip + salt).substring(0, 16);
    }
}
```

### Distributed Tracing

**Trace ID Propagation:**

```java
@Component
public class TracingFilter implements WebFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String traceId = exchange.getRequest().getHeaders()
            .getFirst("X-Trace-ID");
            
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        // Add to MDC for logging
        MDC.put("traceId", traceId);
        
        // Add to response
        exchange.getResponse().getHeaders().add("X-Trace-ID", traceId);
        
        // Propagate to downstream services
        return chain.filter(exchange)
            .contextWrite(Context.of("traceId", traceId));
    }
}
```

**Span Boundaries:**

1. HTTP Request (root span)
2. Redis Geospatial Query (child span)
3. Driver Selection (child span)
4. WebSocket Notification (child span)
5. Kafka Event Publish (child span)
6. PostgreSQL Write (child span)

### Health Checks

```java
@Component
public class SystemHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean healthy = true;
        
        // Check Redis
        try {
            redisTemplate.opsForValue().get("health");
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
        
        // Check Kafka
        try {
            kafkaAdmin.describeCluster().nodes().get(5, TimeUnit.SECONDS);
            details.put("kafka", "UP");
        } catch (Exception e) {
            details.put("kafka", "DEGRADED: " + e.getMessage());
            // Kafka down is not critical for core functionality
        }
        
        // Check online drivers
        long onlineDrivers = driverService.countOnline();
        details.put("online_drivers", onlineDrivers);
        if (onlineDrivers < 100) {
            details.put("driver_warning", "Low driver count");
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
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30
```

### Dashboards

**Operations Dashboard:**
- Request rate by endpoint
- Error rate by service
- Latency percentiles (P50, P95, P99)
- Active WebSocket connections
- Kafka consumer lag

**Business Dashboard:**
- Active rides
- Online drivers
- Rides per hour
- Average wait time
- Completion rate
- Revenue per hour

**Infrastructure Dashboard:**
- CPU/Memory utilization
- Network throughput
- Database connections
- Redis memory usage
- Disk I/O

---

## 3. Security Considerations

### Authentication Mechanism

**JWT Token Authentication:**

```java
@Service
public class JwtService {
    
    private final SecretKey key;
    
    public String generateToken(User user, String deviceId) {
        return Jwts.builder()
            .setSubject(user.getId())
            .claim("type", user.getType())  // RIDER or DRIVER
            .claim("device_id", deviceId)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 24 * 60 * 60 * 1000))
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }
    
    public TokenClaims validateToken(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody();
            
        return TokenClaims.builder()
            .userId(claims.getSubject())
            .userType(claims.get("type", String.class))
            .deviceId(claims.get("device_id", String.class))
            .build();
    }
}
```

**Phone Verification:**

```java
@Service
public class PhoneVerificationService {
    
    public void sendVerificationCode(String phoneNumber) {
        String code = generateSecureCode();
        
        // Store with 5-minute expiry
        redisTemplate.opsForValue().set(
            "verify:" + phoneNumber,
            hashCode(code),
            Duration.ofMinutes(5)
        );
        
        // Send via SMS
        smsService.send(phoneNumber, "Your code: " + code);
    }
    
    public boolean verifyCode(String phoneNumber, String code) {
        String stored = redisTemplate.opsForValue().get("verify:" + phoneNumber);
        
        if (stored != null && stored.equals(hashCode(code))) {
            redisTemplate.delete("verify:" + phoneNumber);
            return true;
        }
        return false;
    }
}
```

### Authorization and Access Control

**Permission Model:**

| Resource | Rider | Driver | Admin |
|----------|-------|--------|-------|
| Create ride | Own | No | Yes |
| View ride | Own | Assigned | All |
| Update ride status | Cancel own | Assigned | All |
| View driver location | During ride | Own | All |
| View earnings | No | Own | All |
| Update profile | Own | Own | All |

### Data Encryption

**At Rest:**
- PostgreSQL: AWS RDS encryption (AES-256)
- Redis: ElastiCache encryption
- Kafka: AWS MSK encryption
- S3 (documents): SSE-S3

**In Transit:**
- All external: TLS 1.3
- Internal services: mTLS (optional)
- WebSocket: WSS

### Input Validation

```java
@Service
public class LocationValidator {
    
    private static final double MIN_LAT = -90.0;
    private static final double MAX_LAT = 90.0;
    private static final double MIN_LNG = -180.0;
    private static final double MAX_LNG = 180.0;
    
    // Service area bounding box
    private static final Polygon SERVICE_AREA = createServiceArea();
    
    public ValidationResult validate(Location location) {
        // Basic range check
        if (location.getLat() < MIN_LAT || location.getLat() > MAX_LAT) {
            return ValidationResult.invalid("Invalid latitude");
        }
        if (location.getLng() < MIN_LNG || location.getLng() > MAX_LNG) {
            return ValidationResult.invalid("Invalid longitude");
        }
        
        // Service area check
        Point point = new Point(location.getLng(), location.getLat());
        if (!SERVICE_AREA.contains(point)) {
            return ValidationResult.invalid("Location outside service area");
        }
        
        // Velocity check (for location updates)
        if (location.getSpeed() != null && location.getSpeed() > 200) {
            return ValidationResult.invalid("Unrealistic speed");
        }
        
        return ValidationResult.valid();
    }
}
```

### Location Spoofing Prevention

```java
@Service
public class LocationIntegrityService {
    
    public boolean isLocationSuspicious(String driverId, Location newLocation) {
        Location lastLocation = getLastLocation(driverId);
        
        if (lastLocation == null) {
            return false;
        }
        
        // Calculate distance and time
        double distanceKm = calculateDistance(lastLocation, newLocation);
        long timeDiffSeconds = Duration.between(
            lastLocation.getTimestamp(), 
            newLocation.getTimestamp()
        ).getSeconds();
        
        // Maximum possible speed: 200 km/h = 55 m/s
        double maxPossibleDistance = (timeDiffSeconds * 55) / 1000.0;
        
        if (distanceKm > maxPossibleDistance * 1.5) {  // 50% buffer
            log.warn("Suspicious location jump",
                kv("driver_id", driverId),
                kv("distance_km", distanceKm),
                kv("time_seconds", timeDiffSeconds),
                kv("max_possible_km", maxPossibleDistance)
            );
            return true;
        }
        
        return false;
    }
}
```

### PII Handling and Compliance

**GDPR/CCPA Compliance:**

1. **Data Minimization**: Only collect necessary data
2. **Right to Access**: Export user data on request
3. **Right to Deletion**: Delete user data within 30 days
4. **Consent**: Explicit consent for location tracking

**Data Retention:**

| Data Type | Retention | Deletion Method |
|-----------|-----------|-----------------|
| Ride history | 7 years | Anonymize after 3 years |
| Location history | 90 days | Hard delete |
| Payment data | 7 years | Required for tax |
| User profiles | Until deletion | Soft delete, then hard |
| Support tickets | 3 years | Archive, then delete |

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **GPS spoofing** | Velocity checks, device attestation |
| **Account takeover** | Phone verification, device binding |
| **Fare manipulation** | Server-side calculation only |
| **Driver impersonation** | Photo verification, background checks |
| **Payment fraud** | Address verification, velocity limits |
| **DDoS** | Rate limiting, CDN, auto-scaling |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Complete Ride Flow

**Step-by-step:**

1. **Rider opens app**
   - App sends location to get nearby drivers count
   - Redis GEORADIUS returns driver count

2. **Rider enters destination**
   - API: `GET /v1/rides/estimate`
   - Pricing Service calculates fare with surge
   - Maps API returns route and ETA

3. **Rider confirms ride**
   - API: `POST /v1/rides`
   - Trip Service creates ride (PostgreSQL)
   - Matching Service queries Redis for drivers
   - Top 5 drivers notified via WebSocket

4. **Driver accepts (within 15 seconds)**
   - WebSocket: Driver sends ACCEPT
   - Matching Service assigns driver
   - Other drivers notified of cancellation
   - Rider notified via WebSocket

5. **Driver en route**
   - Driver location updates every 4 seconds
   - Rider sees driver on map
   - ETA updates dynamically

6. **Driver arrives**
   - Driver marks ARRIVED
   - Rider gets push notification
   - 5-minute wait timer starts

7. **Trip starts**
   - Driver marks IN_PROGRESS
   - Route tracking begins
   - Location history stored in TimescaleDB

8. **Trip ends**
   - Driver marks COMPLETED
   - Final fare calculated
   - Payment authorized and captured
   - Rating prompts sent to both parties

### Journey 2: No Drivers Available

**Step-by-step:**

1. **Rider requests ride**
   - Matching Service searches 5km radius
   - No online drivers found

2. **Expand search**
   - Search 10km, 15km, 20km
   - Still no drivers

3. **Return error**
   - API returns 503 with retry_after
   - App shows "No drivers available" message

4. **Background retry**
   - App polls every 30 seconds
   - Eventually driver comes online
   - Rider notified via push

### Failure & Recovery Walkthrough

**Scenario: Redis Cluster Partial Failure**

**RTO (Recovery Time Objective):** < 30 seconds (automatic failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, driver locations may be stale for < 30s)

**Timeline:**

```
T+0s:    Redis node 2 crashes (handles slots 5461-10922)
T+0-5s:  Requests to affected slots timeout
T+5s:    Redis cluster detects failure
T+10s:   Replica promoted to primary
T+15s:   Cluster topology updated
T+20s:   All requests succeeding
```

**What degrades:**
- Location updates for ~1/3 of drivers fail temporarily
- Matching queries may timeout
- Circuit breaker opens, falls back to PostGIS

**What stays up:**
- Active rides continue (state in PostgreSQL)
- Payments process normally
- Notifications still work

**What recovers automatically:**
- Redis cluster self-heals
- Circuit breaker closes after success
- Driver locations repopulate on next update

**What requires human intervention:**
- Investigate root cause
- Replace failed node
- Review capacity

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Timeouts prevent thread exhaustion
- PostGIS fallback keeps matching working

### High-Load / Contention Scenario: Surge Pricing Event

```
Scenario: Major event ends (concert, sports game), thousands request rides simultaneously
Time: Post-event rush (peak demand)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Event ends, 10,000 riders request rides simultaneously│
│ - Location: Stadium area (high density)                    │
│ - Expected: ~100 requests/second normal traffic            │
│ - 100x traffic spike                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-60s: Matching Service Overload                         │
│ - 10,000 requests hit matching service                     │
│ - Each request: Geospatial query (5km radius)             │
│ - Available drivers: 500 (high demand, low supply)         │
│ - Driver-to-rider ratio: 1:20 (severe imbalance)          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Surge Pricing Activation:                              │
│    - Algorithm detects demand spike (10K requests)        │
│    - Supply low (500 drivers)                              │
│    - Surge multiplier: 3.0x (300% price increase)         │
│    - Notify riders: "Surge pricing active"                │
│                                                              │
│ 2. Geospatial Query Optimization:                        │
│    - Queries batched (10 requests per query)              │
│    - Redis geospatial index handles 1K queries/sec        │
│    - PostGIS fallback if Redis overloaded                  │
│    - Latency: 50ms per batch (acceptable)                  │
│                                                              │
│ 3. Matching Algorithm:                                    │
│    - First 500 requests: Matched immediately              │
│    - Remaining 9,500: Queued for next available driver    │
│    - Queue processing: 10 matches/second                  │
│    - Estimated wait: 15-20 minutes for queued riders      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - 500 rides matched immediately (5% of requests)          │
│ - 9,500 riders in queue (95% wait)                        │
│ - Surge pricing encourages more drivers to come online    │
│ - Queue clears within 20 minutes                          │
│ - No service crashes (system handles load gracefully)      │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Concurrent Ride Cancellation and Assignment

```
Scenario: Rider cancels ride while driver is being assigned
Edge Case: Race condition between cancellation and matching

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: Rider requests ride                                 │
│ - Request ID: ride_123                                     │
│ - Status: PENDING                                          │
│ - Matching service searches for driver                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+100ms: Matching service finds driver                    │
│ - Driver ID: driver_456                                    │
│ - Status update: PENDING → ASSIGNED                        │
│ - Database transaction begins                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+150ms: Rider cancels ride (concurrent)                  │
│ - Cancel request: DELETE /rides/ride_123                  │
│ - Status check: PENDING (transaction not committed)        │
│ - Status update: PENDING → CANCELLED                      │
│ - Database transaction begins                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Race Condition: Two Concurrent Updates                    │
│                                                              │
│ Transaction 1 (Matching):                                  │
│   UPDATE rides SET status='ASSIGNED', driver_id='456'     │
│   WHERE id='ride_123' AND status='PENDING'                │
│                                                              │
│ Transaction 2 (Cancellation):                              │
│   UPDATE rides SET status='CANCELLED'                      │
│   WHERE id='ride_123' AND status='PENDING'                │
│                                                              │
│ Problem: Both succeed, final state unclear                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Optimistic Locking with Version                  │
│                                                              │
│ 1. Add version column to rides table:                     │
│    - id, status, version, driver_id, ...                  │
│    - Version increments on each update                    │
│                                                              │
│ 2. Matching transaction:                                  │
│   UPDATE rides SET status='ASSIGNED', driver_id='456',   │
│                    version=version+1                       │
│   WHERE id='ride_123' AND status='PENDING' AND version=1  │
│   Result: 1 row updated (success)                         │
│                                                              │
│ 3. Cancellation transaction (runs after):                 │
│   UPDATE rides SET status='CANCELLED', version=version+1  │
│   WHERE id='ride_123' AND status='PENDING' AND version=1  │
│   Result: 0 rows updated (status already ASSIGNED)        │
│                                                              │
│ 4. Handle cancellation of assigned ride:                  │
│    - Check current status before cancelling               │
│    - If ASSIGNED, notify driver of cancellation           │
│    - Apply cancellation policy (fee if < 2 min)           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - First transaction wins (matching succeeds)              │
│ - Second transaction fails gracefully (ride already assigned)│
│ - Driver notified of cancellation if ride was assigned    │
│ - No data corruption or inconsistent state                │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EKS) | $15,000 | 25% |
| Database (RDS) | $8,000 | 13% |
| Cache (ElastiCache) | $3,000 | 5% |
| Kafka (MSK) | $5,000 | 8% |
| Maps API | $20,000 | 33% |
| Data Transfer | $5,000 | 8% |
| Load Balancers | $2,000 | 3% |
| Storage (S3, EBS) | $2,000 | 3% |
| Monitoring | $1,000 | 2% |
| **Total** | **$61,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $61,000
**Target Monthly Cost:** $40,000 (34% reduction)

**Top 3 Cost Drivers:**
1. **Maps API:** $19,500/month (32%) - Largest single component (external API costs)
2. **Compute (EC2):** $15,000/month (25%) - Application servers
3. **Database (RDS):** $12,000/month (20%) - PostgreSQL database

**Optimization Strategies (Ranked by Impact):**

1. **Maps API Caching (50% savings on API calls):**
   - **Current:** High number of external Maps API calls
   - **Optimization:** 
     - Cache route calculations (TTL: 5 minutes)
     - Cache ETA calculations (TTL: 1 minute)
     - Cache geocoding results (TTL: 24 hours)
   - **Savings:** $9,750/month (50% of $19.5K Maps API)
   - **Trade-off:** Slight staleness (acceptable for most use cases)

2. **Reserved Instances (40% savings on compute and database):**
   - **Current:** On-demand instances for stable workloads
   - **Optimization:** 1-year Reserved Instances for 80% of compute and database
   - **Savings:** $8,640/month (40% of $21.6K compute+DB)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

3. **Spot Instances for Non-Critical Workers (70% savings):**
   - **Current:** On-demand instances for analytics workers
   - **Optimization:** Use Spot instances for analytics workers
   - **Savings:** $2,100/month (70% of $3K analytics workers)
   - **Trade-off:** Possible interruptions (acceptable for batch jobs)

4. **Data Tiering (50% savings on old data):**
   - **Current:** All data in standard storage
   - **Optimization:** 
     - Move rides > 90 days to S3 Glacier (80% cheaper)
     - Move rides > 30 days to S3 Infrequent Access (50% cheaper)
   - **Savings:** $1,200/month (30% of $4K storage)
   - **Trade-off:** Slower access to old rides (acceptable for archive)

5. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $1,500/month (10% of $15K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $23,190/month (38% reduction)
**Optimized Monthly Cost:** $37,810/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Ride Request | $0.0004 | $0.00025 | 38% |
| Route Calculation | $0.00013 | $0.000065 | 50% |
| Location Update | $0.00001 | $0.000006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Maps API Caching, Reserved Instances → $18.39K savings
2. **Phase 2 (Month 2):** Spot instances, Data Tiering → $3.3K savings
3. **Phase 3 (Month 3):** Right-sizing → $1.5K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor cache hit rates (target >80% for Maps API)
- Monitor ride matching latency (ensure no degradation)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Core services | Build | Competitive advantage |
| Database | Buy (RDS) | Managed, reliable |
| Cache | Buy (ElastiCache) | Managed Redis |
| Kafka | Buy (MSK) | Managed, less ops |
| Maps | Buy (Google) | Best quality, coverage |
| Payments | Buy (Stripe) | Compliance, reliability |
| Push notifications | Buy (FCM/APNS) | Required for mobile |

### Scale vs Cost

| Scale | Rides/Day | Monthly Cost | Cost/Ride |
|-------|-----------|--------------|-----------|
| Current | 5M | $61,000 | $0.0004 |
| 2x | 10M | $100,000 | $0.00033 |
| 5x | 25M | $200,000 | $0.00027 |
| 10x | 50M | $350,000 | $0.00023 |

**Economies of scale**: Cost per ride decreases due to:
- Better cache hit rates
- More efficient resource utilization
- Volume discounts on APIs

### Cost per Ride Breakdown

**At 5M rides/day:**

| Component | Cost/Ride |
|-----------|-----------|
| Compute | $0.0001 |
| Database | $0.00005 |
| Maps API | $0.00013 |
| Other | $0.00012 |
| **Total** | **$0.0004** |

**Revenue per ride (20% commission on $15 average):**
- Revenue: $3.00
- Infrastructure cost: $0.0004
- **Gross margin on infra: 99.99%**

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | ALB + NLB | < 1ms added latency |
| Rate Limiting | Redis sliding window | Per-endpoint limits |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 15s matching |
| Security | JWT + phone verification | Zero breaches |
| DR | Active-passive multi-region | RTO < 15 min |
| Cost | $61,000/month | $0.0004/ride |

