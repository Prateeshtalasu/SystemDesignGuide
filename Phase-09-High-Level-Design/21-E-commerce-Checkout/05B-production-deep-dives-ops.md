# E-commerce Checkout - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Service Distribution:**
- Cart Service: 6 instances behind load balancer
- Order Service: 8 instances behind load balancer
- Payment Service: 15 instances (I/O bound, gateway limited)
- Inventory Service: 3 instances (high throughput)

**Load Balancing Strategy:**
- Round-robin for stateless services
- Sticky sessions for cart operations (session affinity)
- Health check-based routing

### Rate Limiting Implementation

**Per-User Rate Limiting:**

```java
@Service
public class CheckoutRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean isAllowed(String userId, String operation) {
        String key = "rate_limit:" + userId + ":" + operation;
        String count = redis.opsForValue().get(key);
        
        int limit = getLimit(operation);  // checkout: 10/min, payment: 5/min
        
        if (count == null) {
            redis.opsForValue().set(key, "1", Duration.ofMinutes(1));
            return true;
        }
        
        int current = Integer.parseInt(count);
        if (current >= limit) {
            return false;
        }
        
        redis.opsForValue().increment(key);
        return true;
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Payment Gateway | Yes | Queue payment for retry |
| Inventory Service | Yes | Reject checkout |
| Order Service | Yes | Queue order creation |
| Shipping Service | Yes | Use default shipping cost |

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-gateway:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      inventory:
        slidingWindowSize: 50
        failureRateThreshold: 30
        waitDurationInOpenState: 30s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Payment Processing | 10s | 2 | 1s, 3s |
| Inventory Check | 2s | 3 | 100ms, 500ms, 1s |
| Order Creation | 5s | 2 | 1s, 2s |
| Shipping Calculation | 3s | 2 | 500ms, 1s |

### Replication and Failover

**Services:**
- Multi-AZ deployment (3 availability zones)
- Auto-scaling based on load
- Health check-based failover

**Database:**
- PostgreSQL primary-replica setup
- Automatic failover (RTO < 30 seconds)
- Synchronous replication for critical data

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Order history queries (can be slower)
2. **Second**: Cart recommendations
3. **Third**: Shipping cost calculation (use default)
4. **Last**: Stop accepting new orders

### Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes
- Maximum acceptable data loss
- Based on database replication lag
- Orders stored with synchronous replication

**RTO (Recovery Time Objective):** < 15 minutes
- Maximum acceptable downtime
- Time to restore service via failover
- Includes database promotion and service restart

**Backup Strategy:**
- **Database Backups**: Hourly incremental, daily full
- **Cross-region Replication**: Async replication to DR region
- **Redis Snapshots**: Hourly snapshots of cart data

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database replica to primary - 2 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 5 minutes
5. Verify checkout service health and resume traffic - 6 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 5 minutes
- [ ] Payment processor failover configured
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] No active checkout sessions during test window (or minimal)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture checkout success rate
   - Document active cart/order count
   - Verify database replication lag (< 5 minutes)
   - Test checkout flow (100 sample checkouts)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-8 min:** Warm up cache from database
   - **T+8-10 min:** Switch payment processor to backup
   - **T+10-12 min:** Verify all services healthy in DR region
   - **T+12-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+25 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check database replication lag at failure time
   - Test checkout: Process 1,000 test checkouts, verify >99% success
   - Test payment processing: Verify payment processing works correctly
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify order integrity: Check all orders accessible

5. **Data Integrity Verification:**
   - Compare order count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random orders accessible
   - Check cart state: Verify cart state matches pre-failover
   - Test edge cases: Large orders, payment failures, refunds

6. **Failback Procedure (T+25 to T+30 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via database replication lag)
- ✅ Checkout works: >99% checkouts successful
- ✅ No data loss: Order count matches pre-failover (within 5 min window)
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

### Journey 1: Customer Completes Checkout

**Step-by-step:**

1. **User Action**: Customer adds items to cart, proceeds to checkout via `POST /v1/checkout` with `{"cart_id": "cart_abc123", "shipping_address": {...}, "payment_method_id": "pm_xyz"}`
2. **Checkout Service**: 
   - Validates cart: `GET /v1/carts/cart_abc123` → Returns items
   - Calculates totals (subtotal, tax, shipping)
   - Creates order: `INSERT INTO orders (id, user_id, cart_id, total, status)`
   - Publishes to Kafka: `{"event": "order_created", "order_id": "order_xyz789"}`
3. **Payment Service** (consumes Kafka):
   - Processes payment: `POST /v1/payments` → Charges payment method
   - Receives response: `{"status": "succeeded", "charge_id": "ch_abc123"}`
   - Updates order: `UPDATE orders SET payment_status = 'paid', payment_id = 'ch_abc123'`
4. **Inventory Service** (consumes Kafka):
   - Reserves inventory: `UPDATE inventory SET reserved = reserved + quantity WHERE product_id = ...`
   - Updates order: `UPDATE orders SET inventory_status = 'reserved'`
5. **Response**: `201 Created` with `{"order_id": "order_xyz789", "status": "confirmed"}`
6. **Fulfillment Service** (consumes Kafka):
   - Creates shipment: `INSERT INTO shipments (order_id, status)`
   - Sends to warehouse
7. **Result**: Order placed, payment processed, inventory reserved within 3 seconds

**Total latency: ~3 seconds** (payment 1s + inventory 1s + order creation 1s)

### Journey 2: Cart Abandonment

**Step-by-step:**

1. **User Action**: Customer adds items to cart, views cart multiple times
2. **Cart Service**: 
   - Stores cart in Redis: `SET cart:user123 {items, updated_at} EX 2592000` (30 days)
   - Tracks cart views: `INCR cart_views:user123`
3. **User leaves** (doesn't complete checkout)
4. **Background Job** (runs hourly):
   - Identifies abandoned carts: `SELECT * FROM carts WHERE updated_at < NOW() - INTERVAL '1 hour' AND status = 'active'`
   - Sends reminder email: `POST /v1/notifications` → Email service
5. **Result**: Abandoned cart reminder sent within 1 hour

**Total time: ~1 hour** (background job interval)

### Failure & Recovery Walkthrough

**Scenario: Payment Gateway Failure During Checkout**

**RTO (Recovery Time Objective):** < 5 minutes (gateway recovery + retry)  
**RPO (Recovery Point Objective):** 0 (orders queued, no data loss)

**Timeline:**

```
T+0s:    Payment gateway starts returning 503 errors
T+0-10s: Checkout requests fail
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Orders queued to dead letter queue (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   Payment gateway recovers
T+35s:   Circuit breaker closes (after 3 successful requests)
T+40s:   Queued orders processed from DLQ
T+5min:  All queued orders completed
```

**What degrades:**
- Checkout requests fail for 10-30 seconds
- Orders queued in DLQ (no data loss)
- Cart operations unaffected

**What stays up:**
- Cart operations
- Product browsing
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
- Dead letter queue prevents order loss
- Exponential backoff reduces gateway load
- Timeouts prevent thread exhaustion

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
For checkout services:
- 8 CPU cores per pod
- High transaction rate (checkout requires transactions)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for concurrent checkouts: 30 connections per pod
- 32 pods (across all services) × 30 = 960 max connections
- Database max_connections: 1,200 (20% headroom)
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

---

## 2. Monitoring & Observability

### Key Metrics

**Checkout Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Checkout completion rate | > 30% | < 25% | < 20% |
| Payment success rate | > 98% | < 95% | < 90% |
| Order creation latency (p99) | < 5s | > 7s | > 10s |
| Cart abandonment rate | < 70% | > 75% | > 80% |

**Resource Metrics:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 70% | > 85% |
| Memory | > 80% | > 90% |
| Database connections | > 80% | > 90% |

### Metrics Collection

```java
@Component
public class CheckoutMetrics {
    
    private final Counter ordersCreated;
    private final Counter paymentsProcessed;
    private final Counter paymentsFailed;
    private final Timer checkoutLatency;
    private final Timer paymentLatency;
    
    public CheckoutMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("checkout.orders.created")
            .register(registry);
        
        this.paymentLatency = Timer.builder("checkout.payment.latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public void recordOrderCreation(Order order) {
        ordersCreated.increment();
        checkoutLatency.record(order.getCreationTime(), TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low Checkout Rate | < 25% for 10 min | Warning | Investigate |
| High Payment Failure | > 5% for 5 min | Critical | Page on-call |
| Database Connection Pool Exhausted | > 90% for 2 min | Critical | Scale database |
| Order Creation Failure | > 1% for 5 min | Critical | Page on-call |

---

## 3. Security Considerations

### Payment Data Security

**PCI-DSS Compliance:**
- Never store full credit card numbers
- Encrypt payment data at rest (AES-256)
- Use TLS 1.3 for data in transit
- Tokenize payment methods

**Payment Data Handling:**

```java
@Service
public class PaymentSecurityService {
    
    public PaymentToken tokenizePayment(PaymentMethod paymentMethod) {
        // Send to payment gateway for tokenization
        // Store only token, not actual card number
        String token = paymentGateway.tokenize(paymentMethod);
        
        return PaymentToken.builder()
            .token(token)
            .last4(paymentMethod.getLast4())
            .brand(paymentMethod.getBrand())
            .build();
    }
}
```

### Input Validation

```java
@Service
public class CheckoutValidator {
    
    public ValidationResult validateCheckout(CheckoutRequest request) {
        // Validate cart
        if (request.getCartId() == null) {
            return ValidationResult.error("Cart ID required");
        }
        
        // Validate shipping address
        if (!isValidAddress(request.getShippingAddress())) {
            return ValidationResult.error("Invalid shipping address");
        }
        
        // Validate payment method
        if (!isValidPaymentMethod(request.getPaymentMethod())) {
            return ValidationResult.error("Invalid payment method");
        }
        
        return ValidationResult.success();
    }
}
```

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Payment Fraud** | Fraud detection, velocity checks |
| **Inventory Manipulation** | Server-side validation, distributed locking |
| **Order Tampering** | Immutable orders, audit logging |
| **DDoS** | Rate limiting, WAF |
| **SQL Injection** | Parameterized queries, input sanitization |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Successful Checkout

```
1. User adds items to cart
   Cart Service → Redis: SET cart:cart_123 {...}
   Latency: ~5ms

2. User starts checkout
   Checkout Service → Inventory Service: Reserve inventory
   Inventory Service → PostgreSQL: UPDATE products SET reserved_quantity = ...
   Latency: ~50ms

3. User enters payment info
   Payment Service → Payment Gateway: Process payment
   Payment Gateway: Processes payment (2 seconds)
   Payment Service → PostgreSQL: INSERT INTO payments ...
   Latency: ~2 seconds

4. Order created
   Order Service → PostgreSQL: INSERT INTO orders ...
   Order Service → Kafka: Publish order.created event
   Latency: ~20ms

5. Confirmation sent
   Notification Service → Email Service: Send confirmation
   Latency: ~100ms

Total latency: ~2.2 seconds
```

### Journey 2: Payment Failure Recovery

```
1. User starts checkout (inventory reserved)
   Reservation ID: res_123

2. Payment fails
   Payment Gateway: "Card declined"
   Payment Service: Payment status = 'failed'

3. Compensation triggered
   Saga Orchestrator → Inventory Service: Release reservation res_123
   Inventory Service: UPDATE products SET reserved_quantity = reserved_quantity - ...

4. User retries with different payment method
   Payment succeeds, order created

Total latency: ~3 seconds (including retry)
```

### Failure Scenario: Database Primary Failure

**Detection:**
- Database connection errors in logs
- Health check failures

**Recovery Steps:**
1. T+0s: Alert received, database connection failures detected
2. T+10s: Verify primary database status
3. T+20s: Trigger automatic failover to replica
4. T+30s: Replica promoted to primary
5. T+45s: Applications reconnect to new primary
6. T+60s: Normal operations resume

**RTO:** < 1 minute
**RPO:** 0 (synchronous replication)

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EC2) | $12,450 | 71% |
| Database (RDS) | $3,600 | 21% |
| Redis (ElastiCache) | $2,160 | 12% |
| Storage (S3/EBS) | $1,331 | 8% |
| Network | $800 | 5% |
| **Total** | **~$20,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $20,000
**Target Monthly Cost:** $14,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $10,000/month (50%) - Largest single component
2. **Database (RDS):** $6,000/month (30%) - PostgreSQL database
3. **Cache (Redis):** $2,000/month (10%) - Redis cache

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $4,000/month (40% of $10K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Database Optimization (25% savings):**
   - **Current:** Over-provisioned database
   - **Optimization:** 
     - Connection pooling: Reduce connection overhead
     - Query optimization: Faster queries, less CPU
     - Read replicas: Offload reads from primary
   - **Savings:** $1,500/month (25% of $6K database)
   - **Trade-off:** Slight complexity increase (acceptable)

3. **Cache Optimization (30% savings):**
   - **Current:** 70% cache hit rate
   - **Optimization:** 
     - Increase cache hit rates to 85%
     - Optimize cache key patterns
   - **Savings:** $600/month (30% of $2K cache)
   - **Trade-off:** Slight memory increase (acceptable)

4. **Spot Instances (60% savings for non-critical):**
   - **Current:** On-demand instances for all workloads
   - **Optimization:** Spot instances for non-critical workloads (analytics)
   - **Savings:** $1,200/month (60% of $2K non-critical compute)
   - **Trade-off:** Possible interruptions (acceptable for batch jobs)

5. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $700/month (7% of $10K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $8,000/month (40% reduction)
**Optimized Monthly Cost:** $12,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Checkout | $0.000067 | $0.00004 | 40% |
| Cart Operation | $0.00001 | $0.000006 | 40% |
| Order Query | $0.000005 | $0.000003 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Database Optimization → $5.5K savings
2. **Phase 2 (Month 2):** Cache Optimization, Spot Instances → $1.8K savings
3. **Phase 3 (Month 3):** Right-sizing → $700 savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor checkout success rates (ensure no degradation)
- Monitor cache hit rates (target >85%)
- Review and adjust quarterly

### Cost per Order

```
Monthly orders: 300 million
Monthly cost: $20,000

Cost per order = $20,000 / 300M = $0.000067
= $67 per million orders
```

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | Round-robin + sticky sessions | 6-15 instances per service |
| Rate Limiting | Per-user, per-operation | Redis-based |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | > 98% payment success |
| Security | PCI-DSS compliant | Encrypted payment data |
| DR | Multi-region | RTO < 15 min |
| Cost | $20K/month | $67 per million orders |


