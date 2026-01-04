# Distributed Tracing System - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Collector Distribution:**

Load balancing via DNS round-robin and client-side load balancing:

```
Services → DNS (collector.tracing.internal) → Round-robin → Collector 1, 2, ..., 50
```

**Query Service Distribution:**

```
Users → Load Balancer → Query Service 1, 2, ..., 10
```

### Rate Limiting Implementation

**Per-Service Rate Limiting:**

```java
@Service
public class SpanRateLimiter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    public void acquire(String serviceName) {
        RateLimiter limiter = limiters.computeIfAbsent(serviceName, 
            s -> RateLimiter.create(getRate(s)));
        limiter.acquire();
    }
    
    private double getRate(String serviceName) {
        // Get rate limit from config (default: 10K spans/second)
        return configService.getRateLimit(serviceName);
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Span Ingestion | Yes | Buffer locally, retry |
| Elasticsearch Write | Yes | Queue spans, retry |
| Redis Cache | Yes | Direct to Elasticsearch |
| Query Service | Yes | Return cached results |

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Span Ingestion | 100ms | 2 | 10ms, 50ms |
| Elasticsearch Query | 5s | 2 | 1s, 2s |
| Trace Aggregation | 30s | 0 | N/A |
| Cache Lookup | 10ms | 0 | N/A |

### Replication and Failover

**Collectors:**
- 50 collectors across 3 AZs
- Stateless design (state in Kafka/Redis)
- Auto-scaling based on span rate

**Failover:**
- Collector fails → Load balancer routes to healthy collectors
- Aggregator fails → Kafka rebalances partitions
- Elasticsearch node fails → Automatic replica promotion

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Reduce sampling rate (1% → 0.5%)
2. **Second**: Disable trace reconstruction (store spans only)
3. **Third**: Disable cold storage (hot storage only)
4. **Last**: Stop collecting spans (unacceptable)

### Disaster Recovery

**Multi-Region:**
- Primary: us-east-1
- DR: us-west-2 (async replication)

**RPO (Recovery Point Objective):** < 1 hour
- Maximum acceptable data loss
- Based on Kafka retention (1 hour)
- Traces stored in S3 with cross-region replication

**RTO (Recovery Time Objective):** < 30 minutes
- Maximum acceptable downtime
- Time to restore service via failover to DR region

**Backup Strategy:**
- **Elasticsearch Backups**: Daily snapshots, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to DR region
- **Kafka Mirroring**: Real-time mirroring to DR region (traces buffered)
- **S3 Cold Storage**: Cross-region replication enabled

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote Elasticsearch replica cluster to primary - 10 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Restore from latest S3 snapshot if needed - 10 minutes
5. Warm up Redis cache from Elasticsearch - 5 minutes
6. Verify tracing service health and resume traffic - 3 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Elasticsearch replication lag < 1 hour
- [ ] Kafka replication lag < 1 hour
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture trace collection success rate
   - Document active trace count
   - Verify Elasticsearch replication lag (< 1 hour)
   - Test trace collection (100 sample traces)

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

4. **Post-Failover Validation (T+30 to T+45 minutes):**
   - Verify RTO < 30 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 hour: Check Elasticsearch replication lag at failure time
   - Test trace collection: Collect 1,000 test traces, verify >99% success
   - Test trace queries: Query 1,000 test traces, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify trace integrity: Check traces accessible

5. **Data Integrity Verification:**
   - Compare trace count: Pre-failover vs post-failover (should match within 1 hour)
   - Spot check: Verify 100 random traces accessible
   - Check trace completeness: Verify traces complete
   - Test edge cases: Large traces, high-throughput, complex queries

6. **Failback Procedure (T+45 to T+60 minutes):**
   - Restore primary region services
   - Sync Elasticsearch data from DR to primary
   - Verify replication lag < 1 hour
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 30 minutes: Time from failure to service resumption
- ✅ RPO < 1 hour: Maximum data loss (verified via Elasticsearch replication lag)
- ✅ Trace collection works: >99% traces collected successfully
- ✅ No critical traces lost: Trace count matches pre-failover (within 1 hour window)
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

### Journey 1: Request Generates Trace

**Step-by-step:**

1. **User Request**: Client sends `GET /v1/api/users/123`
2. **API Gateway**: 
   - Generates trace ID: `trace_abc123`
   - Creates root span: `span_1` (gateway)
   - Adds trace context to headers: `X-Trace-ID: trace_abc123, X-Span-ID: span_1`
3. **User Service**: 
   - Receives request with trace context
   - Creates child span: `span_2` (user-service)
   - Queries database: `SELECT * FROM users WHERE id = 123`
   - Creates child span: `span_3` (database-query)
   - Returns user data
4. **User Service** (continues):
   - Calls email service: `GET /v1/emails/user_123`
   - Creates child span: `span_4` (email-service-call)
   - Email service creates child span: `span_5` (email-service)
5. **Spans sent to collector** (async):
   - Spans batched and sent: `POST /v1/spans` with `[span_1, span_2, span_3, span_4, span_5]`
   - Collector stores in Elasticsearch
6. **Trace query** (user queries trace):
   - Request: `GET /v1/traces/trace_abc123`
   - Query Elasticsearch: Returns all spans for trace
   - Reconstructs trace tree
7. **Response**: `200 OK` with complete trace visualization
8. **Result**: Full trace captured and queryable within 1 second

**Total latency: Request ~100ms, Trace storage ~500ms, Query ~200ms**

### Journey 2: Error Trace (100% Sampling)

**Step-by-step:**

1. **Request fails** (database error):
   - User service throws exception
   - Error span created with `error=true` tag
   - Error trace sampled at 100% (not 1%)
2. **Error trace sent**:
   - All spans for error trace sent to collector
   - Collector stores with `error=true` flag
3. **Alert triggered**:
   - Monitoring system queries: `GET /v1/traces?error=true&service=user-service&time_range=5m`
   - Finds error traces
   - Sends alert to on-call engineer
4. **Engineer investigates**:
   - Views trace: `GET /v1/traces/trace_error_xyz`
   - Sees database connection error in span_3
   - Identifies root cause
5. **Result**: Error traced and alerted within 30 seconds

**Total time: Error occurs → Alert → Investigation ~30 seconds**

### Failure & Recovery Walkthrough

**Scenario: Trace Collector Failure**

**RTO (Recovery Time Objective):** < 5 minutes (collector restart + buffer replay)  
**RPO (Recovery Point Objective):** < 1 minute (spans buffered, no data loss)

**Timeline:**

```
T+0s:    Trace collector crashes (handles 50% of spans)
T+0-10s: Spans fail to send
T+10s:   Client-side buffer fills (1000 spans)
T+15s:   Spans dropped (buffer full)
T+20s:   Health check fails
T+25s:   Kubernetes restarts collector
T+30s:   Collector restarts, begins accepting spans
T+1min:  Buffer replay begins (replay dropped spans if available)
T+5min:  All operations normal
```

**What degrades:**
- Trace collection unavailable for 20-30 seconds
- Some spans dropped (buffer overflow)
- Trace queries unaffected (reads from Elasticsearch)

**What stays up:**
- Application requests (tracing is non-blocking)
- Trace queries (reads from Elasticsearch)
- Other collectors (if multiple)

**What recovers automatically:**
- Kubernetes restarts failed collector
- Collector resumes accepting spans
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of collector failure
- Review collector capacity
- Check for systemic issues

**Cascading failure prevention:**
- Client-side buffering prevents span loss
- Sampling reduces load
- Circuit breakers prevent retry storms
- Timeouts prevent thread exhaustion

### Database Connection Pool Configuration

**Elasticsearch Connection Pool (HTTP Client):**

```yaml
elasticsearch:
  client:
    # Connection pool sizing
    max-connections: 50          # Max connections per application instance
    max-connections-per-route: 10 # Max connections per Elasticsearch node
    
    # Timeouts
    connection-timeout: 5000     # 5s - Max time to establish connection
    socket-timeout: 30000        # 30s - Max time for request/response
    
    # Keep-alive
    keep-alive: 30000            # 30s - Keep connections alive
