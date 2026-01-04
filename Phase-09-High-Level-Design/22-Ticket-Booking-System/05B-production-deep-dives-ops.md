# Ticket Booking System - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

- Round-robin for stateless services
- Sticky sessions for seat locking (maintain lock state)
- Health check-based routing

### Rate Limiting Implementation

**Per-User Rate Limiting:**

```java
@Service
public class BookingRateLimiter {
    
    public boolean isAllowed(String userId, String operation) {
        String key = "rate_limit:" + userId + ":" + operation;
        // Check rate limit in Redis
        // Limits: seat_selection: 60/min, booking: 30/min
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Payment Gateway | Yes | Queue payment for retry |
| Seat Service | Yes | Reject seat selection |
| Booking Service | Yes | Queue booking creation |

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Seat Locking | 2s | 2 | 100ms, 500ms |
| Payment Processing | 10s | 2 | 1s, 3s |
| Booking Creation | 5s | 2 | 1s, 2s |

### Replication and Failover

- Multi-AZ deployment (3 availability zones)
- PostgreSQL primary-replica with automatic failover
- Redis cluster with automatic failover

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Seat recommendations
2. **Second**: Event search (can be slower)
3. **Third**: Booking history queries
4. **Last**: Stop accepting new bookings

### Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes
- Maximum acceptable data loss
- Based on database replication lag (5 minutes max)
- Booking data replicated to read replicas within 5 minutes
- Seat locks stored in Redis with persistence

**RTO (Recovery Time Objective):** < 15 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Hourly incremental, daily full
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Redis Snapshots**: Hourly snapshots of seat locks and booking state

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database replica to primary - 2 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 5 minutes
5. Verify booking service health and resume traffic - 6 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 5 minutes
- [ ] Redis replication lag < 5 minutes
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available
- [ ] No active bookings during test window (or minimal)

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture booking success rate
   - Document active booking/seat lock count
   - Verify database replication lag (< 5 minutes)
   - Test booking flow (100 sample bookings)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-8 min:** Warm up Redis cache from database
   - **T+8-10 min:** Verify all services healthy in DR region
   - **T+10-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+25 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check database replication lag at failure time
   - Test booking: Process 1,000 test bookings, verify >99% success
   - Test seat locking: Verify seat locking works correctly
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify booking integrity: Check all bookings accessible

5. **Data Integrity Verification:**
   - Compare booking count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random bookings accessible
   - Check seat locks: Verify seat locks match pre-failover
   - Test edge cases: High-demand events, concurrent bookings, payment failures

6. **Failback Procedure (T+25 to T+30 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via database replication lag)
- ✅ Booking works: >99% bookings successful
- ✅ No data loss: Booking count matches pre-failover (within 5 min window)
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

### Journey 1: User Books Tickets

**Step-by-step:**

1. **User Action**: User views seat map via `GET /v1/events/event_123/shows/show_456/seats`
2. **Seat Service**: 
   - Fetches seat availability from Redis: `GET seats:show_456` → Returns available seats
   - Returns seat map to client
3. **Response**: `200 OK` with seat map
4. **User Action**: User selects seats, locks them via `POST /v1/bookings/lock` with `{"show_id": "show_456", "seat_ids": ["A1", "A2"]}`
5. **Booking Service**: 
   - Locks seats in Redis: `SET seat_lock:show_456:A1 {user_id, expires_at} EX 300` (5 min)
   - Locks seats in database: `UPDATE seats SET status = 'locked', locked_by = 'user123', locked_until = NOW() + INTERVAL '5 minutes'`
6. **Response**: `201 Created` with `{"lock_id": "lock_xyz789", "expires_at": "..."}`
7. **User Action**: User completes booking via `POST /v1/bookings` with `{"lock_id": "lock_xyz789", "payment_method_id": "pm_abc"}`
8. **Booking Service**: 
   - Validates lock (still valid, not expired)
   - Creates booking: `INSERT INTO bookings (id, user_id, show_id, seat_ids, status)`
   - Processes payment: `POST /v1/payments` → Charges payment method
   - Confirms seats: `UPDATE seats SET status = 'booked', booking_id = 'booking_abc123'`
   - Releases lock: `DEL seat_lock:show_456:A1`
9. **Response**: `201 Created` with `{"booking_id": "booking_abc123", "status": "confirmed"}`
10. **Result**: Tickets booked, payment processed within 2 seconds

**Total latency: ~2 seconds** (payment 1s + booking creation 1s)

### Journey 2: Seat Lock Expiration

**Step-by-step:**

1. **User Action**: User locks seats but doesn't complete booking
2. **Lock expires** (5 minutes later):
   - Redis TTL expires: `seat_lock:show_456:A1` deleted
   - Background job detects expired locks: `SELECT * FROM seats WHERE status = 'locked' AND locked_until < NOW()`
   - Releases locks: `UPDATE seats SET status = 'available', locked_by = NULL, locked_until = NULL`
3. **Seats available** for other users
4. **Result**: Seats released automatically after 5 minutes

**Total time: 5 minutes** (lock TTL)

### Failure & Recovery Walkthrough

**Scenario: Database Failure During Booking**

**RTO (Recovery Time Objective):** < 15 minutes (database failover + application recovery)  
**RPO (Recovery Point Objective):** < 5 minutes (async replication lag)

**Timeline:**

```
T+0s:    Primary database crashes
T+0-5s:  Booking requests fail
T+5s:    Health checks fail
T+10s:   Replica promoted to primary
T+15s:   Application reconnects to new primary
T+20s:   Booking requests resume
T+30s:   All operations normal
```

**What degrades:**
- Booking requests fail for 15-20 seconds
- Seat locks may expire during failover
- No data loss (async replication)

**What stays up:**
- Seat map viewing (Redis cache)
- User authentication
- All read operations (from replicas)

**What recovers automatically:**
- Database failover to replica
- Application reconnects automatically
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of database failure
- Replace failed hardware
- Review database capacity

**Cascading failure prevention:**
- Database replication prevents data loss
- Circuit breakers prevent retry storms
- Seat lock expiration prevents deadlocks
- Timeouts prevent thread exhaustion

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
For booking services:
- 8 CPU cores per pod
- High transaction rate (seat locking, booking creation)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin: 25 connections per pod
- 40 pods × 25 = 1,000 max connections to database
- Database max_connections: 1,200 (20% headroom)
```

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 30             # Max connections per instance
      max-idle: 15                # Max idle connections
      min-idle: 5                 # Min idle connections
      
    # Timeouts
    connection-timeout: 2000      # 2s - Max time to get connection
    socket-timeout: 3000          # 3s - Max time for operation
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

