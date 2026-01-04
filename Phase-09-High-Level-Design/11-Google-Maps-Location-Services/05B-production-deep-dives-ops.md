# Google Maps / Location Services - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Geographic Load Balancing:**
- Route users to nearest region
- Reduce latency for global users
- Distribute load across regions

**Application Load Balancing:**
- Round-robin for stateless services
- Sticky sessions for stateful services
- Health check-based routing

### Rate Limiting Implementation

**Per-User Rate Limiting:**
```java
@Service
public class RateLimiterService {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean isAllowed(String userId, String endpoint) {
        String key = String.format("rate:%s:%s", userId, endpoint);
        Long count = redis.opsForValue().increment(key);
        
        if (count == 1) {
            redis.expire(key, Duration.ofMinutes(1));
        }
        
        int limit = getLimitForTier(userId);
        return count <= limit;
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| Search Service | Yes | Return cached results |
| Routing Service | Yes | Return cached route |
| Geocoding Service | Yes | Return cached geocode |
| Traffic Service | Yes | Use historical averages |

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Search query | 200ms | 1 | None |
| Route calculation | 500ms | 1 | None |
| Geocoding | 300ms | 2 | 50ms |
| Traffic update | 100ms | 3 | Exponential |

### Replication and Failover

**Database Replication:**
- Primary: 1 per shard (writes)
- Replicas: 2-3 per shard (reads)
- Automatic failover (30s RTO)

**Service Replication:**
- Multiple instances per service
- Health check-based routing
- Automatic restart on failure

### Database Connection Pool Configuration

**PostgreSQL Connection Pool (HikariCP):**

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
For location services:
- 8 CPU cores per pod
- High read rate (search, routing queries)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin: 25 connections per pod
- 100 pods × 25 = 2,500 max connections to database
- Database max_connections: 3,000 (20% headroom)
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

3. **Connection Pooler (PgBouncer):**
   - Place PgBouncer between app and PostgreSQL
   - PgBouncer pool: 150 connections
   - App pools can share PgBouncer connections efficiently

### Backup and Recovery

**RPO (Recovery Point Objective):** < 5 minutes
- Maximum acceptable data loss
- Based on database replication lag (5 minutes max)
- Location data replicated to read replicas within 5 minutes
- Search index updates replicated within 5 minutes

**RTO (Recovery Time Objective):** < 10 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes database failover, cache warm-up, and traffic rerouting

**Backup Strategy:**
- **Database Backups**: Daily full backups, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to secondary region
- **Redis Persistence**: AOF (Append-Only File) with 1-second fsync
- **Search Index Backups**: Hourly snapshots

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote database replica to primary - 2 minutes
3. Update DNS records to point to secondary region - 1 minute
4. Warm up Redis cache from database - 3 minutes
5. Restore search index from latest snapshot - 2 minutes
6. Verify location service health and resume traffic - 1 minute

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 5 minutes
- [ ] Traffic data replication lag < 5 minutes
- [ ] CDN origin failover configured
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture search and routing success rate
   - Document active POI count
   - Verify database replication lag (< 5 minutes)
   - Test location services (100 sample requests)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+10 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-6 min:** Update CDN origin to point to DR region
   - **T+6-8 min:** Warm up cache from database
   - **T+8-9 min:** Verify all services healthy in DR region
   - **T+9-10 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+10 to T+20 minutes):**
   - Verify RTO < 10 minutes: ✅ PASS/FAIL
   - Verify RPO < 5 minutes: Check database replication lag at failure time
   - Test search: Perform 1,000 test searches, verify >99% success
   - Test routing: Calculate 1,000 test routes, verify >99% success
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify location services: Check all services functional

5. **Data Integrity Verification:**
   - Compare POI count: Pre-failover vs post-failover (should match within 5 min)
   - Spot check: Verify 100 random POIs accessible
   - Check traffic data: Verify traffic data matches pre-failover
   - Test edge cases: Remote locations, complex routes, real-time updates

6. **Failback Procedure (T+20 to T+25 minutes):**
   - Restore primary region services
   - Sync data from DR to primary
   - Verify replication lag < 5 minutes
   - Update DNS and CDN to route traffic back to primary
   - Monitor for 5 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 10 minutes: Time from failure to service resumption
- ✅ RPO < 5 minutes: Maximum data loss (verified via database replication lag)
- ✅ Search and routing work: >99% requests successful
- ✅ No data loss: POI count matches pre-failover (within 5 min window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Location services functional: All services operational

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Searches Location and Gets Route

**Step-by-step:**

1. **User Action**: User searches "Starbucks near me" via `GET /v1/search?q=Starbucks&lat=37.7749&lng=-122.4194`
2. **Location Service**: 
   - Checks Redis cache: `GET search:Starbucks:37.7749:-122.4194` → MISS
   - Queries Elasticsearch: `GET /pois/_search {"query": {"bool": {"must": [{"match": {"name": "Starbucks"}}, {"geo_distance": {"distance": "5km", "location": [37.7749, -122.4194]}}]}}}`
   - Returns top 10 results
   - Caches result: `SET search:Starbucks:37.7749:-122.4194 {results} TTL 1hour`
3. **Response**: `200 OK` with `{"results": [{"name": "Starbucks", "lat": 37.775, "lng": -122.420, ...}, ...]}`
4. **User Action**: User selects location, requests route via `GET /v1/route?origin=37.7749,-122.4194&destination=37.775,-122.420`
5. **Route Service**:
   - Checks Redis cache: `GET route:37.7749:-122.4194:37.775:-122.420` → MISS
   - Queries routing engine (GraphHopper): Calculates route (500ms)
   - Returns route with waypoints
   - Caches result: `SET route:37.7749:-122.4194:37.775:-122.420 {route} TTL 1hour`
6. **Response**: `200 OK` with route data
7. **Result**: User sees route on map within 600ms

**Total latency: Search ~100ms, Route ~600ms, Total ~700ms**

### Journey 2: Real-time Traffic Updates

**Step-by-step:**

1. **Background Job**: Updates traffic data every 5 minutes
   - Fetches traffic data from external APIs
   - Updates Redis: `SET traffic:segment_12345 {"speed": 45, "timestamp": ...}`
2. **User Action**: User requests route with traffic
3. **Route Service**:
   - Calculates base route
   - Fetches traffic data: `MGET traffic:segment_12345, traffic:segment_67890, ...`
   - Adjusts route based on traffic (avoids slow segments)
   - Returns optimized route
4. **Response**: `200 OK` with route including traffic-aware ETA
5. **Result**: User gets fastest route based on current traffic

**Total latency: ~800ms** (route calculation + traffic lookup)

### Failure & Recovery Walkthrough

**Scenario: Elasticsearch Cluster Degradation**

**RTO (Recovery Time Objective):** < 5 minutes (cluster rebalancing + replica promotion)  
**RPO (Recovery Point Objective):** 0 (replication factor 3, no data loss)

**Timeline:**

```
T+0s:    Elasticsearch node 3 crashes (handles shards 5-7)
T+0-10s: Search queries to shards 5-7 fail
T+10s:   Cluster detects node failure
T+15s:   Replicas promoted to primary for shards 5-7
T+20s:   Cluster rebalances, redistributes shards
T+30s:   All queries succeeding again
T+2min:  New node provisioned, begins recovery
T+5min:  New node fully synced, rejoins cluster
```

**What degrades:**
- Search queries to shards 5-7 delayed by 10-30 seconds
- Fallback to cached results (if available)
- Route calculations unaffected (different service)

**What stays up:**
- Search queries to other shards
- Route calculations
- Map tile serving (CDN)
- Cached search results

**What recovers automatically:**
- Elasticsearch cluster promotes replicas
- Cluster rebalances automatically
- New node streams and rejoins
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of node failure
- Replace failed hardware
- Review cluster capacity

**Cascading failure prevention:**
- Replication factor 3 prevents data loss
- Cluster rebalancing distributes load
- Circuit breakers prevent retry storms
- Fallback to cached results

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Search      │  │ Routing     │  │ API        │           │  │
│  │  │ Service     │  │ Service     │  │ Service     │           │  │
│  │  │ (20 pods)   │  │ (30 pods)   │  │ (40 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  PostgreSQL (POI data, routes) - Sharded        │          │  │
│  │  │  ──async repl──> PostgreSQL Replica (DR)         │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (cache) - 20 nodes                       │ │  │
│  │  │  ──async repl──> Redis Replica (DR)                     │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  S3 Bucket (map tiles)                                    │ │  │
│  │  │  ──CRR──> S3 Bucket (DR region)                          │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  CDN (CloudFront) - Global edge network                   │ │  │
│  │  │  Origin: us-east-1 S3 (primary)                           │ │  │
│  │  │  Failover: us-west-2 S3 (DR)                             │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async replication──                            │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: us-west-2                                │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  PostgreSQL Replica (async replication from us-east-1)            │ │
│  │  Redis Replica (async replication + snapshots)                   │ │
│  │  S3 Bucket (CRR destination, read-only until failover)            │ │
│  │  Search Service (5 pods, minimal for DR readiness)                │ │
│  │  Routing Service (5 pods, minimal for DR readiness)                 │ │
│  │  API Service (5 pods, minimal for DR readiness)                  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 5 minutes for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **S3 Cross-Region Replication (CRR):**
   - **Primary:** us-east-1 maintains map tiles storage
   - **Replication:** Async replication to DR region (us-west-2)
   - **Replication Lag:** < 5 minutes (acceptable for tiles, RPO < 5 min)
   - **Conflict Resolution:** Not applicable (single-writer model)

3. **Redis Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Snapshots:** Every 5 minutes to S3 for faster recovery
   - **Cache Warm-up:** On failover, restore from latest snapshot

4. **CDN Origin Failover:**
   - **Primary:** CloudFront origin points to us-east-1 S3
   - **Failover:** On primary failure, CloudFront automatically fails over to us-west-2 S3
   - **RTO:** < 30 seconds (automatic CDN failover)

5. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Route users to nearest region for lower latency

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Replication Lag** | Stale POI/traffic data in DR region | Acceptable for DR (RPO < 5 min), popular data cached in CDN |
| **CDN Cache** | CDN cache may be cold after failover | Acceptable for DR (cache warms up quickly), popular tiles already cached |
| **Traffic Data** | Real-time traffic data may be stale | Acceptable for DR (5 min staleness), traffic data updates frequently |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR is read-only until failover |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Data Consistency** | Eventual consistency across regions | Acceptable for location services (slight staleness), strong consistency within region |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle traffic)
   - Recovery: Automatic (Kubernetes restarts pod)
   - RTO: < 2 minutes (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 10 minutes (manual process)
   - RPO: < 5 minutes (async replication lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale data
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $75,400,000/month (full infrastructure, including Map APIs)
- **DR Region:** $22,620,000/month (reduced capacity, 30% of primary)
- **S3 CRR:** $1,000/month (cross-region replication)
- **Replication Bandwidth:** $2,000/month (cross-region data transfer)
- **Total Multi-Region Cost:** $98,023,000/month (30% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** S3 CRR provides additional backup layer

**Trade-offs:**
- **Cost:** 30% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)
- **Consistency:** Eventual consistency (acceptable for location services, slight staleness)

---

## 2. Monitoring & Observability

### Key Metrics

**Performance Metrics:**
- Search latency (p50, p95, p99)
- Route calculation latency
- Cache hit rates
- Error rates

**Business Metrics:**
- Requests per second
- Active users
- Popular searches
- Route success rate

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High search latency | p95 > 300ms | Warning | Investigate |
| Route calculation failure | > 1% | Critical | Page on-call |
| Cache hit rate drop | < 70% | Warning | Check cache |
| Traffic service down | 100% failure | Critical | Failover |

---

## 3. Security Considerations

### Input Validation

**Coordinate Validation:**
```java
public boolean isValidCoordinate(double lat, double lng) {
    return lat >= -90 && lat <= 90 && 
           lng >= -180 && lng <= 180;
}
```

**Query Sanitization:**
- Prevent SQL injection
- Sanitize user queries
- Limit query length

### Privacy

**Location Data:**
- Anonymize user locations
- Encrypt at rest
- 90-day retention
- User consent required

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Search and Route

```
1. User searches "coffee near me"
   ↓
