# Centralized Logging System - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Collector Distribution:**

DNS round-robin load balancing:

```
Services → DNS (collector.logging.internal) → Round-robin → Collector 1, 2, ..., 200
```

### Rate Limiting Implementation

**Per-Service Rate Limiting:**

```java
@Service
public class LogRateLimiter {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    public void acquire(String serviceName) {
        RateLimiter limiter = limiters.computeIfAbsent(serviceName,
            s -> RateLimiter.create(getRate(s)));
        limiter.acquire();
    }

    private double getRate(String serviceName) {
        return configService.getRateLimit(serviceName);  // Default: 1M logs/second
    }
}
```

### Circuit Breaker Placement

| Service             | Circuit Breaker | Fallback                |
| ------------------- | --------------- | ----------------------- |
| Log Ingestion       | Yes             | Buffer locally, retry   |
| Elasticsearch Write | Yes             | Queue logs, retry       |
| Redis Cache         | Yes             | Direct to Elasticsearch |

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Reduce log retention (7 days → 3 days)
2. **Second**: Disable search caching (increase DB load)
3. **Third**: Reduce indexing (store logs only, no full-text index)
4. **Last**: Stop collecting logs (unacceptable)

---

## 2. Monitoring & Observability

### Key Metrics

**System Metrics:**

| Metric                | Target | Warning | Critical |
| --------------------- | ------ | ------- | -------- |
| Log ingestion rate    | 100M/s | < 90M/s | < 80M/s  |
| Ingestion latency p99 | < 10ms | > 20ms  | > 50ms   |
| Search latency p95    | < 1s   | > 2s    | > 5s     |
| Storage utilization   | < 80%  | > 90%   | > 95%    |

---

## 3. Security Considerations

### Authentication

**Log Ingestion:**

- API key authentication (per service)
- Keys stored in secure vault

**Query API:**

- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles

### Data Encryption

- **At Rest**: AES-256 encryption for storage
- **In Transit**: TLS 1.3 for all communications
- **PII Redaction**: Automatic redaction of sensitive fields

---

## 4. End-to-End Simulation

### User Journey: Search Logs for Error

**Step-by-Step Flow:**

```
Step 1: User Searches Logs
┌─────────────────────────────────────────────────────────────┐
│ UI: Search "Payment processing failed" service=order-service│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Search Engine
┌─────────────────────────────────────────────────────────────┐
│ Search Engine:                                              │
│ - Checks cache (Redis) - MISS                               │
│ - Executes query against Elasticsearch                      │
│ - Returns matching logs                                     │
│ - Caches result (1-minute TTL)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: User Views Logs
┌─────────────────────────────────────────────────────────────┐
│ UI: Displays logs with context                              │
│ User: Identifies root cause from log details                │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes

- Maximum acceptable data loss
- Logs buffered in Kafka (1-day retention) provide buffer
- Elasticsearch replication lag: < 5 minutes
- Critical logs (errors) replicated with < 1 minute lag

**RTO (Recovery Time Objective):** < 30 minutes

- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes Elasticsearch cluster restoration, cache warm-up, and traffic rerouting

**Backup Strategy:**

- **Elasticsearch Backups**: Daily snapshots to S3, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to secondary region
- **Kafka Mirroring**: Real-time mirroring to DR region (logs buffered)
- **S3 Cold Storage**: Cross-region replication enabled

**Restore Steps:**

1. Detect primary region failure (health checks) - 1 minute
2. Promote Elasticsearch replica cluster to primary - 10 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Restore from latest S3 snapshot if needed - 10 minutes
5. Warm up Redis cache from Elasticsearch - 5 minutes
6. Verify logging service health and resume traffic - 3 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**

- [ ] DR region infrastructure provisioned and healthy
- [ ] Elasticsearch replication lag < 5 minutes
- [ ] Kafka replication lag < 5 minutes
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**

   - Record current traffic metrics (QPS, latency, error rate)
   - Capture log ingestion success rate
   - Document active log count
   - Verify Elasticsearch replication lag (< 5 minutes)
   - Test log ingestion (100 sample logs)

2. **Simulate Primary Region Failure (T+0):**

   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+30 minutes):**

   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-15 min:** Rebuild Elasticsearch index from Kafka
   - **T+15-20 min:** Warm up cache from Elasticsearch
   - **T+20-25 min:** Verify all services healthy in DR region
   - **T+25-30 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+30 to T+40 minutes):**

   - Verify RTO < 30 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check Elasticsearch replication lag at failure time
   - Test log ingestion: Ingest 1,000 test logs, verify >99% success
   - Test log search: Search 1,000 test logs, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify log integrity: Check logs accessible

5. **Data Integrity Verification:**

   - Compare log count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random logs accessible
   - Check search index: Verify search index intact
   - Test edge cases: High-volume logs, complex queries, PII redaction

6. **Failback Procedure (T+40 to T+45 minutes):**
   - Restore primary region services
   - Sync Elasticsearch data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**

- ✅ RTO < 30 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via Elasticsearch replication lag)
- ✅ Log ingestion works: >99% logs ingested successfully
- ✅ No critical logs lost: Log count matches pre-failover (within 5 min window)
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

### Journey 1: Log Ingestion and Search

**Step-by-step:**

1. **Service emits log**: User service sends `POST /v1/logs` with `{"level": "ERROR", "message": "Database connection failed", "service": "user-service", "timestamp": "..."}`
2. **Log Collector**: 
   - Receives log
   - Validates format
   - Publishes to Kafka: `{"log": {...}, "partition": 5}` (partitioned by service)
3. **Log Processor** (consumes Kafka):
   - Receives log from partition 5
   - Parses log fields
   - Indexes in Elasticsearch: `POST /logs/_doc` with log data
   - Stores in S3: `s3://logs/2024/12/19/user-service/partition_5/chunk_001`
