# Food Delivery - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis. These topics ensure the food delivery system operates reliably in production at scale.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Multi-Layer Load Balancing:**

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│       AWS Global Accelerator        │  ← Geographic routing
└─────────────────────────────────────┘
    │
    ├─── US-West Region ───────────────┐
    │                                  │
    ▼                                  ▼
┌─────────────────┐          ┌─────────────────┐
│  ALB (REST API) │          │  ALB (WebSocket)│
│  Path routing   │          │  Sticky sessions│
└────────┬────────┘          └────────┬────────┘
         │                            │
         ▼                            ▼
    App Servers               WebSocket Servers
```

**Path-Based Routing:**

| Path | Target |
|------|--------|
| /v1/restaurants/* | Restaurant Service |
| /v1/orders/* | Order Service |
| /v1/deliveries/* | Delivery Service |
| /ws/* | WebSocket Gateway |

### Rate Limiting Implementation

**Multi-Tier Rate Limiting:**

```java
@Service
public class RateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public RateLimitResult checkLimit(String userId, String endpoint) {
        RateLimitConfig config = getConfig(endpoint);
        String key = String.format("ratelimit:%s:%s", userId, endpoint);
        
        long now = System.currentTimeMillis();
        long windowStart = now - (config.getWindowSeconds() * 1000L);
        
        // Sliding window with Redis sorted set
        List<Object> results = redis.executePipelined((RedisCallback<Object>) conn -> {
            conn.zRemRangeByScore(key.getBytes(), 0, windowStart);
            conn.zAdd(key.getBytes(), now, String.valueOf(now).getBytes());
            conn.zCard(key.getBytes());
            conn.expire(key.getBytes(), config.getWindowSeconds());
            return null;
        });
        
        long count = (Long) results.get(2);
        boolean allowed = count <= config.getLimit();
        
