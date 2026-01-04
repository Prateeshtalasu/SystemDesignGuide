# Payment System - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Application Load Balancing:**
- Round-robin for stateless services
- Health check-based routing
- Sticky sessions for stateful services (if needed)

**Database Load Balancing:**
- Primary: All writes
- Read replicas: 70% of reads
- Connection pooling: Max 100 connections per server

### Rate Limiting Implementation

**Per-API-Key Rate Limiting:**

```java
@Service
public class RateLimiterService {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean isAllowed(String apiKey, String endpoint) {
        String key = String.format("rate:%s:%s", apiKey, endpoint);
        Long count = redis.opsForValue().increment(key);
        
        if (count == 1) {
            redis.expire(key, Duration.ofMinutes(1));
        }
        
        int limit = getLimitForTier(apiKey);
        return count <= limit;
    }
}
```

**Rate Limits:**

| Tier | Requests/minute | Burst |
|------|-----------------|-------|
| Free | 100 | 10 |
| Pro | 1,000 | 100 |
| Enterprise | 10,000 | 1,000 |

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Payment Processor API | Yes | Failover to backup processor |
| Fraud Service | Yes | Allow transaction (fail open) |
| Database | Yes | Return cached data |
| Redis | Yes | Fallback to database |

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-processor:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
        halfOpenMaxCalls: 10
      fraud-service:
        slidingWindowSize: 50
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Payment processing | 5s | 1 | None (idempotent) |
| Fraud check | 1s | 2 | 100ms |
| Database write | 2s | 3 | Exponential (100ms, 200ms, 400ms) |
| Payment processor API | 3s | 2 | 500ms |

### Replication and Failover

**Database Replication:**
- Primary: 1 per shard (writes)
- Replicas: 2-3 per shard (reads)
- Synchronous replication for financial data
- Automatic failover (30s RTO)

**Service Replication:**
- Multiple instances per service
- Health check-based routing
- Automatic restart on failure

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Non-critical features (analytics, reporting)
2. **Second**: Fraud checks (fail open, allow transactions)
3. **Third**: Read operations (serve from cache)
4. **Last**: Payment processing (core functionality)

### Backup and Recovery

**RPO (Recovery Point Objective):** 0 (zero data loss)
- Maximum acceptable data loss
- Synchronous replication ensures zero loss
- All transactions persisted immediately

**RTO (Recovery Time Objective):** < 30 seconds
- Maximum acceptable downtime
- Automatic failover to replica
- Includes database promotion and connection updates

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Transaction Logs**: Continuous archiving (WAL)
- **Cross-region Replication**: Async replication to DR region
- **Point-in-time Recovery**: 7-year retention

**Restore Steps:**
1. Detect primary failure (health checks) - 5 seconds
2. Promote replica to primary - 10 seconds
3. Update connection strings - 5 seconds
4. Verify write operations - 5 seconds
5. Resume normal operations - 5 seconds

**Disaster Recovery Testing:**

**Frequency:** Monthly (every month, critical system)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag = 0 (synchronous replication)
- [ ] Payment processor failover configured
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] No active transactions during test window (or minimal)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture transaction success rate
   - Document active transaction count
   - Verify database replication lag (= 0, synchronous)
   - Test payment processing (100 sample transactions)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+30 seconds):**
   - **T+0-5s:** Detect failure via health checks
   - **T+5-10s:** Promote secondary region to primary
   - **T+10-15s:** Update DNS records (Route53 health checks)
   - **T+15-20s:** Switch payment processor to backup
   - **T+20-25s:** Verify all services healthy in DR region
   - **T+25-30s:** Resume traffic to DR region

4. **Post-Failover Validation (T+30s to T+5 minutes):**
   - Verify RTO < 30 seconds: ✅ PASS/FAIL
   - Verify RPO = 0: Check transaction count matches pre-failover (exact match)
   - Test payment processing: Process 1,000 test transactions, verify 100% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify transaction integrity: Check all transactions processed correctly