2. Check cache → MISS
   ↓
3. Query PostgreSQL with PostGIS
   ↓
4. Return 10 results
   ↓
5. Cache results (1h TTL)
   ↓
6. User selects location
   ↓
7. Request route from current location
   ↓
8. Check route cache → MISS
   ↓
9. Calculate route with A* algorithm
   ↓
10. Apply traffic data
   ↓
11. Return route with ETA
   ↓
12. Cache route (1h TTL)
```

### Journey 2: Real-time Navigation

```
1. User starts navigation
   ↓
2. WebSocket connection established
   ↓
3. Send initial route
   ↓
4. User location updates every 5 seconds
   ↓
5. Check if off-route
   ↓
6. If off-route: Recalculate route
   ↓
7. Traffic updates every 30 seconds
   ↓
8. Update ETA based on traffic
   ↓
9. Send turn-by-turn directions
   ↓
10. Continue until destination reached
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (EC2) | $19,620 | 26% |
| Storage (S3) | $7,505 | 10% |
| Map APIs | $25,100,000 | 33% |
| CDN | $825 | 1% |
| Database (RDS) | $4,500 | 6% |
| Network | $500 | 1% |
| **Total** | **~$75.4M** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $75,400,000
**Target Monthly Cost:** $60,000,000 (20% reduction)

