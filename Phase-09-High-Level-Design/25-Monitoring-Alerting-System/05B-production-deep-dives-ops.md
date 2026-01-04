# Monitoring & Alerting System - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Collector Distribution:**

DNS round-robin load balancing:

```
Services → DNS (collector.monitoring.internal) → Round-robin → Collector 1, 2, ..., 100
```

**Query Service Distribution:**

```
Users → Load Balancer → Query Service 1, 2, ..., 50
```

### Rate Limiting Implementation

**Per-Service Rate Limiting:**

```java
@Service
public class MetricRateLimiter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    public void acquire(String serviceName) {
        RateLimiter limiter = limiters.computeIfAbsent(serviceName, 
            s -> RateLimiter.create(getRate(s)));
        limiter.acquire();
    }
    
    private double getRate(String serviceName) {
        return configService.getRateLimit(serviceName);  // Default: 100K metrics/second
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Metric Ingestion | Yes | Buffer locally, retry |
| Time-Series DB Write | Yes | Queue metrics, retry |
| Redis Cache | Yes | Direct to DB |
| Query Service | Yes | Return cached results |

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Metric Ingestion | 100ms | 2 | 10ms, 50ms |
| Query Execution | 5s | 1 | N/A |
| Alert Evaluation | 30s | 0 | N/A |
| Cache Lookup | 10ms | 0 | N/A |

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Reduce metric retention (30 days → 7 days)
2. **Second**: Disable query caching (increase DB load)
3. **Third**: Reduce alert evaluation frequency (15s → 60s)
4. **Last**: Stop collecting metrics (unacceptable)

### Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes
- Maximum acceptable data loss
- Based on time-series database replication lag (5 minutes max)
- Metrics replicated to read replicas within 5 minutes
- Alert state stored in Redis with persistence

**RTO (Recovery Time Objective):** < 10 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Time-Series DB Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Redis Snapshots**: Hourly snapshots of alert state

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database replica to primary - 3 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 2 minutes
5. Verify monitoring service health and resume traffic - 3 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Time-series DB replication lag < 5 minutes
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture metric collection success rate
   - Document active metric count
   - Verify time-series DB replication lag (< 5 minutes)
   - Test metric collection (100 sample metrics)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+10 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-7 min:** Warm up cache from time-series DB
   - **T+7-8 min:** Verify all services healthy in DR region
   - **T+8-10 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+10 to T+20 minutes):**
   - Verify RTO < 10 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check time-series DB replication lag at failure time
   - Test metric collection: Collect 1,000 test metrics, verify >99% success
   - Test alerting: Verify alerting works correctly
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify metric integrity: Check metrics accessible

5. **Data Integrity Verification:**
   - Compare metric count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random metrics accessible
   - Check alert rules: Verify alert rules intact
   - Test edge cases: High-cardinality metrics, complex queries

6. **Failback Procedure (T+20 to T+25 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 10 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via time-series DB replication lag)
- ✅ Metric collection works: >99% metrics collected successfully
- ✅ No critical metrics lost: Metric count matches pre-failover (within 5 min window)
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

### Journey 1: Metric Ingestion and Alert Trigger

**Step-by-step:**

1. **Service emits metric**: User service sends `POST /v1/metrics` with `{"name": "api.latency", "value": 500, "labels": {"endpoint": "/users"}}`
2. **Metrics Collector**: 
   - Receives metric
   - Validates format
   - Publishes to Kafka: `{"metric": "api.latency", "value": 500, "timestamp": "..."}`
3. **Metrics Processor** (consumes Kafka):
   - Aggregates metrics: Calculates 1-minute average
   - Stores in time-series DB: `INSERT INTO metrics (name, value, timestamp)`
4. **Alert Evaluator** (runs every 30 seconds):
   - Queries time-series DB: `SELECT avg(value) FROM metrics WHERE name = 'api.latency' AND time > NOW() - INTERVAL '5 minutes'`
   - Evaluates rule: `avg(api.latency) > 400` → true (500 > 400)
   - Triggers alert: `INSERT INTO alerts (id, rule_id, status, triggered_at)`
   - Publishes to Kafka: `{"event": "alert_triggered", "alert_id": "alert_xyz789"}`
5. **Notification Service** (consumes Kafka):
   - Sends notification: Email to on-call engineer
   - Updates alert: `UPDATE alerts SET notified = true`
6. **Engineer receives alert** (within 30 seconds):
   - Views dashboard: `GET /v1/dashboards/api-latency`
   - Sees latency spike
   - Investigates root cause
7. **Result**: Alert triggered and notified within 1 minute

**Total latency: Metric ingestion ~100ms, Alert evaluation ~30s, Notification ~10s**

### Journey 2: Alert Resolution

**Step-by-step:**

1. **Engineer investigates**: Views metrics, identifies issue, applies fix
2. **Metric returns to normal**: `api.latency` drops to 200ms
3. **Alert Evaluator** (next evaluation):
   - Queries: `SELECT avg(value) FROM metrics WHERE name = 'api.latency' AND time > NOW() - INTERVAL '5 minutes'`
   - Evaluates: `avg(api.latency) > 400` → false (200 < 400)
   - Resolves alert: `UPDATE alerts SET status = 'resolved', resolved_at = NOW()`
4. **Notification Service**:
   - Sends resolution notification: Email to engineer
5. **Result**: Alert resolved and notified within 1 minute

**Total time: Issue fixed → Alert resolved ~1 minute**

### Failure & Recovery Walkthrough

**Scenario: Time-Series Database Failure**

**RTO (Recovery Time Objective):** < 15 minutes (database failover + metric replay)  
**RPO (Recovery Point Objective):** < 5 minutes (metrics buffered in Kafka, no data loss)

**Timeline:**

```
T+0s:    Time-series database crashes
T+0-5s:  Metric writes fail
T+5s:    Metrics buffered in Kafka (retention 7 days)
T+10s:   Health check fails
T+15s:   Database failover to replica
T+20s:   Application reconnects to new primary
T+25s:   Metric writes resume
T+30s:   Kafka replay begins (replay buffered metrics)
T+5min:  All metrics replayed
T+15min: All operations normal
```

**What degrades:**
- Metric writes fail for 20-25 seconds
- Metrics buffered in Kafka (no data loss)
- Alert evaluation delayed (no new metrics)

**What stays up:**
- Metric ingestion (buffered in Kafka)
- Alert queries (reads from replicas)
- All read operations

**What recovers automatically:**
- Database failover to replica
- Application reconnects automatically
- Kafka replay processes buffered metrics
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of database failure
- Replace failed hardware
- Review database capacity

**Cascading failure prevention:**
- Kafka buffering prevents metric loss
- Database replication prevents data loss
- Circuit breakers prevent retry storms
- Timeouts prevent thread exhaustion

### Database Connection Pool Configuration

**Time-Series Database Connection Pool:**

```yaml
timeseries:
  client:
    # Connection pool sizing
    max-connections: 40          # Max connections per application instance
    max-connections-per-route: 10 # Max connections per DB node
    
    # Timeouts
    connection-timeout: 5000     # 5s - Max time to establish connection
    socket-timeout: 30000        # 30s - Max time for request/response