4. **User searches logs** (5 minutes later):
   - Request: `GET /v1/logs/search?q=error&service=user-service&time_range=1h`
   - Query Elasticsearch: `GET /logs/_search {"query": {"bool": {"must": [{"match": {"message": "error"}}, {"term": {"service": "user-service"}}]}}}`
   - Returns matching logs
5. **Response**: `200 OK` with `{"logs": [{"level": "ERROR", "message": "Database connection failed", ...}, ...], "total": 10}`
6. **Result**: Log ingested and searchable within 1 second

**Total latency: Ingestion ~100ms, Indexing ~500ms, Search ~200ms**

### Journey 2: Log Aggregation and Dashboard

**Step-by-step:**

1. **Background Job** (runs every 5 minutes):
   - Queries Elasticsearch: Aggregates logs by service, level, time
   - Calculates metrics: Error rate, log volume per service
   - Updates dashboard data: `UPDATE dashboards SET metrics = {...}`
2. **User views dashboard**:
   - Request: `GET /v1/dashboards/logs-overview`
   - Returns aggregated metrics
3. **Response**: `200 OK` with dashboard data
4. **Result**: Dashboard shows real-time log metrics

**Total time: Aggregation ~30 seconds, Dashboard query ~100ms**

### Failure & Recovery Walkthrough

**Scenario: Elasticsearch Cluster Degradation**

**RTO (Recovery Time Objective):** < 10 minutes (cluster rebalancing + index recovery)  
**RPO (Recovery Point Objective):** 0 (logs buffered in Kafka, no data loss)

**Timeline:**

```
T+0s:    Elasticsearch node 3 crashes (handles shards 5-7)
T+0-10s: Log indexing to shards 5-7 fails
T+10s:   Logs buffered in Kafka (retention 7 days)
T+15s:   Cluster detects node failure
T+20s:   Replicas promoted to primary for shards 5-7
T+25s:   Cluster rebalances, redistributes shards
T+30s:   Log indexing resumes
T+2min:  Kafka replay begins (replay buffered logs)
T+10min: All logs indexed, cluster fully recovered
```

**What degrades:**
- Log indexing delayed for 20-30 seconds
- Logs buffered in Kafka (no data loss)
- Search queries may hit replicas (slightly higher latency)

**What stays up:**
- Log ingestion (buffered in Kafka)
- Search queries (reads from other shards)
- All read operations

**What recovers automatically:**
- Elasticsearch cluster promotes replicas
- Cluster rebalances automatically
- Kafka replay processes buffered logs
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of node failure
- Replace failed hardware
- Review cluster capacity

**Cascading failure prevention:**
- Kafka buffering prevents log loss
- Cluster replication prevents data loss
- Circuit breakers prevent retry storms
- Timeouts prevent thread exhaustion

### Database Connection Pool Configuration

**Elasticsearch Connection Pool (HTTP Client):**

```yaml
elasticsearch:
  client:
    # Connection pool sizing
    max-connections: 50 # Max connections per application instance
    max-connections-per-route: 10 # Max connections per Elasticsearch node

    # Timeouts
    connection-timeout: 5000 # 5s - Max time to establish connection
    socket-timeout: 30000 # 30s - Max time for request/response

    # Keep-alive
    keep-alive: 30000 # 30s - Keep connections alive
```

**Pool Sizing Calculation:**