**Top 3 Cost Drivers:**
1. **Map APIs:** $25,100,000/month (33%) - Largest single component (external API costs)
2. **Compute (EC2):** $19,620/month (0.03%) - Application servers
3. **Storage (S3):** $7,505/month (0.01%) - Map tiles and data storage

**Note:** The majority of costs are external Map API costs, which are harder to optimize directly.

**Optimization Strategies (Ranked by Impact):**

1. **Aggressive Caching (80% reduction in API calls):**
   - **Current:** High number of external Map API calls
   - **Optimization:** 
     - Cache search results (TTL: 1 hour)
     - Cache route calculations (TTL: 5 minutes)
     - Cache POI data (TTL: 24 hours)
     - Cache map tiles (TTL: 7 days)
   - **Savings:** $20,080,000/month (80% of $25.1M Map APIs)
   - **Trade-off:** Slight staleness (acceptable for most use cases)

2. **CDN for Tiles (50% savings on bandwidth):**
   - **Current:** Direct S3 access for map tiles
   - **Optimization:** 
     - Use CloudFront CDN for map tiles
     - Reduce bandwidth costs by 50%
   - **Savings:** $3,752/month (50% of $7.5K storage bandwidth)
   - **Trade-off:** Slight complexity increase (acceptable)

3. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand instances for application servers
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $7,848/month (40% of $19.6K compute)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