        return RateLimitResult.builder()
            .allowed(allowed)
            .limit(config.getLimit())
            .remaining(Math.max(0, config.getLimit() - (int) count))
            .build();
    }
}
```

**Rate Limit Tiers:**

| Endpoint | Anonymous | Authenticated | Premium |
|----------|-----------|---------------|---------|
| Search | 30/min | 60/min | 200/min |
| Order placement | 3/min | 10/min | 30/min |
| Menu view | 60/min | 120/min | 500/min |
| Location update | N/A | 60/min | 60/min |

### Circuit Breaker Placement

**Services with Circuit Breakers:**

| Service | Dependency | Fallback |
|---------|------------|----------|
| Search Service | Elasticsearch | PostgreSQL query |
| Order Service | PostgreSQL | Return error |
| Payment Service | Stripe | Queue for retry |
| Notification | FCM/APNS | Queue for retry |
| Maps Service | Google Maps | Cached distances |

**Configuration (Resilience4j):**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      elasticsearch:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
      postgresql:
        slidingWindowSize: 50
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      payment:
        slidingWindowSize: 20
        failureRateThreshold: 30
        waitDurationInOpenState: 120s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Redis GET | 50ms | 1 | None |
| PostgreSQL Query | 500ms | 2 | 100ms exp |
| Elasticsearch | 1s | 2 | 200ms exp |
| Kafka Produce | 100ms | 3 | 100ms exp |
| Payment Auth | 10s | 2 | 1s exp |
| Push Notification | 5s | 3 | 1s exp |

### Replication and Failover

**PostgreSQL:**
- 1 Primary + 2 Read Replicas (same region)
- 1 Async Replica (DR region)
- Automatic failover with Patroni (< 30 seconds)

**Redis:**
- 3-node cluster
- Automatic failover per slot
- Sentinel for monitoring

**Elasticsearch:**
- 6 nodes (3 primary shards + 3 replicas)
- Automatic shard rebalancing

**Kafka:**
- 6 brokers across 3 AZs
- Replication factor 3
- min.insync.replicas = 2

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: Search recommendations
   - Impact: No personalized suggestions
   - Trigger: Elasticsearch latency > 2s

2. **Second to drop**: Real-time tracking updates
   - Impact: Updates every 30s instead of 5s
   - Trigger: WebSocket server overload

3. **Third to drop**: Promotional content
   - Impact: No deals shown
   - Trigger: High latency on main API

4. **Fourth to drop**: New restaurant onboarding
   - Impact: No new restaurants
   - Trigger: Database at capacity

5. **Last resort**: Order-only mode
   - Impact: Only existing customers, saved addresses
   - Trigger: Multiple critical failures

### Backup and Recovery

**RPO (Recovery Point Objective):** < 1 minute
- Maximum acceptable data loss
- Critical for orders - must minimize data loss
- Database replication with synchronous replica (RPO = 0 for critical data)
- Order state changes replicated in real-time

**RTO (Recovery Time Objective):** < 30 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Synchronous replication for critical data (orders)
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync
- **Elasticsearch Backups**: Daily snapshots to S3

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database synchronous replica to primary - 2 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 10 minutes
5. Restore Elasticsearch from latest snapshot - 10 minutes
6. Verify food delivery service health and resume traffic - 6 minutes

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
- [ ] No active orders during test window (or minimal active orders)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture active order count
   - Document active delivery partner count
   - Verify database replication lag (< 1 minute)
   - Test order placement (100 sample orders)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+30 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-10 min:** Warm up Redis cache from database
   - **T+10-15 min:** Rebalance Kafka partitions
   - **T+15-20 min:** Rebuild Elasticsearch index from database
   - **T+20-25 min:** Verify all services healthy in DR region
   - **T+25-28 min:** Resume traffic to DR region
   - **T+28-30 min:** Verify active orders continue without interruption

4. **Post-Failover Validation (T+30 to T+45 minutes):**
   - Verify RTO < 30 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 minute: Check database replication lag at failure time
   - Test order placement: Place 1,000 test orders, verify >99% success
   - Test active orders: Verify all active orders continue without data loss
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify location updates: Check delivery partner location updates working

5. **Data Integrity Verification:**
   - Compare order count: Pre-failover vs post-failover (should match within 1 min)
   - Spot check: Verify 100 random orders accessible
   - Check active orders: Verify no active orders lost (RPO=0 for critical data)
   - Test edge cases: Order status updates, delivery tracking, payment processing

6. **Failback Procedure (T+45 to T+55 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 1 minute
   - Update DNS to route traffic back to primary
   - Monitor for 10 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 30 minutes: Time from failure to service resumption
- ✅ RPO < 1 minute: Maximum data loss (verified via database replication lag)
- ✅ No active orders lost: All active orders continue without interruption
- ✅ Order placement works: >99% orders placed successfully
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Location updates functional: Delivery partner locations update correctly

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Customer Places Order and Tracks Delivery

**Step-by-step:**

1. **User Action**: Customer searches restaurants via `GET /v1/restaurants/search?q=pizza&lat=37.7749&lng=-122.4194`
2. **Search Service**: 
   - Queries Elasticsearch: Returns nearby restaurants
   - Filters by availability, ratings
   - Returns top 20 results
3. **Response**: `200 OK` with restaurant list
4. **User Action**: Customer selects restaurant, views menu, adds items to cart
5. **User Action**: Customer places order via `POST /v1/orders` with `{"restaurant_id": "rest_123", "items": [...], "delivery_address": {...}}`
6. **Order Service**: 
   - Validates order (restaurant open, items available, address valid)
   - Generates order ID: `order_abc123`
   - Creates order in PostgreSQL: `INSERT INTO orders (id, customer_id, restaurant_id, status, total)`
   - Publishes to Kafka: `{"event": "order_created", "order_id": "order_abc123"}`
   - Processes payment (external payment gateway)
7. **Response**: `201 Created` with `{"order_id": "order_abc123", "status": "confirmed", "estimated_delivery": "30 minutes"}`
8. **Restaurant receives order** (via WebSocket):
   - Notification: `{"type": "new_order", "order_id": "order_abc123"}`
   - Restaurant accepts: `PUT /v1/orders/order_abc123/accept`
9. **Order preparation** (15 minutes):
   - Restaurant updates: `PUT /v1/orders/order_abc123/status {"status": "preparing"}`
   - Notifies customer via WebSocket
10. **Delivery partner assigned**:
    - Matching service finds nearby partner
    - Partner accepts: `PUT /v1/orders/order_abc123/assign {"partner_id": "partner_xyz"}`
11. **Partner picks up** (5 minutes later):
    - Partner sends: `PUT /v1/orders/order_abc123/picked_up`
    - Order status: "out_for_delivery"
12. **Delivery tracking** (10 minutes):
    - Partner location updates every 4 seconds via WebSocket
    - Customer receives real-time location via `GET /v1/orders/order_abc123/tracking`
13. **Delivery completes**:
    - Partner sends: `PUT /v1/orders/order_abc123/delivered`
    - Order status: "delivered"
    - Customer receives notification
14. **Result**: Order delivered within 30 minutes, customer can rate

**Total time: ~30 minutes** (preparation 15min + delivery 15min)

### Journey 2: Restaurant Receives and Fulfills Order

**Step-by-step:**

1. **Order arrives** (via WebSocket notification)
2. **Restaurant accepts**:
   - Sends: `PUT /v1/orders/order_abc123/accept`
   - Order status: "accepted"
3. **Kitchen starts preparation**:
   - Sends: `PUT /v1/orders/order_abc123/status {"status": "preparing"}`
   - Estimated time: 15 minutes
4. **Order ready**:
   - Sends: `PUT /v1/orders/order_abc123/ready`
   - Order status: "ready_for_pickup"
   - Matching service finds delivery partner
5. **Partner picks up**:
   - Partner confirms pickup
   - Order status: "out_for_delivery"
6. **Result**: Restaurant fulfills order, partner delivers

**Total time: ~15 minutes** (preparation time)

### Failure & Recovery Walkthrough

**Scenario: Payment Gateway Failure During Order Placement**

**RTO (Recovery Time Objective):** < 5 minutes (gateway recovery + retry)  
**RPO (Recovery Point Objective):** 0 (orders queued, no data loss)

**Timeline:**

```
T+0s:    Payment gateway starts returning 503 errors
T+0-10s: Order placements fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Orders queued to local buffer (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   Payment gateway recovers
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Queued orders processed from buffer
T+5min:  All queued orders completed
```

**What degrades:**
- Order placements fail for 10-30 seconds
- Orders queued in buffer (no data loss)
- Search and browsing unaffected

**What stays up:**
- Restaurant search
- Menu viewing
- Order status tracking
- All read operations

**What recovers automatically:**
- Circuit breaker closes after gateway recovery
- Queued orders processed automatically
- No manual intervention required

**What requires human intervention:**
- Investigate payment gateway root cause
- Review capacity planning
- Consider alternative payment providers

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Request queuing prevents order loss
- Exponential backoff reduces gateway load
- Timeouts prevent thread exhaustion

### Disaster Recovery

**Multi-Region Setup:**

```
US-West-2 (Primary)              US-East-1 (DR)
┌─────────────────────┐         ┌─────────────────────┐
│ Active Traffic      │         │ Standby             │
│                     │         │                     │
│ ALB                 │         │ ALB (warm)          │
│ App Servers (30)    │   ───>  │ App Servers (5)     │
│ Redis Cluster       │  async  │ Redis (replica)     │
│ PostgreSQL Primary  │  repl   │ PostgreSQL (async)  │
│ Elasticsearch       │         │ Elasticsearch (warm)│
│ Kafka Cluster       │         │ Kafka (mirror)      │
└─────────────────────┘         └─────────────────────┘
```

**RTO/RPO Targets:**

| Scenario | RPO | RTO |
|----------|-----|-----|
| Single AZ failure | 0 | < 1 minute |
| Region failure | < 1 minute | < 30 minutes |
| Database failure | 0 (sync replica) | < 30 seconds |

### What Breaks First Under Load

**Order of failure at 5x traffic (during Super Bowl):**

1. **Search latency** (First)
   - Symptom: Search takes > 2 seconds
   - Cause: Elasticsearch overwhelmed
   - Fix: Add nodes, enable caching

2. **Database connections**
   - Symptom: Connection pool exhausted
   - Cause: Too many concurrent orders
   - Fix: PgBouncer, increase pool (see connection pool configuration below)

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
- With safety margin: 25 connections per pod (higher due to order processing load)
- 15 pods × 25 = 375 max connections to database
- Database max_connections: 500 (20% headroom)
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
   - PgBouncer pool: 150 connections
   - App pools can share PgBouncer connections efficiently

3. **Menu cache evictions**
   - Symptom: Cache hit rate drops
   - Cause: Too many restaurants accessed
   - Fix: Increase Redis memory

4. **Order processing lag**
   - Symptom: Orders delayed
   - Cause: Kafka consumers can't keep up
   - Fix: Add consumers, partitions

---

## 2. Monitoring & Observability

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Search P50 | < 200ms | > 500ms | > 1s |
| Search P99 | < 500ms | > 1s | > 2s |
| Order placement P50 | < 1s | > 2s | > 5s |
| Menu load P50 | < 300ms | > 500ms | > 1s |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Orders/minute | 30 | > 100 | > 200 |
| Searches/second | 60 | > 150 | > 300 |
| Active orders | 15K | > 30K | > 50K |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Order failure rate | < 1% | > 2% | > 5% |
| Payment failure rate | < 0.5% | > 1% | > 2% |
| API error rate (5xx) | < 0.1% | > 0.5% | > 1% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU (app servers) | > 70% | > 90% |
| Memory | > 80% | > 95% |
| Database connections | > 80% | > 95% |
| Redis memory | > 70% | > 85% |
| Elasticsearch heap | > 75% | > 90% |

### Metrics Collection

```java
@Component
public class OrderMetrics {
    
    private final MeterRegistry registry;
    
    private final Counter ordersPlaced;
    private final Counter ordersCompleted;
    private final Counter ordersFailed;
    private final Timer orderPlacementLatency;
    private final Gauge activeOrders;
    
    public OrderMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.ordersPlaced = Counter.builder("orders.placed.total")
            .description("Total orders placed")
            .register(registry);
            
        this.orderPlacementLatency = Timer.builder("order.placement.latency")
            .description("Time to place order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
            
        this.activeOrders = Gauge.builder("orders.active", this, 
            m -> orderRepository.countActive())
            .description("Currently active orders")
            .register(registry);
    }
    
    public void recordOrderPlacement(String restaurantId, boolean success, long durationMs) {
        ordersPlaced.increment();
        orderPlacementLatency.record(durationMs, TimeUnit.MILLISECONDS);
        
        registry.counter("orders.placed", 
            "restaurant_id", restaurantId,
            "success", String.valueOf(success))
            .increment();
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Order failure rate | > 3% for 5 min | Critical | Page on-call |
| Search latency P99 | > 2s for 5 min | Critical | Page on-call |
| No orders in 5 min | 0 orders | Critical | Page on-call |
| Payment failures | > 2% for 3 min | Critical | Page on-call |
| Database CPU | > 90% for 10 min | Warning | Ticket |
| Elasticsearch lag | > 1 min | Warning | Ticket |

### Logging Strategy

**Structured Logging:**

```java
@Slf4j
@Service
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        String orderId = generateOrderId();
        
        log.info("order.creation.started",
            kv("order_id", orderId),
            kv("customer_id", request.getCustomerId()),
            kv("restaurant_id", request.getRestaurantId()),
            kv("items_count", request.getItems().size()),
            kv("total", request.getTotal())
        );
        
        try {
            Order order = processOrder(request, orderId);
            
            log.info("order.creation.completed",
                kv("order_id", orderId),
                kv("duration_ms", order.getProcessingTimeMs()),
                kv("payment_status", order.getPaymentStatus())
            );
            
            return order;
        } catch (Exception e) {
            log.error("order.creation.failed",
                kv("order_id", orderId),
                kv("error", e.getMessage()),
                kv("error_type", e.getClass().getSimpleName())
            );
            throw e;
        }
    }
}
```

### Distributed Tracing

**Trace ID Propagation:**

```java
@Component
public class TracingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String traceId = httpRequest.getHeader("X-Trace-ID");
        
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

### Health Checks

```java
@Component
public class SystemHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean healthy = true;
        
        // Check PostgreSQL
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            details.put("postgresql", "UP");
        } catch (Exception e) {
            details.put("postgresql", "DOWN: " + e.getMessage());
            healthy = false;
        }
        
        // Check Redis
        try {
            redisTemplate.opsForValue().get("health");
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DOWN: " + e.getMessage());
            healthy = false;
        }
        
        // Check Elasticsearch
        try {
            esClient.ping(RequestOptions.DEFAULT);
            details.put("elasticsearch", "UP");
        } catch (Exception e) {
            details.put("elasticsearch", "DOWN: " + e.getMessage());
            // ES down is degraded, not critical
        }
        
        return healthy ? Health.up().withDetails(details).build()
                       : Health.down().withDetails(details).build();
    }
}
```

---

## 3. Security Considerations

### Authentication Mechanism

**JWT Token Authentication:**

```java
@Service
public class JwtService {
    