5. **Data Integrity Verification:**
   - Compare transaction count: Pre-failover vs post-failover (should match exactly, RPO=0)
   - Spot check: Verify 100 random transactions accessible
   - Check ledger balance: Verify ledger balances match pre-failover
   - Test edge cases: Large transactions, refunds, chargebacks

6. **Failback Procedure (T+5 to T+10 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag = 0
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 30 seconds: Time from failure to service resumption
- ✅ RPO = 0: No data loss (verified via transaction count, exact match)
- ✅ Payment processing works: 100% transactions processed successfully
- ✅ No data loss: Transaction count matches pre-failover exactly
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Ledger integrity: All ledger balances correct

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 1 month]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Customer Makes Payment

**Step-by-step:**

1. **User Action**: Customer initiates payment via `POST /v1/payments` with `{"amount": 100, "currency": "USD", "payment_method_id": "pm_123", "order_id": "order_abc"}`
2. **Payment Service**: 
   - Validates request (amount, currency, payment method)
   - Generates transaction ID: `txn_xyz789`
   - Creates transaction record: `INSERT INTO transactions (id, amount, currency, status, payment_method_id)`
   - Publishes to Kafka: `{"event": "payment_initiated", "transaction_id": "txn_xyz789"}`
3. **Fraud Check Service** (consumes Kafka):
   - Fetches user history: `SELECT * FROM transactions WHERE user_id = 'user123'`
   - Runs fraud checks (velocity, amount, location)
   - Updates transaction: `UPDATE transactions SET fraud_score = 0.1, fraud_status = 'approved'`
4. **Payment Processor** (consumes Kafka):
   - Charges payment method via external API: `POST https://processor.com/charge`
   - Receives response: `{"status": "succeeded", "charge_id": "ch_abc123"}`
   - Updates transaction: `UPDATE transactions SET status = 'completed', processor_response = '...'`
5. **Response**: `201 Created` with `{"transaction_id": "txn_xyz789", "status": "completed"}`
6. **Webhook Service** (consumes Kafka):
   - Sends webhook to merchant: `POST https://merchant.com/webhook {"transaction_id": "txn_xyz789", "status": "completed"}`
7. **Result**: Payment processed and merchant notified within 2 seconds

**Total latency: ~2 seconds** (fraud check 500ms + processor 1s + webhook 500ms)

### Journey 2: Payment Refund

**Step-by-step:**

1. **User Action**: Customer requests refund via `POST /v1/refunds` with `{"transaction_id": "txn_xyz789", "amount": 100}`
2. **Refund Service**: 
   - Validates refund (transaction exists, amount valid, not already refunded)
   - Creates refund record: `INSERT INTO refunds (id, transaction_id, amount, status)`
   - Publishes to Kafka: `{"event": "refund_initiated", "refund_id": "ref_abc123"}`
3. **Payment Processor** (consumes Kafka):
   - Processes refund via external API: `POST https://processor.com/refund`
   - Receives response: `{"status": "succeeded", "refund_id": "re_xyz789"}`
   - Updates refund: `UPDATE refunds SET status = 'completed', processor_response = '...'`
4. **Response**: `201 Created` with `{"refund_id": "ref_abc123", "status": "completed"}`
5. **Result**: Refund processed within 1 second

**Total latency: ~1 second** (processor API call)

### Failure & Recovery Walkthrough

**Scenario: Payment Processor API Failure**

**RTO (Recovery Time Objective):** < 5 minutes (automatic retry + fallback processor)  
**RPO (Recovery Point Objective):** 0 (transactions queued, no data loss)

**Timeline:**

```
T+0s:    Payment processor API starts returning 503 errors
T+0-10s: Payment processing fails
T+10s:   Circuit breaker opens (after 5 consecutive failures)
T+10s:   Transactions queued to dead letter queue (Kafka)
T+15s:   Retry with exponential backoff (1s, 2s, 4s, 8s)
T+30s:   Fallback processor activated
T+35s:   Queued transactions routed to fallback processor
T+40s:   Transactions processed via fallback
T+5min:  All queued transactions completed
```

