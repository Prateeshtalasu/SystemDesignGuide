# Distributed Cache - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing

**Strategy:**
- Consistent hashing distributes keys across nodes
- No central load balancer needed
- Client library routes to correct node

### Replication Strategy

**Configuration:**
- Replication factor: 2-3
- Primary: 1 per shard
- Replicas: 1-2 per shard

**Write Path:**
1. Write to primary
2. Replicate to replicas (synchronous for critical data)
3. Confirm write

**Read Path:**
1. Read from primary (strong consistency)
2. Or read from replica (eventual consistency, lower latency)

### Circuit Breaker

**Configuration:**
- Failure threshold: 50% for 10 seconds
- Half-open timeout: 30 seconds
- Fallback: Return null (cache miss, fetch from database)

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Cache GET | 10ms | 1 | None |
| Cache SET | 10ms | 1 | None |
| Node connection | 5s | 2 | 1s |

---

## 2. Monitoring & Observability

### Key Metrics

| Metric | Target | Warning | Critical | How to Measure |
|--------|--------|---------|----------|----------------|
| Hit rate | > 80% | < 75% | < 70% | Cache hits / Total requests |
| Latency p50 | < 0.5ms | > 1ms | > 2ms | Time from request to response |
| Latency p95 | < 1ms | > 2ms | > 5ms | 95th percentile latency |
| Latency p99 | < 2ms | > 5ms | > 10ms | 99th percentile latency |
| Node availability | 99.99% | < 99.9% | < 99% | Health check success rate |
| Memory usage | < 80% | > 85% | > 90% | Used memory / Total memory |

### Metrics Collection

```java
@Component
public class CacheMetrics {
    
    private final MeterRegistry registry;
    
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Timer cacheLatency;
    
    public CacheMetrics(MeterRegistry registry) {
        this.cacheHits = Counter.builder("cache.hits")
            .register(registry);
        
        this.cacheMisses = Counter.builder("cache.misses")
            .register(registry);
        
        this.cacheLatency = Timer.builder("cache.latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public void recordHit(String key) {
        cacheHits.increment();
    }
    
    public void recordMiss(String key) {
        cacheMisses.increment();
    }
    
    public void recordLatency(long durationMs) {
        cacheLatency.record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low hit rate | < 75% for 10 min | Warning | Investigate |
| Very low hit rate | < 70% for 5 min | Critical | Page on-call |
| High latency | p95 > 2ms for 5 min | Warning | Check nodes |
| Very high latency | p95 > 5ms for 2 min | Critical | Page on-call |
| Node down | 100% failure | Critical | Automatic failover |
| Memory high | > 85% for 10 min | Warning | Check eviction |

### Health Checks

```java
@Component
public class CacheHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check node connectivity
        boolean nodesHealthy = checkAllNodes();
        details.put("nodes", nodesHealthy ? "UP" : "DOWN");
        
        // Check hit rate
        double hitRate = metrics.getHitRate();
        details.put("hit_rate", hitRate);
        
        // Check memory usage
        double memoryUsage = getMemoryUsage();
        details.put("memory_usage", memoryUsage);
        
        if (!nodesHealthy) {
            return Health.down().withDetails(details).build();
        }
        
        if (hitRate < 0.70) {
            return Health.down()
                .withDetails(details)
                .withDetail("error", "Hit rate too low")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

---

## 3. Security Considerations

### Authentication

**Redis AUTH:**
```conf
# redis.conf
requirepass strong_password_here
```

**Connection Security:**
- TLS for all connections
- Certificate-based authentication
- IP whitelisting

### Key Validation

**Prevent Key Injection:**
```java
public boolean isValidKey(String key) {
    // Only allow alphanumeric, colon, underscore, dash
    return key.matches("^[a-zA-Z0-9:_-]+$") && 
           key.length() <= 255 &&
           !key.contains("..") &&  // Prevent path traversal
           !key.startsWith("_internal_");  // Protect internal keys
}
```

### Data Encryption

**At Rest:**
- Redis AOF encryption (optional)
- Disk encryption (EBS encryption)

**In Transit:**
- TLS 1.3 for all connections
- Certificate pinning

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Cache Hit

```
1. Application requests: GET cache/user:123
   ↓
2. Client library: Hash key → Find node (Node2)
   ↓
3. Connect to Node2
   ↓
4. Redis: GET user:123
   Result: HIT, value: { "name": "John" }
   ↓
5. Return value to application
   ↓
6. Total latency: ~1ms
```

### Journey 2: Cache Miss

```
1. Application requests: GET cache/user:456
   ↓
2. Client library: Hash key → Find node (Node1)
   ↓
3. Connect to Node1
   ↓
4. Redis: GET user:456
   Result: MISS
   ↓
5. Application: Fetch from database
   Database: SELECT * FROM users WHERE id=456
   Latency: ~10ms
   ↓
6. Application: Cache value
   Redis: SET user:456 {value} EX 3600
   ↓
7. Return value to application
   ↓
8. Total latency: ~15ms (vs ~1ms for cache hit)
```

### Journey 3: Node Failure

```
1. Node2 fails (health check detects)
   ↓
2. Remove Node2 from hash ring
   ↓
3. Keys reassigned to other nodes
   ↓
4. Client library: Update node list
   ↓
5. Next request: Hash key → Find new node (Node3)
   ↓
6. Connect to Node3
   ↓
7. Cache MISS (key not on Node3 yet)
   ↓
8. Fetch from database, cache on Node3
   ↓
9. Normal operations resume
```

### Failure & Recovery

**Scenario: Cache Node Failure**

**RTO (Recovery Time Objective):** < 1 minute (automatic failover)  
**RPO (Recovery Point Objective):** 0 (no data loss, data in replicas)

**Runbook Steps:**

1. **Detection (T+0s to T+10s):**
   - Monitor: Health check failures
   - Alert: PagerDuty alert triggered at T+10s
   - Verify: Check node status
   - Confirm: Node failure confirmed

2. **Immediate Response (T+10s to T+30s):**
   - Action: Remove node from hash ring
   - Fallback: Reassign keys to other nodes
   - Impact: Cache misses for reassigned keys
   - Monitor: Verify other nodes healthy

3. **Recovery (T+30s to T+60s):**
   - T+30s: Automatic failover to replica
   - T+45s: Replica promoted to primary
   - T+60s: Keys repopulate from database
   - T+60s: Normal operations resume

4. **Post-Recovery (T+60s to T+300s):**
   - Monitor: Watch hit rate recovery
   - Verify: Cache repopulating
   - Document: Log incident

**Prevention:**
- Replication (2-3 replicas per shard)
- Health checks every 10 seconds
- Automatic failover enabled

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Redis replication lag = 0 (synchronous replication)
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture cache hit rate
   - Document active key count
   - Verify Redis replication lag (= 0, synchronous)
   - Test cache operations (100 sample operations)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+1 minute):**
   - **T+0-10s:** Detect failure via health checks
   - **T+10-20s:** Promote secondary region to primary
   - **T+20-30s:** Update DNS records (Route53 health checks)
   - **T+30-50s:** Verify all services healthy in DR region
   - **T+50-60s:** Resume traffic to DR region