---

## 2. Monitoring & Observability

### Key Metrics

**Booking Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Booking completion rate | > 40% | < 35% | < 30% |
| Seat locking latency (p99) | < 100ms | > 150ms | > 200ms |
| Double booking incidents | 0 | > 0 | > 0 |
| Payment success rate | > 98% | < 95% | < 90% |

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High Double Booking Rate | > 0 for 1 min | Critical | Page on-call |
| Low Booking Completion | < 35% for 10 min | Warning | Investigate |
| Seat Lock Failures | > 5% for 5 min | Critical | Page on-call |

---

## 3. Security Considerations

### Payment Data Security

- PCI-DSS compliance
- Tokenize payment methods
- Encrypt payment data at rest (AES-256)
- TLS 1.3 for data in transit

### Ticket Security

- QR code encryption
- Ticket validation on entry
- Prevent ticket duplication

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Double Booking** | Distributed locking, database constraints |
| **Ticket Fraud** | QR code encryption, validation |
| **DDoS** | Rate limiting, WAF |
| **Seat Hoarding** | Lock timeout (5 minutes) |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Successful Booking

```
1. User views seat map
   Seat Service → Redis: GET seat_map:show_123
   Latency: ~5ms

2. User selects seats
   Seat Service → Redis: SET seat_lock:show_123:seat_1 user_789 EX 300
   Latency: ~10ms

3. User creates booking
   Booking Service → Payment Service: Process payment
   Payment Gateway: Processes payment (2 seconds)
   Latency: ~2 seconds

4. Booking confirmed
   Booking Service → PostgreSQL: INSERT INTO bookings
   Booking Service → Redis: Remove seat locks
   Latency: ~20ms

5. Ticket generated
   Ticket Service: Generate QR code
   Notification Service: Send ticket email
   Latency: ~100ms

Total latency: ~2.1 seconds
```

### Journey 2: Concurrent Seat Selection

```
Time T+0ms: User A selects seat_1
  Seat Service A: Acquires lock
  Redis: SET seat_lock:show_123:seat_1 user_A EX 300
  Result: OK

Time T+10ms: User B selects seat_1
  Seat Service B: Tries to acquire lock
  Redis: GET seat_lock:show_123:seat_1
  Result: "user_A" (already locked)
  Service B: Returns error "Seat unavailable"

Result: User A gets seat, User B receives error
No double booking
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EC2) | $12,330 | 67% |
| Database (RDS) | $3,600 | 20% |
| Redis (ElastiCache) | $2,160 | 12% |
| Storage | $500 | 3% |
| Network | $2,500 | 14% |
| **Total** | **~$18,400** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $18,400
**Target Monthly Cost:** $12,880 (30% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $12,330/month (67%) - Largest single component
2. **Database (RDS):** $3,600/month (20%) - PostgreSQL database
3. **Network (Data Transfer):** $2,500/month (14%) - High data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $4,932/month (40% of $12.33K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Database Optimization (25% savings):**
   - **Current:** Over-provisioned database
   - **Optimization:** 
     - Connection pooling: Reduce connection overhead
     - Query optimization: Faster queries, less CPU
     - Read replicas: Offload reads from primary
   - **Savings:** $900/month (25% of $3.6K database)
   - **Trade-off:** Slight complexity increase (acceptable)

3. **Network Optimization (30% savings):**
   - **Current:** High data transfer costs
   - **Optimization:** 
     - Regional deployment (route users to nearest region)
     - Compress data before transfer
   - **Savings:** $750/month (30% of $2.5K network)
   - **Trade-off:** Slight complexity increase (acceptable)

4. **Redis Optimization (20% savings):**
   - **Current:** Over-provisioned Redis
   - **Optimization:** 
     - Right-size Redis instances
     - Optimize TTLs
   - **Savings:** $432/month (20% of $2.16K Redis)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $7,014/month (38% reduction)
**Optimized Monthly Cost:** $11,386/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Booking | $0.00061 | $0.00038 | 38% |
| Seat Lock | $0.0001 | $0.00006 | 40% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Database Optimization → $5.83K savings
2. **Phase 2 (Month 2):** Network Optimization, Redis Optimization → $1.18K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor booking success rates (ensure no degradation)
- Monitor database performance (target < 100ms p99)
- Review and adjust quarterly

### Cost per Booking

```
Monthly bookings: 30 million
Monthly cost: $18,400

Cost per booking = $18,400 / 30M = $0.00061
= $610 per million bookings
```

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | Round-robin + sticky sessions | 8-15 instances per service |
| Rate Limiting | Per-user, per-operation | Redis-based |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | > 40% booking completion |
| Security | PCI-DSS compliant | Encrypted payment data |
| DR | Multi-region | RTO < 15 min |
| Cost | $18.4K/month | $610 per million bookings |