    public String generateToken(User user) {
        return Jwts.builder()
            .setSubject(user.getId())
            .claim("type", user.getType())
            .claim("restaurant_id", user.getRestaurantId())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 24 * 60 * 60 * 1000))
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }
}
```

### Authorization and Access Control

**Permission Model:**

| Resource | Customer | Restaurant | Partner | Admin |
|----------|----------|------------|---------|-------|
| Place order | Own | No | No | Yes |
| View order | Own | Own restaurant | Assigned | All |
| Update order status | Cancel own | Own restaurant | Assigned | All |
| Update menu | No | Own | No | All |
| View earnings | No | Own | Own | All |

### Data Encryption

**At Rest:**
- PostgreSQL: AWS RDS encryption (AES-256)
- Redis: ElastiCache encryption
- Elasticsearch: Encrypted at rest
- S3 (images): SSE-S3

**In Transit:**
- All external: TLS 1.3
- Internal services: TLS

### Input Validation

```java
@Service
public class OrderValidator {
    
    public ValidationResult validate(OrderRequest request) {
        List<String> errors = new ArrayList<>();
        
        // Validate restaurant
        Restaurant restaurant = restaurantRepository
            .findById(request.getRestaurantId())
            .orElse(null);
            
        if (restaurant == null) {
            errors.add("Restaurant not found");
            return ValidationResult.invalid(errors);
        }
        
        if (!restaurant.isOpen()) {
            errors.add("Restaurant is closed");
        }
        
        if (!restaurant.isAcceptingOrders()) {
            errors.add("Restaurant is not accepting orders");
        }
        
        // Validate items
        for (OrderItem item : request.getItems()) {
            MenuItem menuItem = menuItemRepository
                .findById(item.getItemId())
                .orElse(null);
                
            if (menuItem == null) {
                errors.add("Item not found: " + item.getItemId());
            } else if (!menuItem.isAvailable()) {
                errors.add("Item not available: " + menuItem.getName());
            }
        }
        
        // Validate minimum order
        if (request.getSubtotal().compareTo(restaurant.getMinimumOrder()) < 0) {
            errors.add("Order below minimum: " + restaurant.getMinimumOrder());
        }
        
        // Validate delivery address
        if (!isWithinDeliveryRadius(restaurant, request.getDeliveryAddress())) {
            errors.add("Address outside delivery area");
        }
        
        return errors.isEmpty() ? ValidationResult.valid() 
                               : ValidationResult.invalid(errors);
    }
}
```

### PII Handling and Compliance

**GDPR/CCPA Compliance:**

1. **Data Minimization**: Only collect necessary data
2. **Right to Access**: Export user data on request
3. **Right to Deletion**: Delete user data within 30 days
4. **Consent**: Explicit consent for marketing

**Data Retention:**

| Data Type | Retention | Deletion Method |
|-----------|-----------|-----------------|
| Order history | 7 years | Required for tax |
| Delivery locations | 90 days | Hard delete |
| Payment data | 7 years | Required for compliance |
| User profiles | Until deletion | Soft delete, then hard |

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Fake orders** | Phone verification, payment auth |
| **Menu scraping** | Rate limiting, CAPTCHA |
| **Payment fraud** | 3D Secure, velocity checks |
| **Account takeover** | 2FA, device fingerprinting |
| **Partner fraud** | GPS verification, photo proof |
| **DDoS** | CDN, rate limiting, auto-scaling |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Customer Orders Food

**Step-by-step:**

1. **Customer opens app**
   - App requests location permission
   - Sends location to search service

2. **Customer browses restaurants**
   - Search Service queries Elasticsearch
   - Results cached in Redis (5 min)
   - Returns ranked list with ETAs

3. **Customer views menu**
   - Menu Service checks Redis cache
   - Cache miss: Load from PostgreSQL
   - Returns full menu with availability

4. **Customer adds items to cart**
   - Client-side cart management
   - Real-time price calculation

5. **Customer places order**
   - Order Service validates items
   - Pricing Service calculates total
   - Payment Service authorizes card
   - Order created in PostgreSQL
   - Event published to Kafka

6. **Restaurant receives order**
   - WebSocket notification to tablet
   - Restaurant has 5 min to accept
   - Restaurant confirms order

7. **Kitchen prepares food**
   - Status: PREPARING
   - Customer notified via push

8. **Food ready**
   - Status: READY
   - Delivery Service assigns partner
   - Partner notified via app

9. **Partner picks up**
   - Partner confirms pickup
   - Status: PICKED_UP
   - Customer sees partner on map

10. **Delivery complete**
    - Partner marks delivered
    - Payment captured
    - Rating prompts sent

### Journey 2: Restaurant Manages Orders

**Step-by-step:**

1. **Restaurant opens tablet app**
   - WebSocket connection established
   - Pending orders loaded

2. **New order arrives**
   - Audio alert plays
   - Order details shown
   - 5-minute countdown starts

3. **Restaurant accepts order**
   - Status: CONFIRMED
   - Estimated prep time set
   - Customer notified

4. **Restaurant marks preparing**
   - Status: PREPARING
   - Kitchen starts cooking

5. **Food ready**
   - Restaurant marks READY
   - Partner dispatched
   - Customer notified

6. **Partner picks up**
   - Restaurant confirms handoff
   - Order leaves restaurant

### Failure & Recovery Walkthrough

**Scenario: Payment Service Outage**

**RTO (Recovery Time Objective):** < 5 minutes (manual failover to backup payment provider)  
**RPO (Recovery Point Objective):** < 1 minute (orders in-flight may need retry)

**Timeline:**

```
T+0s:    Stripe API returns 503
T+0-30s: Payment requests fail
T+30s:   Circuit breaker opens
T+30s+:  Orders queued for retry
T+5min:  Stripe recovers
T+5min:  Circuit breaker half-open
T+6min:  Queued payments processed
T+10min: Normal operation
```

**What degrades:**
- New orders can't be placed
- Existing orders continue normally
- Customers see "Payment temporarily unavailable"

**What stays up:**
- Restaurant operations
- Delivery tracking
- Order status updates

**What recovers automatically:**
- Circuit breaker closes after success
- Queued payments processed
- Customers notified of success

**What requires human intervention:**
- Investigate root cause
- Handle failed payments manually
- Customer support for complaints

### High-Load / Contention Scenario: Dinner Rush Hour

```
Scenario: Friday night dinner rush, thousands of orders placed simultaneously
Time: 7:00 PM - 9:00 PM (peak ordering hours)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Dinner rush begins, 5,000 orders/minute              │
│ - Expected: ~500 orders/minute normal traffic              │
│ - 10x traffic spike                                         │
│ - Popular restaurants: 100+ orders queued                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-60min: Order Processing Load                          │
│ - Order Service: 5,000 orders/minute                      │
│ - Each order: Validate → Payment → Notify → Assign        │
│ - Restaurant notifications: 5,000/minute                   │
│ - Partner matching: 5,000/minute                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Order Service Scaling:                                 │
│    - Auto-scale: 10 → 50 instances (5x scale)             │
│    - Load distributed via load balancer                   │
│    - Each instance: 100 orders/minute                     │
│    - Latency: 200ms per order (within target)              │
│                                                              │
│ 2. Payment Processing:                                    │
│    - Payment service: 5,000 transactions/minute           │
│    - Stripe API: Handles 10K transactions/minute (safe)   │
│    - Queue buffers if payment service lags                │
│    - Payment latency: 300ms (acceptable)                  │
│                                                              │
│ 3. Restaurant Notifications:                              │
│    - WebSocket: Real-time push to restaurant tablets      │
│    - Queue for offline restaurants (SMS/email fallback)   │
│    - Notification service: 5,000 notifications/minute     │
│    - Latency: 100ms (WebSocket), 5s (SMS)                 │
│                                                              │
│ 4. Partner Matching:                                      │
│    - Matching algorithm: 5,000 matches/minute             │
│    - Geospatial queries: Redis (handles 10K queries/min)  │
│    - Match latency: 2-5 seconds (acceptable)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All 5,000 orders/minute processed successfully          │
│ - P99 latency: 500ms (under 1s target)                    │
│ - Auto-scaling handles spike gracefully                    │
│ - Restaurants receive orders in real-time                  │
│ - Partners matched within 5 seconds                       │
│ - No service degradation                                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Order Cancellation After Restaurant Acceptance