**What degrades:**
- Payment processing delayed by 10-30 seconds
- Transactions queued in DLQ (no data loss)
- Fallback processor may have higher fees

**What stays up:**
- Transaction creation and storage
- Fraud checks
- Webhook delivery
- All read operations

**What recovers automatically:**
- Circuit breaker closes after processor recovery
- Queued transactions processed automatically
- Fallback processor handles overflow
- No manual intervention required

**What requires human intervention:**
- Investigate processor API root cause
- Review capacity planning
- Consider alternative processors

**Cascading failure prevention:**
- Circuit breaker prevents retry storms
- Dead letter queue prevents transaction loss
- Exponential backoff reduces processor load
- Fallback processor ensures availability

### Disaster Recovery

**Multi-Region:**
- Primary: us-east-1
- DR: us-west-2

**Replication:**
- Synchronous: Financial transactions (zero loss)
- Asynchronous: Analytics, logs (acceptable loss)

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
For payment service:
- 8 CPU cores per pod
- High transaction rate (payment processing requires transactions)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for concurrent payments: 30 connections per pod
- 50 payment service pods × 30 = 1,500 max connections to database
- Database max_connections: 2,000 (20% headroom)
```

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 50             # Max connections per instance
      max-idle: 25                # Max idle connections
      min-idle: 10                # Min idle connections
      
    # Timeouts
    connection-timeout: 2000      # 2s - Max time to get connection
    socket-timeout: 3000          # 3s - Max time for operation
```

**Pool Sizing Calculation:**
```
For payment cache:
- 8 CPU cores per pod
- High read/write rate (payment status, rate limiting)
- Calculated: 50 connections per pod
- 50 payment service pods × 50 = 2,500 max connections
- Redis cluster: 6 nodes, 500 connections per node capacity
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

---

## 2. Monitoring & Observability

### Key Metrics

**Payment Metrics:**

| Metric | Target | Warning | Critical | How to Measure |
|--------|--------|---------|----------|----------------|
| Payment success rate | > 99% | < 98% | < 95% | Successful / Total attempts |
| Payment latency p50 | < 500ms | > 700ms | > 1s | Time from request to response |
| Payment latency p95 | < 2s | > 3s | > 5s | 95th percentile latency |
| Payment latency p99 | < 5s | > 7s | > 10s | 99th percentile latency |
| Fraud rate | < 0.1% | > 0.2% | > 0.5% | Fraudulent / Total transactions |
| Chargeback rate | < 0.5% | > 0.7% | > 1% | Chargebacks / Total transactions |

**System Metrics:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 70% | > 85% |
| Memory | > 80% | > 90% |
| Database connections | > 80% | > 90% |
| Error rate | > 1% | > 5% |

### Metrics Collection

```java
@Component
public class PaymentMetrics {
    
    private final MeterRegistry registry;
    
    private final Counter paymentsTotal;
    private final Counter paymentsSucceeded;
    private final Counter paymentsFailed;
    private final Timer paymentLatency;
    private final Counter fraudDetected;
    