4. **Post-Failover Validation (T+1 to T+5 minutes):**
   - Verify RTO < 1 minute: ✅ PASS/FAIL
   - Verify RPO = 0: Check key count matches pre-failover (exact match)
   - Test cache operations: Perform 1,000 test cache operations, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify keys repopulate: Check keys repopulate correctly

5. **Data Integrity Verification:**
   - Compare key count: Pre-failover vs post-failover (should match exactly, RPO=0)
   - Spot check: Verify 100 random cache keys accessible
   - Test edge cases: High-throughput operations, large values, TTL expiration

6. **Failback Procedure (T+5 to T+6 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag = 0
   - Update DNS to route traffic back to primary
   - Monitor for 1 minute before declaring success

**Validation Criteria:**
- ✅ RTO < 1 minute: Time from failure to service resumption
- ✅ RPO = 0: No data loss (verified via key count, exact match)
- ✅ Cache operations work: >99% operations successful
- ✅ No data loss: Key count matches pre-failover exactly
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

### Journey 1: Cache Hit (Read Operation)

**Step-by-step:**

1. **Application Request**: App requests data via `GET /v1/cache/user:123`
2. **Cache Service**: 
   - Calculates hash: `hash("user:123") % 10 = 5`
   - Routes to node 5
   - Checks cache: `GET user:123` → HIT
   - Returns cached value: `{"name": "John", "email": "john@example.com"}`
3. **Response**: `200 OK` with cached data
4. **Result**: Data served from cache, < 1ms latency

**Total latency: < 1ms** (cache lookup)

### Journey 2: Cache Miss (Read-Through)

**Step-by-step:**

1. **Application Request**: App requests data via `GET /v1/cache/user:456`
2. **Cache Service**: 
   - Routes to node 7
   - Checks cache: `GET user:456` → MISS
3. **Cache Service** (read-through):
   - Fetches from database: `SELECT * FROM users WHERE id = 456`
   - Receives data: `{"name": "Jane", "email": "jane@example.com"}`
   - Stores in cache: `SET user:456 {...} EX 3600` (1 hour TTL)
   - Returns data to application
4. **Response**: `200 OK` with data from database
5. **Result**: Data fetched from database, cached for future requests

**Total latency: ~50ms** (cache miss 1ms + database query 45ms + cache write 4ms)

### Failure & Recovery Walkthrough

**Scenario: Cache Node Failure**

**RTO (Recovery Time Objective):** < 5 minutes (node replacement + data rebalancing)  
**RPO (Recovery Point Objective):** 0 (data rebalanced, no data loss)

**Timeline:**

```
T+0s:    Cache node 5 crashes (handles keys 40-50%)
T+0-5s:  Requests to node 5 fail
T+5s:    Cluster detects node failure
T+10s:   Keys rebalanced to remaining nodes
T+15s:   New node provisioned
T+30s:   New node joins cluster
T+60s:   Keys rebalanced to include new node
T+5min:  Cluster fully recovered
```

**What degrades:**
- Cache unavailable for keys 40-50% for 10-15 seconds
- Requests fall back to database (higher latency)
- No data loss (keys rebalanced)

**What stays up:**
- Other cache nodes continue serving
- Database fallback works
- All read operations succeed

**What recovers automatically:**
- Cluster rebalances keys
- New node joins and receives keys
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of node failure
- Replace failed hardware
- Review node capacity

**Cascading failure prevention:**
- Key rebalancing prevents data loss
- Database fallback ensures availability
- Circuit breakers prevent retry storms

### Database Connection Pool Configuration

**Redis Connection Pool (Jedis):**

```yaml
redis:
  jedis:
    pool:
      max-active: 100            # Max connections per instance
      max-idle: 50                # Max idle connections
      min-idle: 20                # Min idle connections
      
    # Timeouts
    connection-timeout: 2000      # 2s - Max time to get connection
    socket-timeout: 3000          # 3s - Max time for operation
```

**Pool Sizing Calculation:**
```
For cache service:
- 8 CPU cores per pod
- Very high read/write rate (cache operations)
- Calculated: 100 connections per pod
- 10 pods × 100 = 1,000 max connections
- Redis cluster: 6 nodes, 200 connections per node capacity
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

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total | Notes |
|-----------|--------------|------------|-------|
| Compute (EC2) | $8,000 | 76% | 10 nodes, 64 GB RAM each |
| Memory (EBS) | $2,000 | 19% | 1 TB total |
| Network | $500 | 5% | Data transfer |
| Monitoring | $200 | 2% | Prometheus, Grafana |
| Misc (10%) | $1,000 | 10% | Support, overhead |
| **Total** | **~$11,700** | 100% | |

### Cost Optimization Strategies

**Current Monthly Cost:** $8,000
**Target Monthly Cost:** $5,600 (30% reduction)

**Top 3 Cost Drivers:**
1. **Redis (ElastiCache):** $5,000/month (63%) - Largest single component
2. **Compute (EC2):** $2,000/month (25%) - Application servers
3. **Network (Data Transfer):** $1,000/month (13%) - Data transfer costs

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances
   - **Optimization:** 
     - Reserved instances for baseline (80% of compute)
     - Spot instances: 60% savings (for non-critical workloads)
   - **Savings:** $800/month (40% of $2K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Memory Optimization (20% savings on Redis):**
   - **Current:** Uncompressed data, long TTLs
   - **Optimization:** 
     - Data compression: 30% reduction
     - Aggressive TTL: Free memory faster
   - **Savings:** $1,000/month (20% of $5K Redis)
   - **Trade-off:** Slight CPU overhead for compression (acceptable)

3. **Connection Pooling (10% savings):**
   - **Current:** High connection overhead
   - **Optimization:** 
     - Reduce connection overhead
     - Reuse connections
   - **Savings:** $200/month (10% of $2K compute)
   - **Trade-off:** Slight complexity increase (acceptable)

4. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $400/month (20% of $2K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $2,400/month (30% reduction)
**Optimized Monthly Cost:** $5,600/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Cache GET | $0.000000045 | $0.000000032 | 29% |
| Cache SET | $0.00000005 | $0.000000035 | 30% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Memory Optimization → $1.8K savings
2. **Phase 2 (Month 2):** Connection Pooling, Right-sizing → $600 savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor cache hit rates (target >80%)
- Monitor Redis memory usage (target < 80%)
- Review and adjust quarterly

### Cost per Request

```
Monthly requests: 100K QPS × 86,400 × 30 = 259.2 billion requests/month
Monthly cost: $11,700

Cost per request = $11,700 / 259.2B
                 = $0.000000045
                 = $0.045 per million requests
```

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Scaling | Consistent hashing | 100K QPS |
| Reliability | Replication + failover | 99.99% availability |
| Monitoring | Prometheus + Grafana | > 80% hit rate |
| Security | TLS, AUTH | Zero breaches |
| Cost | $11.7K/month | $0.045 per million requests |