```

**Pool Sizing Calculation:**
```
For trace service:
- 4 CPU cores per pod
- High write rate (span ingestion)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 50 connections per pod
- 50 collector pods × 50 = 2,500 max connections
- Elasticsearch cluster: 50 nodes, 50 connections per node capacity
```

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 20             # Max connections per instance
      max-idle: 10                # Max idle connections
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

**Span Collection Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Span ingestion rate | 550K/s | < 500K/s | < 400K/s |
| Ingestion latency p99 | < 10ms | > 20ms | > 50ms |
| Ingestion success rate | > 99.9% | < 99% | < 95% |
| Sampling accuracy | 1% ± 0.1% | ± 0.2% | ± 0.5% |

**Trace Processing Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Trace completion rate | > 99% | < 95% | < 90% |
| Aggregation latency p95 | < 1s | > 2s | > 5s |
| Storage write latency p99 | < 500ms | > 1s | > 2s |

**Query Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Query latency p95 | < 500ms | > 1s | > 2s |
| Query latency p99 | < 2s | > 5s | > 10s |
| Cache hit rate | > 80% | < 70% | < 60% |

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low ingestion rate | < 500K/s for 5 min | Warning | Investigate |
| High ingestion latency | > 50ms p99 for 2 min | Critical | Page on-call |
| High error rate | > 1% for 5 min | Warning | Check services |
| Storage full | > 90% capacity | Critical | Expand storage |

### Health Checks

```java
@Component
public class TracingHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check Elasticsearch connectivity
        boolean esHealthy = elasticsearchService.ping();
        details.put("elasticsearch", esHealthy ? "UP" : "DOWN");
        
        // Check Kafka connectivity
        boolean kafkaHealthy = kafkaTemplate.send("health-check", "ping")
            .get(5, TimeUnit.SECONDS) != null;
        details.put("kafka", kafkaHealthy ? "UP" : "DOWN");
        
        // Check Redis connectivity
        boolean redisHealthy = redis.getConnectionFactory()
            .getConnection().ping() != null;
        details.put("redis", redisHealthy ? "UP" : "DOWN");
        
        if (!esHealthy || !kafkaHealthy || !redisHealthy) {
            return Health.down().withDetails(details).build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

---

## 3. Security Considerations

### Authentication

**Span Ingestion:**
- API key authentication (per service)
- Keys stored in secure vault
- Rate limiting per API key

**Query API:**
- OAuth 2.0 / JWT Bearer token
- RBAC: Read-only, Admin roles

### Data Encryption

- **At Rest**: AES-256 encryption for Elasticsearch and S3
- **In Transit**: TLS 1.3 for all communications
- **PII Redaction**: Automatic redaction of sensitive tags (user IDs, passwords)

### Input Validation

```java
@Service
public class SpanValidator {
    
    private static final int MAX_TAG_COUNT = 100;
    private static final int MAX_TAG_KEY_LENGTH = 256;
    private static final int MAX_TAG_VALUE_LENGTH = 4096;
    
    public void validate(Span span) {
        // Validate trace_id format
        if (!isValidTraceId(span.getTraceId())) {
            throw new InvalidSpanException("Invalid trace_id format");
        }
        
        // Validate tag count
        if (span.getTags().size() > MAX_TAG_COUNT) {
            throw new InvalidSpanException("Too many tags");
        }
        
        // Validate tag keys/values
        span.getTags().forEach((key, value) -> {
            if (key.length() > MAX_TAG_KEY_LENGTH) {
                throw new InvalidSpanException("Tag key too long");
            }
            if (value.length() > MAX_TAG_VALUE_LENGTH) {
                throw new InvalidSpanException("Tag value too long");
            }
        });
        
        // Redact PII
        redactPII(span);
    }
    
    private void redactPII(Span span) {
        // Redact sensitive tags
        span.getTags().remove("user.password");
        span.getTags().remove("user.token");
        // Mask user IDs
        if (span.getTags().containsKey("user.id")) {
            span.getTags().put("user.id", maskUserId(span.getTags().get("user.id")));
        }
    }
}
```

---

## 4. End-to-End Simulation

### User Journey: Debug Production Incident

**Scenario:** User reports slow order creation

**Step-by-Step Flow:**

```
Step 1: User Searches for Error Traces
┌─────────────────────────────────────────────────────────────┐
│ UI: GET /v1/traces?service_name=order-service&tag=http.status_code=500&start_time=1705312245&end_time=1705312246
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Query Service Processes Request
┌─────────────────────────────────────────────────────────────┐
│ Query Service:                                              │
│ - Validates query parameters                                │
│ - Checks cache (Redis) - MISS                               │
│ - Queries Elasticsearch                                     │
│ - Filters by service_name, status_code, time range          │
│ - Returns 10 traces with errors                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: User Views Trace Details
┌─────────────────────────────────────────────────────────────┐
│ UI: GET /v1/traces/abc123...                                │
│ Query Service:                                              │
│ - Checks cache - MISS                                       │
│ - Queries Elasticsearch by trace_id                         │
│ - Retrieves complete trace with all spans                   │
│ - Caches result (5-minute TTL)                              │
│ - Returns trace                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: User Identifies Root Cause
┌─────────────────────────────────────────────────────────────┐
│ Trace shows:                                                │
│ - API Gateway (100ms)                                       │
│   └─ Order Service (2000ms) ← SLOW                         │
│      └─ Database Query (1800ms) ← BOTTLENECK              │
│                                                                
│ User identifies: Database query is slow                     │
│ Action: Optimize database query or add index                │
└─────────────────────────────────────────────────────────────┘
```

### Failure & Recovery Walkthrough

**Scenario:** Elasticsearch cluster loses one node

```
T+0s: Elasticsearch node fails
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Impact:                                                      │
│ - Elasticsearch cluster degrades (reduced capacity)         │
│ - Query latency increases (p95: 500ms → 1s)                │
│ - Write latency increases (p99: 500ms → 1.5s)              │
│ - Span collection continues (Kafka buffers)                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Automatic Recovery:                                          │
│ - Elasticsearch detects node failure                        │
│ - Promotes replica shards to primary                        │
│ - Rebalances shards across remaining nodes                   │
│ - System continues operating (degraded performance)         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Human Intervention:                                          │
│ - Alert fires (node down)                                   │
│ - On-call engineer investigates                             │
│ - Replaces failed node                                      │
│ - Elasticsearch rebalances shards to new node                │
│ - Performance returns to normal                              │
└─────────────────────────────────────────────────────────────┘

Recovery Time:
- Automatic failover: < 30 seconds
- Full recovery: < 10 minutes (after node replacement)
- Data Loss: None (replicated shards)
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | Percentage |
|-----------|--------------|------------|
| Storage (S3 + Elasticsearch) | $155,610 | 66% |
| Compute (Collectors, Aggregators, Query) | $38,000 | 16% |
| Network (Cross-AZ, Data Transfer) | $2,500 | 1% |
| Misc (Monitoring, Load Balancers) | $39,222 | 17% |
| **Total** | **$235,332** | **100%** |

### Cost Optimization Strategies

**Current Monthly Cost:** $235,000
**Target Monthly Cost:** $164,500 (30% reduction)

**Top 3 Cost Drivers:**
1. **Elasticsearch:** $150,000/month (64%) - Largest single component
2. **Storage (S3):** $50,000/month (21%) - Trace storage
3. **Compute (EC2):** $25,000/month (11%) - Collection and query services

**Optimization Strategies (Ranked by Impact):**

1. **Sampling (40% savings on ingestion):**
   - **Current:** 100% sampling rate
   - **Optimization:** 
     - Reduce sampling rate during low-traffic periods (50% sampling)
     - Adaptive sampling based on traffic
   - **Savings:** $60,000/month (40% of $150K Elasticsearch)
   - **Trade-off:** Slight trace coverage reduction (acceptable)

2. **Tiered Storage (60% savings on old traces):**
   - **Current:** All traces in Elasticsearch
   - **Optimization:** 
     - Move traces > 7 days to S3 Glacier (80% cheaper)
     - Keep recent traces in Elasticsearch
   - **Savings:** $30,000/month (60% of $50K storage)
   - **Trade-off:** Slower access to old traces (acceptable for archive)

3. **Compression (30% savings on storage):**
   - **Current:** Basic compression
   - **Optimization:** 
     - Better compression algorithms (reduce size by 30%)
     - Optimize compression settings
   - **Savings:** $15,000/month (30% of $50K storage)
   - **Trade-off:** Slight CPU overhead (acceptable)

4. **Index Optimization (20% savings on Elasticsearch):**
   - **Current:** All fields indexed
   - **Optimization:** 
     - Reduce index size (fewer indexed fields)
     - Optimize index settings
   - **Savings:** $30,000/month (20% of $150K Elasticsearch)
   - **Trade-off:** Slight query flexibility reduction (acceptable)

5. **Cache Tuning (30% savings on queries):**
   - **Current:** 60% cache hit rate
   - **Optimization:** 
     - Increase cache hit rate to 80%
     - Reduce Elasticsearch queries
   - **Savings:** $15,000/month (30% of $50K query costs)
   - **Trade-off:** Slight memory increase (acceptable)

**Total Potential Savings:** $150,000/month (64% reduction)
**Optimized Monthly Cost:** $85,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Trace Collection | $0.000235 | $0.000085 | 64% |
| Trace Query | $0.0001 | $0.00004 | 60% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Sampling, Tiered Storage → $90K savings
2. **Phase 2 (Month 2):** Compression, Index Optimization → $45K savings
3. **Phase 3 (Month 3):** Cache Tuning → $15K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor trace coverage (ensure >95% coverage)
- Monitor storage costs (target < $20K/month)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Span Collection | Build | Custom protocol support, control |
| Storage | Buy (Elasticsearch) | Complex to build, managed service |
| Query Service | Build | Custom query logic, integration |
| Visualization UI | Buy (Jaeger UI) | Open source, feature-rich |

### Cost at Scale

| Scale | Monthly Cost | Cost per Million Traces |
|-------|--------------|-------------------------|
| Current (1M req/s) | $235K | $8.20 |
| 10x (10M req/s) | $2.2M | $7.70 |
| 100x (100M req/s) | $20M | $7.00 |

Cost per trace decreases at scale due to fixed costs amortization.

---

## Summary

| Aspect | Decision |
|--------|----------|
| Load Balancing | DNS round-robin (collectors), LB (query services) |
| Rate Limiting | Per-service, 10K spans/second default |
| Circuit Breakers | All external dependencies |
| Disaster Recovery | Multi-region, RPO < 1h, RTO < 30min |
| Monitoring | Comprehensive metrics, alerting, health checks |
| Security | API keys (ingestion), OAuth (query), encryption, PII redaction |
| Cost Optimization | Sampling, compression, tiered storage |