```

**Pool Sizing Calculation:**
```
For monitoring service:
- 8 CPU cores per pod
- High write rate (metric ingestion)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin: 40 connections per pod
- 100 collector pods × 40 = 4,000 max connections
- Time-series DB cluster: 50 nodes, 80 connections per node capacity
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

---

## 2. Monitoring & Observability

### Key Metrics

**System Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Metric ingestion rate | 10M/s | < 9M/s | < 8M/s |
| Ingestion latency p99 | < 10ms | > 20ms | > 50ms |
| Query latency p95 | < 100ms | > 200ms | > 500ms |
| Alert evaluation latency | < 5s | > 10s | > 30s |

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low ingestion rate | < 9M/s for 5 min | Warning | Investigate |
| High query latency | > 500ms p95 for 2 min | Critical | Page on-call |
| Storage full | > 90% capacity | Critical | Expand storage |

---

## 3. Security Considerations

### Authentication

**Metrics Ingestion:**
- API key authentication (per service)
- Keys stored in secure vault

**Query API:**
- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles

### Data Encryption

- **At Rest**: AES-256 encryption for storage
- **In Transit**: TLS 1.3 for all communications

---

## 4. End-to-End Simulation

### User Journey: Create Alert and Receive Notification

**Step-by-Step Flow:**