    public PaymentMetrics(MeterRegistry registry) {
        this.paymentsTotal = Counter.builder("payments.total")
            .register(registry);
        
        this.paymentLatency = Timer.builder("payments.latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.fraudDetected = Counter.builder("fraud.detected")
            .register(registry);
    }
    
    public void recordPayment(PaymentResult result) {
        paymentsTotal.increment();
        if (result.isSuccess()) {
            paymentsSucceeded.increment();
        } else {
            paymentsFailed.increment();
        }
        paymentLatency.record(result.getDuration(), TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low payment success rate | < 98% for 5 min | Warning | Investigate |
| Very low success rate | < 95% for 2 min | Critical | Page on-call |
| High payment latency | p95 > 3s for 5 min | Warning | Check processors |
| Very high latency | p95 > 5s for 2 min | Critical | Page on-call |
| High fraud rate | > 0.2% for 10 min | Warning | Review fraud rules |
| Database primary down | 100% failure | Critical | Automatic failover |
| Payment processor down | 100% failure | Critical | Failover to backup |

### Health Checks

```java
@Component
public class PaymentHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check database connectivity
        boolean dbHealthy = databaseHealthCheck();
        details.put("database", dbHealthy ? "UP" : "DOWN");
        
        // Check Redis connectivity
        boolean redisHealthy = redisHealthCheck();
        details.put("redis", redisHealthy ? "UP" : "DOWN");
        
        // Check payment processor
        boolean processorHealthy = processorHealthCheck();
        details.put("payment_processor", processorHealthy ? "UP" : "DOWN");
        
        // Check payment success rate
        double successRate = metrics.getPaymentSuccessRate();
        details.put("payment_success_rate", successRate);
        
        if (!dbHealthy || !redisHealthy) {
            return Health.down().withDetails(details).build();
        }
        
        if (successRate < 0.95) {
            return Health.down()
                .withDetails(details)
                .withDetail("error", "Payment success rate too low")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

### Logging Strategy

**Structured Logging:**
```java
log.info("Payment processed", 
    "transaction_id", transactionId,
    "amount", amount,
    "status", status,
    "latency_ms", latency,
    "fraud_score", fraudScore
);
```

**Log Levels:**
- **ERROR**: Payment failures, system errors
- **WARN**: High fraud scores, slow operations
- **INFO**: Payment processed, key operations
- **DEBUG**: Detailed flow, intermediate steps

**PII Redaction:**
- Card numbers: Never logged
- Payment method IDs: Allowed (tokenized)
- Customer emails: Redacted in logs
- Amounts: Allowed (necessary for audit)

### Distributed Tracing

**Trace IDs:**
- Generated at API gateway
- Propagated through all services
- Included in all logs

**Span Boundaries:**
- Payment processing span
- Fraud check span
- Ledger write span
- Processor API call span

---

## 3. Security Considerations

### Authentication Mechanism

**API Key Authentication:**
```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response,
                                   FilterChain filterChain) {
        String apiKey = request.getHeader("Authorization");
        
        if (apiKey == null || !apiKey.startsWith("Bearer ")) {
            response.setStatus(401);
            return;
        }
        
        apiKey = apiKey.substring(7);
        ApiKey key = apiKeyRepository.findByKey(apiKey);
        
        if (key == null || !key.isActive()) {
            response.setStatus(401);
            return;
        }
        
        SecurityContextHolder.getContext().setAuthentication(
            new ApiKeyAuthentication(key)
        );
        
        filterChain.doFilter(request, response);
    }
}
```

### Authorization and Access Control

**Role-Based Access Control (RBAC):**
- **Admin**: Full access
- **Merchant**: Own transactions only
- **Read-only**: View transactions, no modifications

### Data Encryption

**At Rest:**
- Database: AES-256 encryption
- Payment methods: Tokenized (no full card numbers stored)
- Ledger entries: Encrypted sensitive fields

**In Transit:**
- TLS 1.3 for all API communication
- Mutual TLS for service-to-service communication
- Certificate pinning for mobile apps

### Input Validation and Sanitization

**Amount Validation:**
```java
public boolean isValidAmount(BigDecimal amount) {
    return amount != null &&
           amount.compareTo(BigDecimal.ZERO) > 0 &&
           amount.compareTo(new BigDecimal("1000000")) <= 0;  // Max $1M
}
```

**Payment Method Validation:**
```java
public boolean isValidPaymentMethod(String paymentMethodId) {
    // Check format
    if (!paymentMethodId.matches("^pm_[a-zA-Z0-9]+$")) {
        return false;
    }
    
    // Check exists and belongs to user
    PaymentMethod pm = paymentMethodRepository.findById(paymentMethodId);
    return pm != null && pm.getUserId().equals(currentUserId);
}
```

### PCI DSS Compliance

**Requirements:**
1. **Never store full card numbers**
2. **Tokenize all card data**
3. **Encrypt sensitive data at rest**
4. **Use strong encryption in transit**
5. **Regular security audits**
6. **Access controls and logging**

**Implementation:**
- Payment methods stored as tokens (e.g., "pm_abc123")
- Full card numbers never touch our database
- Card data encrypted by payment processor
- Access logs for all sensitive operations

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Card Data Theft** | Tokenization, never store full numbers |
| **Duplicate Charges** | Idempotency keys, database constraints |
| **Fraud** | ML models, rule-based detection |
| **SQL Injection** | Parameterized queries, input validation |
| **Replay Attacks** | Idempotency keys, nonce validation |
| **Man-in-the-Middle** | TLS 1.3, certificate pinning |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Successful Payment

```
1. User initiates payment: $100 for subscription
   ↓
2. Client sends request with Idempotency-Key
   ↓
3. API Gateway: Authenticate, rate limit
   ↓
4. Payment Service: Check idempotency (Redis) → MISS
   ↓
5. Validate request: Amount, payment method
   ↓
6. Fraud Service: Check transaction
   - Rule checks: Pass
   - ML model: Score 0.15 (low risk)
   - Decision: ALLOW
   ↓
7. Payment Processor: Charge payment method
   - Stripe API: Charge $100
   - Response: Success, transaction_id: "ch_xyz"
   ↓
8. Store transaction in database
   - Status: succeeded
   - Processor ID: "ch_xyz"
   ↓
9. Ledger Service: Record double-entry
   - Debit: Customer account (+$100)
   - Credit: Revenue account (+$100)
   - Verify: Balanced
   ↓
10. Cache result in Redis (idempotency)
   ↓
11. Return success to client
   ↓
12. Send webhook to merchant (async)
```

### Journey 2: Payment with Fraud Detection

```
1. User initiates payment: $5,000 for purchase
   ↓
2. Payment Service: Check idempotency → MISS
   ↓
3. Fraud Service: Check transaction
   - Rule check 1: Amount > $1,000 → Flagged
   - Rule check 2: New payment method (< 24h) → Flagged
   - Rule check 3: Unusual location → Flagged
   - Rule score: 0.85 (high risk)
   ↓
4. ML Model: Score transaction
   - Features: Amount, location, device, history
   - Score: 0.78 (high fraud probability)
   ↓
5. Combined score: 0.82 (high risk)
   ↓
6. Decision: BLOCK transaction
   ↓
7. Store blocked transaction
   - Status: blocked
   - Reason: "High fraud risk"
   ↓
8. Return error to client
   - Code: FRAUD_DETECTED
   - Message: "Transaction blocked for security"
   ↓
9. Alert fraud team (async)
   - Manual review queue
```

### Journey 3: Refund Processing

```
1. Merchant requests refund: $50 (partial)
   ↓
2. Validate refund request
   - Original payment exists
   - Refund amount <= original amount
   - Refund not already processed
   ↓
3. Check idempotency → MISS
   ↓
4. Payment Processor: Process refund
   - Stripe API: Refund $50
   - Response: Success, refund_id: "re_xyz"
   ↓
5. Store refund in database
   - Status: succeeded
   - Original payment linked
   ↓
6. Ledger Service: Record double-entry
   - Debit: Revenue account (-$50)
   - Credit: Customer account (-$50)
   - Verify: Balanced
   ↓
7. Update original payment status
   - Status: partially_refunded
   ↓
8. Cache refund result
   ↓
9. Return success to merchant
   ↓
10. Send webhook to merchant (async)
```

### Failure & Recovery

**Scenario: Payment Processor Down**

**RTO (Recovery Time Objective):** < 1 minute (failover to backup)  
**RPO (Recovery Point Objective):** 0 (no data loss, transactions queued)

**Runbook Steps:**

1. **Detection (T+0s to T+10s):**
   - Monitor: Circuit breaker opens after 50% failure rate
   - Alert: PagerDuty alert triggered at T+10s
   - Verify: Check payment processor status page
   - Confirm: Processor outage confirmed

2. **Immediate Response (T+10s to T+20s):**
   - Action: Circuit breaker opens, requests fail fast
   - Fallback: Automatic failover to backup processor (PayPal)
   - Impact: Slight latency increase (different processor)
   - Monitor: Verify backup processor healthy

3. **Recovery (T+20s to T+60s):**
   - T+20s: Failover to PayPal complete
   - T+30s: New transactions processed via PayPal
   - T+40s: Queued transactions retried
   - T+60s: Normal operations resumed

4. **Post-Recovery (T+60s to T+300s):**
   - Monitor: Watch payment success rate
   - Verify: All transactions processing normally
   - Document: Log incident, update runbook
   - Follow-up: Investigate primary processor outage

**Prevention:**
- Multiple payment processors (redundancy)
- Circuit breakers (fail fast)
- Health checks every 10 seconds
- Automatic failover

### Failure Scenario: Database Primary Failure

**Detection:**
- How to detect: Database connection errors in logs
- Alert thresholds: > 10% error rate for 30 seconds
- Monitoring: PostgreSQL primary health check endpoint

**Impact:**
- Affected services: All write operations
- User impact: Payment processing fails
- Degradation: Read-only mode (serve cached data)

**Recovery Steps:**
1. **T+0s**: Alert received, database connection failures detected
2. **T+10s**: Verify primary database status (AWS RDS console)
3. **T+20s**: Confirm primary failure (not just network issue)
4. **T+30s**: Trigger automatic failover to synchronous replica
5. **T+45s**: Replica promoted to primary
6. **T+60s**: Update application connection strings
7. **T+90s**: Applications reconnect to new primary
8. **T+120s**: Verify write operations successful
9. **T+150s**: Normal operations resume

**RTO:** < 3 minutes
**RPO:** 0 (synchronous replication)

**Prevention:**
- Multi-AZ deployment with synchronous replication
- Automated failover enabled
- Regular failover testing (monthly)

### Failure Scenario: Redis Cluster Failure

**Detection:**
- How to detect: Grafana alert "Redis cluster down"
- Alert thresholds: 3 consecutive health check failures
- Monitoring: Check Redis cluster status endpoint

**Impact:**
- Affected services: Idempotency checks, caching
- User impact: Payment processing continues (degraded)
- Degradation: Slower idempotency checks (database fallback)

**Recovery Steps:**
1. **T+0s**: Alert received, on-call engineer paged
2. **T+30s**: Verify Redis cluster status via AWS console
3. **T+60s**: Check Redis primary node health
4. **T+90s**: If primary down, promote replica to primary (automatic)
5. **T+120s**: Verify Redis cluster healthy
6. **T+150s**: Applications reconnect to Redis
7. **T+180s**: Cache begins repopulating

**RTO:** < 3 minutes
**RPO:** 0 (no data loss, idempotency in database)

**Prevention:**
- Redis cluster with automatic failover
- Health checks every 10 seconds
- Fallback to database for critical operations

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total | Notes |
|-----------|--------------|------------|-------|
| Payment Processor Fees | $95,200,000 | 79.5% | 2.9% + $0.30 per transaction (enterprise pricing) |
| Compute (EC2) | $15,810 | 0.01% | Payment, fraud, ledger services |
| Storage (EBS) | $24,300 | 0.02% | 243 TB with replication |
| Database (RDS) | $4,500 | 0.004% | PostgreSQL multi-AZ |
| Redis Cluster | $2,160 | 0.002% | Idempotency, caching |
| Network | $500 | <0.001% | Data transfer |
| Monitoring | $1,000 | <0.001% | Prometheus, Grafana |
| Security | $2,000 | <0.001% | WAF, DDoS protection |
| Misc (20%) | $19,040,000 | 15.9% | Support, compliance, overhead |
| **Total** | **~$119.8M** | 100% | |

### Cost Optimization Strategies

**Current Monthly Cost:** $120,000,000
**Target Monthly Cost:** $90,000,000 (25% reduction)

**Top 3 Cost Drivers:**
1. **Payment Processor Fees:** $100,000,000/month (83%) - Largest single component (external fees)
2. **Compute (EC2):** $10,000,000/month (8%) - Application servers
3. **Database (RDS):** $5,000,000/month (4%) - PostgreSQL database

**Note:** The majority of costs are external payment processor fees, which are harder to optimize directly.

**Optimization Strategies (Ranked by Impact):**

1. **Payment Processor Negotiation (25% savings on fees):**
   - **Current:** Standard pricing
   - **Optimization:** 
     - Enterprise pricing: 20-30% discount
     - Volume discounts: Additional 5-10% at scale
   - **Savings:** $25,000,000/month (25% of $100M processor fees)
   - **Trade-off:** Requires negotiation and volume commitments

2. **Compute Optimization (40% savings):**
   - **Current:** On-demand instances
   - **Optimization:** 
     - Reserved instances: 40% savings on baseline
     - Spot instances: 60% savings (for non-critical workloads)
     - Auto-scaling: Scale down during low traffic
   - **Savings:** $4,000,000/month (40% of $10M compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

3. **Storage Optimization (30% savings):**
   - **Current:** All data in standard storage
   - **Optimization:** 
     - Data compression: 30% reduction
     - Tiered storage: Move old data to cheaper tiers
     - Data archival: Archive data > 1 year old
   - **Savings:** $900,000/month (30% of $3M storage)
   - **Trade-off:** Slower access to archived data (acceptable)

4. **Database Optimization (20% savings):**
   - **Current:** Over-provisioned database
   - **Optimization:** 
     - Connection pooling: Reduce connection overhead
     - Query optimization: Faster queries, less CPU
     - Read replicas: Offload reads from primary
   - **Savings:** $1,000,000/month (20% of $5M database)
   - **Trade-off:** Slight complexity increase (acceptable)

**Total Potential Savings:** $30,900,000/month (26% reduction)
**Optimized Monthly Cost:** $89,100,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Payment Processing | $0.40 | $0.30 | 25% |
| Transaction Storage | $0.0001 | $0.00007 | 30% |
| Database Query | $0.00001 | $0.000008 | 20% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Payment Processor Negotiation → $25M savings
2. **Phase 2 (Month 2):** Compute Optimization → $4M savings
3. **Phase 3 (Month 3):** Storage & Database Optimization → $1.9M savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor payment processor fees (target < $75M/month)
- Monitor transaction success rates (ensure no degradation)
- Review and adjust quarterly

**Note:** The largest cost optimization opportunity is payment processor fee negotiation, which can save $25M/month.

### Cost per Transaction

**Calculation:**
```
Monthly transactions: 300 million
Monthly cost: $119.8M (with enterprise pricing)

Cost per transaction = $119.8M / 300M
                     = $0.40 per transaction

Breakdown:
- Processor fees: $0.317 per transaction
- Infrastructure: $0.00005 per transaction
- Overhead: $0.083 per transaction
```

### Scaling Cost Projections

**10x Scale (1B transactions/month):**
- Processor fees: $952M/month (linear)
- Infrastructure: $200K/month (sub-linear, economies of scale)
- Total: ~$952M/month

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | Application + Database | Round-robin, health checks |
| Rate Limiting | Per API key, Redis | 1,000 req/min (pro tier) |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | > 99% success rate |
| Security | PCI DSS Level 1 | Zero breaches |
| DR | Multi-region | RTO < 30s, RPO = 0 |
| Cost | $119.8M/month | $0.40 per transaction |