```
For search services:
- 4 CPU cores per pod
- High query rate (search operations)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin for concurrent searches: 50 connections per pod
- 100 search pods × 50 = 5,000 max connections
- Elasticsearch cluster: 50 nodes, 100 connections per node capacity
```

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 20 # Max connections per instance
      max-idle: 10 # Max idle connections
      min-idle: 5 # Min idle connections

    # Timeouts
    connection-timeout: 2000 # 2s - Max time to get connection
    socket-timeout: 3000 # 3s - Max time for operation
```

**Pool Sizing Calculation:**

```
For search cache:
- 4 CPU cores per pod
- Read-heavy (cache lookups)
- Calculated: 20 connections per pod
- 100 search pods × 20 = 2,000 max connections
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

3. **Connection Pooler:**
   - Use connection pooling libraries (HikariCP for DB, Jedis pool for Redis)
   - Monitor pool metrics and adjust based on load

---

## 6. Cost Analysis

### Major Cost Drivers

| Component                    | Monthly Cost | Percentage |
| ---------------------------- | ------------ | ---------- |
| Storage (S3 + Elasticsearch) | $4,892,750   | 70%        |
| Compute (Collectors, Search) | $840,000     | 12%        |
| Network                      | $60,000      | <1%        |
| Misc                         | $1,207,250   | 18%        |
| **Total**                    | **$7M**      | **100%**   |

### Cost Optimization Strategies

**Current Monthly Cost:** $800,000
**Target Monthly Cost:** $560,000 (30% reduction)

**Top 3 Cost Drivers:**

1. **Storage (Elasticsearch + S3):** $500,000/month (63%) - Largest single component
2. **Compute (EC2):** $200,000/month (25%) - Collection and search services
3. **Network (Data Transfer):** $100,000/month (13%) - High data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Compression (40% savings on storage):**

   - **Current:** Basic compression (4:1 ratio)
   - **Optimization:**
     - Better compression algorithms (6:1 ratio, 50% improvement)
     - Optimize compression settings
   - **Savings:** $200,000/month (40% of $500K storage)
   - **Trade-off:** Slight CPU overhead (acceptable)

2. **Retention Optimization (30% savings on storage):**

   - **Current:** 90-day retention for all logs
   - **Optimization:**
     - Reduce retention for non-critical logs (30 days)
     - Keep critical logs for 90 days
   - **Savings:** $150,000/month (30% of $500K storage)
   - **Trade-off:** Slight data loss for non-critical logs (acceptable)

3. **Tiered Storage (50% savings on old logs):**

   - **Current:** All logs in Elasticsearch
   - **Optimization:**
     - Move logs > 7 days to S3 Glacier (80% cheaper)
     - Keep recent logs in Elasticsearch
   - **Savings:** $200,000/month (40% of $500K storage)
   - **Trade-off:** Slower access to old logs (acceptable for archive)

4. **Index Optimization (20% savings on Elasticsearch):**

   - **Current:** All fields indexed
   - **Optimization:**
     - Reduce index size (fewer indexed fields)
     - Optimize index settings
   - **Savings:** $100,000/month (20% of $500K Elasticsearch)
   - **Trade-off:** Slight query flexibility reduction (acceptable)

5. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 1-year Reserved Instances for baseline
   - **Savings:** $80,000/month (40% of $200K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

**Total Potential Savings:** $730,000/month (91% reduction)
**Optimized Monthly Cost:** $70,000/month

**Cost per Operation Breakdown:**

| Operation     | Current Cost | Optimized Cost | Reduction |
| ------------- | ------------ | -------------- | --------- |
| Log Ingestion | $0.0000185   | $0.0000017     | 91%       |
| Log Search    | $0.0001      | $0.00001       | 90%       |

**Implementation Priority:**

1. **Phase 1 (Month 1):** Compression, Retention Optimization → $350K savings
2. **Phase 2 (Month 2):** Tiered Storage, Index Optimization → $300K savings
3. **Phase 3 (Month 3):** Reserved Instances → $80K savings

**Monitoring & Validation:**

- Track cost reduction weekly
- Monitor log ingestion rates (ensure no degradation)
- Monitor storage costs (target < $200K/month)
- Review and adjust quarterly

---

## Summary

| Aspect            | Decision                                                       |
| ----------------- | -------------------------------------------------------------- |
| Load Balancing    | DNS round-robin (collectors), LB (search services)             |
| Rate Limiting     | Per-service, 1M logs/second default                            |
| Circuit Breakers  | All external dependencies                                      |
| Monitoring        | Comprehensive metrics, alerting, health checks                 |
| Security          | API keys (ingestion), OAuth (query), encryption, PII redaction |
| Cost Optimization | Compression, retention optimization, tiered storage            |