```
Step 1: User Creates Alert Rule
┌─────────────────────────────────────────────────────────────┐
│ UI: POST /v1/alerts/rules                                   │
│ Query Engine: Creates alert rule in PostgreSQL              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Alert Engine Evaluates Rule
┌─────────────────────────────────────────────────────────────┐
│ Alert Engine (every 15 seconds):                            │
│ - Executes query: rate(http_requests_total{status=~'5..'}[5m])
│ - Condition met: 0.015 > 0.01 (threshold)                  │
│ - Records condition met in Redis                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Alert Fires
┌─────────────────────────────────────────────────────────────┐
│ Alert Engine:                                               │
│ - Condition met for 5 minutes (for duration)                │
│ - Creates alert in database                                 │
│ - Sends notification via PagerDuty, Slack, Email            │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | Percentage |
|-----------|--------------|------------|
| Storage (S3 + Time-Series DB) | $1,961,400 | 75% |
| Compute (Collectors, Query, Alerting) | $204,000 | 8% |
| Network | $6,000 | <1% |
| Misc | $434,280 | 17% |
| **Total** | **$2.6M** | **100%** |

### Cost Optimization Strategies

**Current Monthly Cost:** $500,000
**Target Monthly Cost:** $350,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Storage (Time-series DB):** $300,000/month (60%) - Largest single component
2. **Compute (EC2):** $150,000/month (30%) - Collection and query services
3. **Network (Data Transfer):** $50,000/month (10%) - High data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Compression (40% savings on storage):**
   - **Current:** Basic compression
   - **Optimization:** 
     - Better compression algorithms (reduce size by 40%)
     - Optimize compression settings
   - **Savings:** $120,000/month (40% of $300K storage)
   - **Trade-off:** Slight CPU overhead (acceptable)

2. **Retention Optimization (30% savings on storage):**
   - **Current:** 1-year retention for all metrics
   - **Optimization:** 
     - Reduce retention for non-critical metrics (30 days)
     - Keep critical metrics for 1 year
   - **Savings:** $90,000/month (30% of $300K storage)
   - **Trade-off:** Slight data loss for non-critical metrics (acceptable)

3. **Tiered Storage (50% savings on old metrics):**
   - **Current:** All metrics in time-series DB
   - **Optimization:** 
     - Move metrics > 30 days to S3 Glacier (80% cheaper)
     - Keep recent metrics in time-series DB
   - **Savings:** $90,000/month (30% of $300K storage)
   - **Trade-off:** Slower access to old metrics (acceptable for archive)

4. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for baseline
   - **Savings:** $60,000/month (40% of $150K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

5. **Query Optimization (30% savings on queries):**
   - **Current:** High query load
   - **Optimization:** 
     - Cache queries, reduce DB load
     - Pre-aggregate common queries
   - **Savings:** $15,000/month (30% of $50K query costs)
   - **Trade-off:** Slight complexity increase (acceptable)

**Total Potential Savings:** $375,000/month (75% reduction)
**Optimized Monthly Cost:** $125,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Metric Collection | $0.00005 | $0.0000125 | 75% |
| Metric Query | $0.0001 | $0.00003 | 70% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Compression, Retention Optimization → $210K savings
2. **Phase 2 (Month 2):** Tiered Storage, Reserved Instances → $150K savings
3. **Phase 3 (Month 3):** Query Optimization → $15K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor metric collection rates (ensure no degradation)
- Monitor storage costs (target < $90K/month)
- Review and adjust quarterly

---

## Summary

| Aspect | Decision |
|--------|----------|
| Load Balancing | DNS round-robin (collectors), LB (query services) |
| Rate Limiting | Per-service, 100K metrics/second default |
| Circuit Breakers | All external dependencies |
| Monitoring | Comprehensive metrics, alerting, health checks |
| Security | API keys (ingestion), OAuth (query), encryption |
| Cost Optimization | Compression, retention optimization, tiered storage |