4. **Data Compression (30% savings on storage):**
   - **Current:** Uncompressed or basic compression
   - **Optimization:** 
     - Compress POI data (reduce size by 30%)
     - Optimize map tile compression
   - **Savings:** $2,251/month (30% of $7.5K storage)
   - **Trade-off:** Slight CPU overhead (negligible)

5. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
   - **Savings:** $1,962/month (10% of $19.6K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $20,095,813/month (27% reduction)
**Optimized Monthly Cost:** $55,304,187/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Search Request | $0.0025 | $0.0005 | 80% |
| Route Calculation | $0.0025 | $0.0005 | 80% |
| Map Tile (per 1000) | $0.01 | $0.005 | 50% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Aggressive Caching → $20.08M savings
2. **Phase 2 (Month 2):** CDN, Reserved Instances → $11.6K savings
3. **Phase 3 (Month 3):** Data Compression, Right-sizing → $4.2K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor cache hit rates (target >80%)
- Monitor API call reduction (target 80% reduction)
- Review and adjust quarterly

**Note:** The largest cost optimization opportunity is reducing external Map API calls through aggressive caching, which can save $20M/month.

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | Geographic + Application | < 50ms latency |
| Rate Limiting | Per-user, Redis-based | 1000 req/min (pro) |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | p95 < 200ms |
| Security | Input validation, encryption | Zero breaches |
| Cost | $75M/month | $2.50 per 1000 requests |