```
Scenario: Customer cancels order after restaurant has started preparation
Edge Case: Cancellation policy and refund handling

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Customer places order                                │
│ - Order ID: order_789                                      │
│ - Status: PENDING                                          │
│ - Payment: $25.00 (charged)                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+30s: Restaurant accepts order                            │
│ - Status: PENDING → CONFIRMED                              │
│ - Kitchen starts preparation                               │
│ - Estimated prep time: 20 minutes                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+2min: Customer cancels order                            │
│ - Cancel request: POST /orders/order_789/cancel           │
│ - Current status: CONFIRMED (preparation started)         │
│ - Cancellation policy: No refund after acceptance          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Cancellation Policy Application                │
│                                                              │
│ Policy Rules:                                              │
│ 1. Before acceptance (PENDING): Full refund                │
│ 2. After acceptance, < 5 min: 50% refund                  │
│ 3. After acceptance, > 5 min: No refund                   │
│ 4. After pickup: No refund                                 │
│                                                              │
│ Current State:                                             │
│ - Status: CONFIRMED                                        │
│ - Time since acceptance: 2 minutes                        │
│ - Policy: 50% refund ($12.50)                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Cancellation Flow                               │
│                                                              │
│ 1. Validate Cancellation:                                 │
│    - Check order status (CONFIRMED)                        │
│    - Calculate time since acceptance (2 min)              │
│    - Apply policy: 50% refund eligible                    │
│                                                              │
│ 2. Notify Restaurant:                                     │
│    - Send cancellation notification                        │
│    - Restaurant can stop preparation if not started        │
│    - Update order status: CONFIRMED → CANCELLED           │
│                                                              │
│ 3. Process Refund:                                        │
│    - Initiate refund: $12.50 (50%)                        │
│    - Stripe refund API call                              │
│    - Refund processed (3-5 business days)                 │
│    - Customer notified via email                          │
│                                                              │
│ 4. Update Metrics:                                        │
│    - Increment cancellation counter                       │
│    - Track refund amount                                  │
│    - Update restaurant cancellation rate                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Order cancelled successfully                             │
│ - 50% refund processed ($12.50)                           │
│ - Restaurant notified (can stop preparation)              │
│ - Customer refunded within policy guidelines              │
│ - Metrics updated for analytics                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EKS) | $8,000 | 29% |
| Database (RDS) | $5,000 | 18% |
| Elasticsearch | $4,000 | 14% |
| Cache (ElastiCache) | $1,000 | 4% |
| Kafka (MSK) | $2,000 | 7% |
| CDN (CloudFront) | $1,500 | 5% |
| Maps API | $3,000 | 11% |
| Storage (S3) | $500 | 2% |
| Load Balancers | $500 | 2% |
| Monitoring | $1,000 | 4% |
| Other | $1,100 | 4% |
| **Total** | **$27,600** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $27,600
**Target Monthly Cost:** $18,000 (35% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $10,000/month (36%) - Largest single component
2. **Elasticsearch:** $8,000/month (29%) - Search infrastructure
3. **Database (RDS):** $5,000/month (18%) - PostgreSQL database

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute and database):**
   - **Current:** On-demand instances for stable workloads
   - **Optimization:** 1-year Reserved Instances for 80% of compute and database
   - **Savings:** $4,800/month (40% of $12K compute+DB)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Elasticsearch Right-sizing (30% savings):**
   - **Current:** Over-provisioned Elasticsearch cluster
   - **Optimization:** 
     - Right-size based on actual query patterns
     - Use smaller instance types where possible
     - Optimize index settings
   - **Savings:** $2,400/month (30% of $8K Elasticsearch)
   - **Trade-off:** Need careful monitoring to avoid performance issues

3. **Spot Instances for Search Indexers (70% savings):**
   - **Current:** On-demand instances for search indexers
   - **Optimization:** Use Spot instances for search indexers
   - **Savings:** $1,400/month (70% of $2K search indexers)
   - **Trade-off:** Possible interruptions (acceptable for batch jobs)

4. **CDN Caching (40% savings on bandwidth):**
   - **Current:** Direct S3 access for images
   - **Optimization:** 
     - Aggressive caching (increase cache hit rate to 95%)
     - Use CloudFront CDN for images
   - **Savings:** $800/month (40% of $2K bandwidth)
   - **Trade-off:** Slight complexity increase (acceptable)

5. **Maps API Caching (50% savings):**
   - **Current:** High number of external Maps API calls
   - **Optimization:** 
     - Cache distance calculations (TTL: 5 minutes)
     - Cache geocoding results (TTL: 24 hours)
   - **Savings:** $500/month (50% of $1K Maps API)
   - **Trade-off:** Slight staleness (acceptable for most use cases)

6. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $1,000/month (10% of $10K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $10,900/month (39% reduction)
**Optimized Monthly Cost:** $16,700/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Order Placement | $0.002 | $0.0012 | 40% |
| Search Request | $0.0001 | $0.00006 | 40% |
| Location Update | $0.00001 | $0.000006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Elasticsearch Right-sizing → $7.2K savings
2. **Phase 2 (Month 2):** Spot instances, CDN Caching → $2.2K savings
3. **Phase 3 (Month 3):** Maps API Caching, Right-sizing → $1.5K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor search latency (ensure no degradation)
- Monitor cache hit rates (target >95% for CDN)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Core services | Build | Competitive advantage |
| Database | Buy (RDS) | Managed, reliable |
| Search | Buy (ES) | Complex to operate |
| Cache | Buy (ElastiCache) | Managed Redis |
| Kafka | Buy (MSK) | Managed, less ops |
| Payments | Buy (Stripe) | Compliance, reliability |
| Maps | Buy (Google) | Best quality |
| Push notifications | Buy (FCM/APNS) | Required for mobile |

### Scale vs Cost

| Scale | Orders/Day | Monthly Cost | Cost/Order |
|-------|------------|--------------|------------|
| Current | 500K | $27,600 | $0.002 |
| 2x | 1M | $45,000 | $0.0015 |
| 5x | 2.5M | $90,000 | $0.0012 |
| 10x | 5M | $150,000 | $0.001 |

**Economies of scale**: Cost per order decreases due to:
- Better cache hit rates
- More efficient resource utilization
- Volume discounts

### Cost per Order Breakdown

**At 500K orders/day:**

| Component | Cost/Order |
|-----------|-----------|
| Compute | $0.0005 |
| Database | $0.0003 |
| Search | $0.0003 |
| Other | $0.0009 |
| **Total** | **$0.002** |

**Revenue per order (25% commission on $25):**
- Revenue: $6.25
- Infrastructure cost: $0.002
- **Gross margin on infra: 99.97%**

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | ALB with path routing | < 1ms latency |
| Rate Limiting | Redis sliding window | Per-endpoint limits |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 500ms search |
| Security | JWT + validation | Zero breaches |
| DR | Active-passive multi-region | RTO < 30 min |
| Cost | $27,600/month | $0.002/order |

