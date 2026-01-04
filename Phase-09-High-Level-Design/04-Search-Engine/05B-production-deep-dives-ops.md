# Search Engine - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Query Path:**

```
Internet → CDN (CloudFront) → ALB → Query Coordinators → Index Shards
```

**Load Balancing Algorithm:** Least connections for query coordinators

```yaml
# ALB Target Group Configuration
target_group:
  protocol: HTTP
  port: 8080
  health_check:
    protocol: HTTP
    path: /health
    interval: 10s
    timeout: 5s
    healthy_threshold: 2
    unhealthy_threshold: 2
  stickiness:
    enabled: false  # Queries are stateless
  load_balancing_algorithm: least_outstanding_requests
```

### Rate Limiting Implementation

**Algorithm: Token Bucket per User/IP**

```java
@Component
public class SearchRateLimiter {
    
    private final RedisTemplate<String, Long> redis;
    
    public boolean isAllowed(String clientId, int tier) {
        String key = "ratelimit:search:" + clientId;
        int limit = getLimitForTier(tier);
        
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, Duration.ofSeconds(1));
        }
        
        return count <= limit;
    }
    
    private int getLimitForTier(int tier) {
        switch (tier) {
            case 0: return 1;    // Free: 1 QPS
            case 1: return 10;   // Developer: 10 QPS
            case 2: return 100;  // Business: 100 QPS
            default: return 1000; // Enterprise: 1000 QPS
        }
    }
}
```

**Rate Limit Tiers:**

| Tier | Queries/sec | Suggestions/sec | Daily Limit |
|------|-------------|-----------------|-------------|
| Free | 1 | 10 | 1,000 |
| Developer | 10 | 100 | 100,000 |
| Business | 100 | 1,000 | 10,000,000 |
| Enterprise | Custom | Custom | Unlimited |

### Circuit Breaker Placement

**Services with Circuit Breakers:**

| Service | Circuit Breaker | Fallback Behavior |
|---------|-----------------|-------------------|
| Index Shard | Yes | Return partial results from healthy shards |
| Redis Cache | Yes | Query index directly (slower) |
| Spell Check | Yes | Return original query (no suggestions) |
| PageRank Store | Yes | Use default score (0.5) |

**Configuration (Resilience4j):**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      index-shard:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
      redis-cache:
        slidingWindowSize: 50
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Shard Query | 500ms | 1 | None (deadline) |
| Redis Cache | 50ms | 0 | None |
| Spell Check | 100ms | 0 | None |
| Crawl Request | 30s | 3 | 1s exponential |

### Replication and Failover

**Index Shards:**
- 100 shards, 3 replicas each = 300 shard instances
- Any replica can serve reads
- Primary/replica model for writes

**Query Coordinators:**
- 20 stateless instances
- Any coordinator can handle any query
- Auto-scaling based on CPU

**Failover Strategy:**

```java
@Service
public class ShardQueryService {
    
    private final List<ShardReplica> replicas;
    
    public ShardResult queryShardWithFailover(int shardId, Query query) {
        List<ShardReplica> shardReplicas = getReplicasForShard(shardId);
        
        // Shuffle for load distribution
        Collections.shuffle(shardReplicas);
        
        for (ShardReplica replica : shardReplicas) {
            try {
                return replica.query(query);
            } catch (Exception e) {
                log.warn("Replica {} failed, trying next", replica.getId());
            }
        }
        
        // All replicas failed
        throw new ShardUnavailableException(shardId);
    }
}
```

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: Spell suggestions
   - Impact: No "Did you mean?" suggestions
   - Trigger: Spell service slow/down

2. **Second to drop**: Synonym expansion
   - Impact: Fewer results for some queries
   - Trigger: High latency

3. **Third to drop**: Personalization
   - Impact: Generic results
   - Trigger: User service unavailable

4. **Fourth to drop**: Some shards
   - Impact: Partial results (clearly marked)
   - Trigger: Shard failures > 10%

5. **Last resort**: Return cached results only
   - Impact: Stale results
   - Trigger: > 50% shard failures

### Backup and Recovery

**RPO (Recovery Point Objective):** < 1 hour
- Maximum acceptable data loss
- Based on index replication frequency (hourly snapshots)
- Index updates replicated to secondary region within 1 hour

**RTO (Recovery Time Objective):** < 5 minutes
- Maximum acceptable downtime
- Time to restore service via failover to secondary region
- Includes DNS failover and traffic rerouting

**Backup Strategy:**
- **Index Snapshots**: Daily full snapshots, hourly incremental
- **Point-in-time Recovery**: Retention period of 7 days
- **Cross-region Replication**: Async replication to secondary region (us-west-2)
- **DR Region**: Daily snapshots to DR region (eu-west-1)

**Restore Steps:**
1. Detect primary region failure (health checks)
2. Promote secondary region to primary (2 minutes)
3. Update DNS records to point to secondary (1 minute)
4. Verify index integrity and start serving traffic (1 minute)
5. Rebuild failed shards from snapshots if needed (ongoing)

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Index replication lag < 1 hour
- [ ] Index snapshots verified in S3
- [ ] Database replication lag < 1 hour
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture index version and shard health
   - Document active document count
   - Verify cache hit rates
   - Test search accuracy (100 sample queries)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in us-east-1 (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+15 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-3 min:** Promote secondary region (us-west-2) to primary
   - **T+3-4 min:** Update DNS records (Route53 health checks)
   - **T+4-8 min:** Verify index integrity (check all shards healthy)
   - **T+8-12 min:** Warm up cache with popular queries
   - **T+12-14 min:** Verify all services healthy in DR region
   - **T+14-15 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+15 to T+30 minutes):**
   - Verify RTO < 15 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 hour: Check index replication lag at failure time
   - Test search queries: Run 1,000 sample queries, verify results match pre-failover
   - Test query performance: Verify latency < 500ms p99
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify index integrity: Check all shards, no corruption

5. **Data Integrity Verification:**
   - Compare document count: Pre-failover vs post-failover (should match within 1 hour)
   - Spot check: Verify 100 random documents searchable
   - Check index structure: Verify all shards healthy, no missing data
   - Test edge cases: Empty queries, special characters, long queries

6. **Failback Procedure (T+30 to T+45 minutes):**
   - Restore primary region services
   - Sync index from secondary to primary
   - Verify index version matches
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 15 minutes: Time from failure to service resumption
- ✅ RPO < 1 hour: Maximum data loss (verified via index replication lag)
- ✅ Search queries return correct results: >95% result accuracy
- ✅ No data loss: Document count matches pre-failover (within 1 hour window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Index integrity intact: All shards healthy, no corruption

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

### Disaster Recovery

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Query       │  │ Query       │  │ Query       │           │  │
│  │  │ Coordinator │  │ Coordinator │  │ Coordinator │           │  │
│  │  │ (20 pods)   │  │ (20 pods)   │  │ (20 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  Elasticsearch Cluster (100 shards × 3 replicas)│          │  │
│  │  │  ──async repl──> Secondary Region                │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (query cache)                             │ │  │
│  │  │  ──async repl──> Redis Replica (DR region)               │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Crawler Cluster (active, crawling web)                  │ │  │
│  │  │  Document Processors (indexing pipeline)                 │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async index replication──                       │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    SECONDARY REGION: us-west-2                        │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  Query Coordinators (10 pods) - Standby                         │ │
│  │  Elasticsearch Cluster (100 shards × 2 replicas) - Standby      │ │
│  │  Redis Replica (async replication from us-east-1)               │ │
│  │  Crawlers (standby, activated on failover)                      │ │
│  │  Route53 health checks (routes traffic on primary failure)       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: eu-west-1                               │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  Index Snapshots (daily, stored in S3)                           │ │
│  │  Cold Standby (minimal infrastructure)                           │ │
│  │  Can restore from snapshots if needed                             │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Index Replication:**
   - **Primary:** us-east-1 maintains full index (100 shards × 3 replicas)
   - **Secondary:** us-west-2 receives async index updates (100 shards × 2 replicas)
   - **Replication Method:** Elasticsearch cross-cluster replication (CCR)
   - **Replication Lag:** < 1 hour (acceptable for search, slight staleness)
   - **Conflict Resolution:** Not applicable (single-writer model, primary writes only)

2. **Cache Replication:**
   - **Redis Replication:** Async replication to secondary region
   - **Cache Warm-up:** On failover, cache populated from index queries
   - **Cache Invalidation:** Cross-region invalidation via pub/sub (optional)

3. **Traffic Routing:**
   - **Primary:** Route53 health checks route to us-east-1
   - **Failover:** On primary failure, DNS TTL (60s) routes to us-west-2
   - **Geographic Routing:** Optional - route users to nearest region

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Index Replication Lag** | Stale search results in secondary region | Acceptable for DR (RPO < 1 hour), users see recent index |
| **Large Index Size** | 120 TB index, replication takes time | Use incremental replication, compress snapshots |
| **Latency** | Higher latency for cross-region index updates | Only primary region accepts writes, DR regions read-only until failover |
| **Cost** | 2x infrastructure cost (active-passive) | Secondary region uses fewer replicas (2 vs 3), reduces cost |
| **Data Consistency** | Eventual consistency across regions | Acceptable for search (results don't need real-time accuracy) |
| **Index Corruption** | Risk during replication | Use checksums, verify index integrity on failover |

**Failover Scenarios:**

1. **Single Shard Failure:**
   - Impact: Minimal (other shards handle queries)
   - Recovery: Automatic (Elasticsearch promotes replica)
   - RTO: < 1 minute (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to secondary region
   - RTO: < 15 minutes (manual process)
   - RPO: < 1 hour (index replication lag)

3. **Network Partition:**
   - Impact: Primary region isolated, secondary region may have stale index
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $880,000/month (full infrastructure)
- **Secondary Region:** $500,000/month (reduced replicas, 57% of primary)
- **DR Region:** $50,000/month (cold standby, snapshots only)
- **Replication Bandwidth:** $30,000/month (cross-region index replication)
- **Total Multi-Region Cost:** $1,460,000/month (66% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** Index snapshots provide additional backup layer

**Trade-offs:**
- **Cost:** 66% increase for DR capability
- **Complexity:** More complex operations (index replication, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)
- **Consistency:** Eventual consistency (acceptable for search, slight staleness)

### What Breaks First Under Load

**Order of failure at 10x traffic:**

1. **Query Coordinators** (First)
   - Symptom: High latency, timeouts
   - Fix: Auto-scale, add instances

2. **Redis Cache** (Second)
   - Symptom: Cache misses spike
   - Fix: Increase cluster size

3. **Index Shards** (Third)
   - Symptom: Slow queries, high CPU
   - Fix: Add replicas, optimize queries

4. **Network** (Fourth)
   - Symptom: Packet loss between services
   - Fix: Increase bandwidth, optimize payloads

### Database Connection Pool Configuration

**Elasticsearch Connection Pool (HTTP Client):**

```yaml
elasticsearch:
  client:
    # Connection pool sizing
    max-connections: 100         # Max connections per application instance
    max-connections-per-route: 20 # Max connections per Elasticsearch node
    
    # Timeouts
    connection-timeout: 5000     # 5s - Max time to establish connection
    socket-timeout: 30000        # 30s - Max time for request/response
    
    # Keep-alive
    keep-alive: 30000            # 30s - Keep connections alive
```

**Pool Sizing Calculation:**
```
For search service:
- 8 CPU cores per pod
- High query rate (search operations)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for concurrent searches: 100 connections per pod
- 50 query coordinator pods × 100 = 5,000 max connections
- Elasticsearch cluster: 100 nodes, 50 connections per node capacity
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

**Pool Sizing Calculation:**
```
For search cache:
- 8 CPU cores per pod
- Read-heavy (cache lookups)
- Calculated: 30 connections per pod
- 50 query coordinator pods × 30 = 1,500 max connections
- Redis cluster: 6 nodes, 250 connections per node capacity
```

**PostgreSQL Connection Pool (HikariCP) - For Metadata:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # Max connections per instance
      minimum-idle: 5
      connection-timeout: 30000
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

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| P50 | < 100ms | > 150ms | > 200ms |
| P95 | < 300ms | > 400ms | > 500ms |
| P99 | < 500ms | > 700ms | > 1000ms |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Queries/sec | 10K | > 15K | > 20K |
| Cache hit rate | 60% | < 50% | < 30% |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Error rate (5xx) | < 0.1% | > 0.5% | > 1% |
| Zero-result rate | < 5% | > 8% | > 15% |
| Partial results | < 1% | > 5% | > 10% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU (coordinators) | > 60% | > 80% |
| CPU (index servers) | > 70% | > 85% |
| Memory (index servers) | > 80% | > 90% |
| Disk I/O | > 70% | > 85% |

### Metrics Collection

```java
@Component
public class SearchMetrics {
    
    private final MeterRegistry registry;
    
    // Query metrics
    private final Timer queryLatency;
    private final Counter queriesTotal;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Counter zeroResults;
    
    // Index metrics
    private final Gauge indexSize;
    private final Gauge documentsIndexed;
    private final Gauge shardHealth;
    
    // Crawler metrics
    private final Counter pagesCrawled;
    private final Counter crawlErrors;
    
    public SearchMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.queryLatency = Timer.builder("search.query.latency")
            .description("Query latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.queriesTotal = Counter.builder("search.queries.total")
            .description("Total queries")
            .register(registry);
        
        this.zeroResults = Counter.builder("search.queries.zero_results")
            .description("Queries with no results")
            .register(registry);
    }
    
    public void recordQuery(long durationMs, boolean cacheHit, int resultsCount) {
        queryLatency.record(durationMs, TimeUnit.MILLISECONDS);
        queriesTotal.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
        
        if (resultsCount == 0) {
            zeroResults.increment();
        }
    }
}
```

### Alerting Thresholds and Escalation

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High Latency P99 | > 700ms for 5 min | Warning | Ticket |
| High Latency P99 | > 1000ms for 2 min | Critical | Page on-call |
| Error Rate | > 0.5% for 5 min | Warning | Ticket |
| Error Rate | > 1% for 1 min | Critical | Page on-call |
| Cache Hit Rate | < 50% for 10 min | Warning | Ticket |
| Shard Unhealthy | > 5% shards down | Critical | Page on-call |
| Crawl Rate | < 2000/s for 30 min | Warning | Ticket |

### Logging Strategy

**Structured Logging Format:**

```java
@Slf4j
@RestController
public class SearchController {
    
    @GetMapping("/v1/search")
    public ResponseEntity<SearchResponse> search(
            @RequestParam String q,
            @RequestParam(defaultValue = "1") int page,
            HttpServletRequest request) {
        
        String requestId = UUID.randomUUID().toString();
        long startTime = System.nanoTime();
        
        try {
            SearchResponse response = searchService.search(q, page);
            
            long latencyMs = (System.nanoTime() - startTime) / 1_000_000;
            
            log.info("search.query",
                kv("request_id", requestId),
                kv("query", truncate(q, 100)),
                kv("page", page),
                kv("latency_ms", latencyMs),
                kv("results_count", response.getResults().size()),
                kv("cache_hit", response.isFromCache()),
                kv("shards_queried", response.getShardsQueried()),
                kv("client_ip_hash", hashIp(request.getRemoteAddr()))
            );
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            log.error("search.error",
                kv("request_id", requestId),
                kv("query", truncate(q, 100)),
                kv("error", e.getMessage())
            );
            throw e;
        }
    }
}
```

### Distributed Tracing

**Span Boundaries:**

```
search-request (root)
├── cache-lookup
├── query-parse
│   ├── tokenize
│   ├── spell-check
│   └── expand-synonyms
├── shard-query (parallel)
│   ├── shard-0-query
│   ├── shard-1-query
│   └── shard-N-query
├── merge-results
├── rank-results
└── cache-store
```

**Implementation:**

```java
@Service
public class SearchService {
    
    private final Tracer tracer;
    
    public SearchResponse search(String query) {
        Span rootSpan = tracer.spanBuilder("search-request")
            .setAttribute("query", query)
            .startSpan();
        
        try (Scope scope = rootSpan.makeCurrent()) {
            // Cache lookup
            Span cacheSpan = tracer.spanBuilder("cache-lookup").startSpan();
            SearchResponse cached = cache.get(query);
            cacheSpan.end();
            
            if (cached != null) {
                return cached;
            }
            
            // Query shards in parallel
            Span shardSpan = tracer.spanBuilder("shard-query").startSpan();
            List<ShardResult> results = queryAllShards(query);
            shardSpan.setAttribute("shards_queried", results.size());
            shardSpan.end();
            
            // Merge and rank
            Span rankSpan = tracer.spanBuilder("rank-results").startSpan();
            SearchResponse response = rankResults(results);
            rankSpan.end();
            
            return response;
        } finally {
            rootSpan.end();
        }
    }
}
```

### Health Checks

```java
@Component
public class SearchHealthIndicator implements HealthIndicator {
    
    private final ShardManager shardManager;
    private final RedisTemplate redis;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check shard health
        int healthyShards = shardManager.getHealthyShardCount();
        int totalShards = shardManager.getTotalShardCount();
        double shardHealth = (double) healthyShards / totalShards;
        
        details.put("healthy_shards", healthyShards);
        details.put("total_shards", totalShards);
        details.put("shard_health_pct", shardHealth * 100);
        
        // Check cache health
        try {
            redis.opsForValue().get("health-check");
            details.put("cache_status", "UP");
        } catch (Exception e) {
            details.put("cache_status", "DOWN");
        }
        
        // Determine overall health
        if (shardHealth < 0.9) {
            return Health.down()
                .withDetails(details)
                .withDetail("error", "Too many unhealthy shards")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

---

## 3. Security Considerations

### Authentication Mechanism

**API Key Authentication:**

```java
@Component
public class ApiKeyAuthFilter extends OncePerRequestFilter {
    
    private final ApiKeyService apiKeyService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) {
        
        String apiKey = request.getHeader("X-API-Key");
        
        if (apiKey == null) {
            // Allow anonymous with lowest rate limit
            request.setAttribute("tier", 0);
            chain.doFilter(request, response);
            return;
        }
        
        Optional<ApiClient> client = apiKeyService.validate(apiKey);
        
        if (client.isEmpty()) {
            response.setStatus(401);
            response.getWriter().write("{\"error\": \"Invalid API key\"}");
            return;
        }
        
        request.setAttribute("client", client.get());
        request.setAttribute("tier", client.get().getTier());
        chain.doFilter(request, response);
    }
}
```

### Input Validation and Sanitization

```java
@Service
public class QueryValidator {
    
    private static final int MAX_QUERY_LENGTH = 500;
    private static final Pattern DANGEROUS_CHARS = Pattern.compile("[<>\"'&]");
    
    public ValidationResult validate(String query) {
        // Length check
        if (query == null || query.isEmpty()) {
            return ValidationResult.invalid("Query required");
        }
        
        if (query.length() > MAX_QUERY_LENGTH) {
            return ValidationResult.invalid("Query too long");
        }
        
        // Sanitize dangerous characters
        if (DANGEROUS_CHARS.matcher(query).find()) {
            query = sanitize(query);
        }
        
        return ValidationResult.valid(query);
    }
    
    private String sanitize(String query) {
        return query
            .replace("<", "")
            .replace(">", "")
            .replace("\"", "")
            .replace("'", "")
            .replace("&", "");
    }
}
```

### PII Handling and Compliance

**Query Anonymization:**

```java
@Service
public class QueryAnonymizer {
    
    private static final Pattern EMAIL = Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    private static final Pattern PHONE = Pattern.compile("\\d{3}[-.]?\\d{3}[-.]?\\d{4}");
    
    public String anonymizeForLogging(String query) {
        String anonymized = query;
        
        // Replace emails
        anonymized = EMAIL.matcher(anonymized).replaceAll("[EMAIL]");
        
        // Replace phone numbers
        anonymized = PHONE.matcher(anonymized).replaceAll("[PHONE]");
        
        return anonymized;
    }
}
```

**GDPR Compliance:**
- Query logs anonymized after 30 days
- User can request data deletion
- No PII in search results cache

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Query Injection** | Input validation, no SQL execution |
| **DDoS** | Rate limiting, CDN protection |
| **Data Scraping** | Rate limits, CAPTCHA for suspicious patterns |
| **Index Poisoning** | Crawl only trusted sources, spam detection |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Searches "best pizza NYC"

**Step-by-step:**

1. **User Action**: Types query, hits Enter
2. **CDN**: Cache check → MISS (personalized query)
3. **Load Balancer**: Routes to Query Coordinator #7
4. **Coordinator**: Checks Redis cache → MISS
5. **Coordinator**: Parses query
   - Tokenize: ["best", "pizza", "nyc"]
   - Expand: ["best", "pizza", "nyc", "new york city"]
6. **Coordinator**: Broadcasts to all 100 shards
7. **Shards**: Each returns top 20 results (parallel, 150ms)
8. **Coordinator**: Merges 2000 results
9. **Coordinator**: Re-ranks with PageRank, freshness
10. **Coordinator**: Returns top 10
11. **Coordinator**: Caches in Redis (1 hour TTL)
12. **Response**: 200 OK with results

**Total latency: 250ms**

### Journey 2: Index Update After Crawl

**Step-by-step:**

1. **Crawler**: Downloads https://newsite.com/pizza
2. **Crawler**: Produces to Kafka `crawled-pages`
3. **Document Processor**: Consumes message
4. **Processor**: Parses HTML, extracts:
   - Title: "Best Pizza Guide"
   - Content: 1500 words
   - Links: 45 outbound
5. **Processor**: Produces to `processed-documents`
6. **Indexer**: Consumes message
7. **Indexer**: Tokenizes content
8. **Indexer**: Updates posting lists for terms
9. **Indexer**: Updates document metadata in PostgreSQL
10. **Indexer**: Invalidates affected cache entries

**Total time: ~30 seconds from crawl to searchable**

### Failure & Recovery Walkthrough

**Scenario: Index Shard Failure**

**RTO (Recovery Time Objective):** < 5 minutes (replica promotion + reindexing)  
**RPO (Recovery Point Objective):** < 1 hour (index replication lag)

**Timeline:**

```
T+0s:    Shard 42 primary crashes (disk failure)
T+5s:    Health check fails
T+10s:   Coordinator marks shard 42 as unhealthy
T+10s:   Queries route to replica 42-A
T+15s:   Kubernetes detects pod failure
T+30s:   New pod scheduled
T+60s:   Pod starts, begins loading index
T+300s:  Index loaded (from S3 backup)
T+310s:  New replica joins cluster
T+320s:  Shard 42 fully healthy
```

**What degrades:**
- Queries to shard 42 slightly slower (fewer replicas)
- No data loss (replicas have full copy)

**What stays up:**
- All queries still work
- Results complete (other replicas serve)

**Runbook Steps:**

1. **Detection (T+0s to T+10s):**
   - Monitor: Grafana alert "Index shard unhealthy"
   - Alert: PagerDuty alert triggered at T+5s
   - Verify: Check shard health endpoint `/health/shard/42`
   - Confirm: Shard 42 primary node down

2. **Immediate Response (T+10s to T+30s):**
   - Action: Coordinator automatically routes queries to replica 42-A
   - Fallback: All 100 shards continue serving (shard 42 uses replica)
   - Impact: Slight latency increase for shard 42 queries (acceptable)
   - Monitor: Verify query success rate remains > 99%

3. **Recovery (T+30s to T+320s):**
   - T+15s: Kubernetes detects pod failure
   - T+30s: New pod scheduled on healthy node
   - T+60s: Pod starts, begins loading index from S3 backup
   - T+300s: Index fully loaded (10M documents)
   - T+310s: New replica joins cluster, syncs with primary
   - T+320s: Shard 42 fully healthy (primary + 2 replicas)

4. **Post-Recovery (T+320s to T+600s):**
   - Monitor: Watch shard health for 5 minutes
   - Verify: Query latency returns to normal
   - Document: Log incident, root cause analysis
   - Follow-up: Investigate disk failure cause

**Prevention:**
- Regular disk health monitoring
- Proactive disk replacement before failure
- Multi-AZ deployment for redundancy

### Failure Scenario: Redis Cache Down

**Detection:**
- How to detect: Grafana alert "Redis cluster down"
- Alert thresholds: 3 consecutive health check failures
- Monitoring: Check Redis cluster status endpoint

**Impact:**
- Affected services: All search coordinators (cache miss rate 100%)
- User impact: Latency increases from 50ms to 200ms
- Degradation: System continues operating, slower responses

**Recovery Steps:**
1. **T+0s**: Alert received, on-call engineer paged
2. **T+30s**: Verify Redis cluster status via AWS console
3. **T+60s**: Check Redis primary node health
4. **T+90s**: If primary down, promote replica to primary (automatic)
5. **T+120s**: Verify Redis cluster healthy
6. **T+150s**: Coordinators reconnect to Redis
7. **T+180s**: Cache begins warming up (popular queries cached first)
8. **T+600s**: Cache hit rate returns to normal (80%)

**RTO:** < 5 minutes
**RPO:** 0 (no data loss, cache rebuilds from index)

**Prevention:**
- Redis cluster with automatic failover
- Health checks every 10 seconds
- Circuit breaker falls back to index on Redis failure

### Failure Scenario: Database Primary Failure

**Detection:**
- How to detect: Database connection errors in logs
- Alert thresholds: > 10% error rate for 30 seconds
- Monitoring: PostgreSQL primary health check endpoint

**Impact:**
- Affected services: Document metadata updates (can't write new data)
- User impact: Search continues (read-only from index)
- Degradation: New documents not searchable until recovery

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
**RPO:** < 1 hour (index replication lag)

**Prevention:**
- Multi-AZ deployment with synchronous replication
- Automated failover enabled
- Regular failover testing (quarterly)

### Failure Scenario: Kafka Lag (Async Processing)

**Detection:**
- How to detect: Grafana alert "Kafka consumer lag > 1M messages"
- Alert thresholds: Lag > 1M messages for 5 minutes
- Monitoring: Check Kafka consumer lag metrics

**Impact:**
- Affected services: Document processing pipeline
- User impact: New documents not searchable (delayed indexing)
- Degradation: System continues serving existing indexed content

**Recovery Steps:**
1. **T+0s**: Alert received, Kafka lag detected
2. **T+30s**: Check consumer group status, identify slow consumers
3. **T+60s**: Scale up document processors (add 10 more pods)
4. **T+120s**: New pods join consumer group, begin processing
5. **T+300s**: Lag begins decreasing
6. **T+600s**: Lag returns to normal (< 10K messages)

**RTO:** < 10 minutes
**RPO:** 0 (messages in Kafka, no loss)

**Prevention:**
- Auto-scaling based on Kafka lag
- Monitor consumer processing rate
- Alert on lag thresholds

### High-Load / Contention Scenario: Popular Query Storm

```
Scenario: Viral event causes millions to search for same query
Time: Major news event (e.g., "Olympics results", "election")

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Breaking news triggers 100K queries/second            │
│ - Query: "election results" (trending)                      │
│ - Expected: ~1K queries/second normal traffic              │
│ - 100x traffic spike                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-30s: Cache Miss Storm                                  │
│ - Redis cache: MISS (query not cached, first time trending)│
│ - All 100K queries/second hit coordinators                 │
│ - Coordinators broadcast to all 100 shards                  │
│ - 100K × 100 shards = 10M shard queries/second             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Protection Mechanisms:                                      │
│                                                              │
│ 1. Redis Cache Warming:                                    │
│    - First coordinator processes query                     │
│    - Caches result in Redis (1 hour TTL)                   │
│    - Next 99,999 queries: Redis HIT                        │
│    - Cache hit rate: 99.99% after first query              │
│                                                              │
│ 2. Query Deduplication:                                    │
│    - Coordinators dedupe concurrent identical queries      │
│    - Only 1 query per coordinator sent to shards           │
│    - Reduces shard load by 100x                            │
│                                                              │
│ 3. Shard Load Distribution:                                │
│    - Parallel query to all shards (150ms)                  │
│    - Each shard handles ~1K queries/second                │
│    - Well within shard capacity (10K req/sec)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                      │
│ - First query: 250ms (cache miss, shard query)            │
│ - Next 99,999 queries: 5-10ms (Redis cache hit)           │
│ - Shard load: 1K req/sec (safe)                            │
│ - P99 latency: < 100ms (well under 500ms target)          │
│ - No service degradation                                   │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Query with Special Characters and Injection Attempt

```
Scenario: User submits query with SQL injection attempt mixed with valid search
Edge Case: Security filtering must handle malicious input gracefully

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User submits malicious query                        │
│ - Query: "pizza'; DROP TABLE users; --"                   │
│ - Attempting SQL injection (though system uses NoSQL)      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Input Validation                                    │
│ - Query received: "pizza'; DROP TABLE users; --"          │
│ - Length check: PASS (< 200 chars)                         │
│ - Character validation: FAIL (contains ';' and '--')      │
│ - Returns: 400 Bad Request "Invalid characters"            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Alternative Attack - XSS Attempt                   │
│ - Query: "pizza<script>alert('xss')</script>"             │
│ - Character validation: FAIL (contains '<' and '>')        │
│ - Returns: 400 Bad Request "Invalid characters"            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Valid Query with Special Characters                │
│ - Query: "c++ programming" (valid, contains '+')          │
│ - Character validation: PASS (whitelist allows alphanumeric + spaces)│
│ - Normalize: "c++ programming" → "c programming" (strip +)│
│ - Trie/Index lookup: find("c programming")                │
│ - Returns: Valid suggestions                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Valid Query That Looks Suspicious               │
│ - Query: "c++" (legitimate programming language query)     │
│ - Character validation: Currently FAIL (strips '+')        │
│ - Problem: Valid queries rejected                          │
│                                                              │
│ Solution: Context-Aware Validation                          │
│ - Whitelist known valid patterns: "c++", "c#", "f#"       │
│ - Allow alphanumeric + spaces + specific symbols           │
│ - Normalize carefully (preserve programming terms)         │
│                                                              │
│ Result:                                                     │
│ - Malicious queries rejected (security maintained)         │
│ - Valid queries accepted (good UX)                         │
│ - No false positives for legitimate searches               │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Index Servers (90 × i3.4xlarge) | $400,000 | 45% |
| Query Coordinators (20 × c5.4xlarge) | $50,000 | 6% |
| Redis Cluster (10 × r5.2xlarge) | $30,000 | 3% |
| Crawler Cluster (40 × c5.2xlarge) | $40,000 | 5% |
| Document Processors (80 × c5.4xlarge) | $80,000 | 9% |
| Storage (S3 + EBS) | $100,000 | 11% |
| Network (data transfer) | $80,000 | 9% |
| CDN (CloudFront) | $50,000 | 6% |
| Batch Processing (EMR) | $40,000 | 5% |
| Other (monitoring, etc.) | $10,000 | 1% |
| **Total** | **$880,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $880,000
**Target Monthly Cost:** $600,000 (32% reduction)

**Top 3 Cost Drivers:**
1. **Index Servers (EC2):** $400,000/month (45%) - Largest single component
2. **Storage (S3 + EBS):** $100,000/month (11%) - Large index storage
3. **Network (Data Transfer):** $80,000/month (9%) - High data transfer volume

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand i3.4xlarge instances (90 × $4,444 = $400K)
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $160,000/month (40% of $400K)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Index Compression & Optimization:**
   - **Current:** 120 TB compressed index
   - **Optimization:** 
     - Better compression algorithms (reduce size by 25%)
     - Remove low-value terms (reduce index by 10%)
     - Use smaller instance types (i3.2xlarge sufficient after optimization)
   - **Savings:** $100,000/month (25% of $400K index servers)
   - **Trade-off:** Slight query performance impact (acceptable, still < 500ms)

3. **Spot Instances for Non-Critical Workloads:**
   - **Current:** On-demand instances for crawlers and batch jobs
   - **Optimization:** Use Spot instances (70% savings)
   - **Savings:** $56,000/month (70% of $80K crawlers+batch)
   - **Trade-off:** Possible interruptions (acceptable for background jobs)

4. **Cache Optimization (30% savings on query coordinators):**
   - **Current:** 60% cache hit rate
   - **Optimization:** 
     - Increase cache TTL for popular queries
     - Implement cache warming for top 100K queries
     - Use predictive caching
   - **Savings:** $15,000/month (30% of $50K query coordinators)
   - **Trade-off:** Slightly stale results (acceptable for search)

5. **Storage Optimization:**
   - **Current:** All data in standard storage
   - **Optimization:** 
     - Move old index snapshots to S3 Glacier (80% cheaper)
     - Use S3 Intelligent-Tiering for index storage
   - **Savings:** $20,000/month (20% of $100K storage)
   - **Trade-off:** Slower access to old snapshots (acceptable)

6. **Network Optimization:**
   - **Current:** High cross-region data transfer costs
   - **Optimization:** 
     - Compress index updates before transfer
     - Reduce replication frequency (hourly → every 2 hours)
   - **Savings:** $24,000/month (30% of $80K network)
   - **Trade-off:** Slightly stale index in secondary region (acceptable)

**Total Potential Savings:** $375,000/month (43% reduction)
**Optimized Monthly Cost:** $505,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Search Query | $0.000034 | $0.000019 | 44% |
| Index Update | $0.0001 | $0.00006 | 40% |
| Document Crawl | $0.00005 | $0.000015 | 70% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Index Compression → $260K savings
2. **Phase 2 (Month 2):** Spot instances, Cache optimization → $71K savings
3. **Phase 3 (Month 3):** Storage optimization, Network optimization → $44K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor index size (target < 90 TB after compression)
- Monitor cache hit rates (target >75%)
- Monitor query performance (ensure latency < 500ms p99)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Inverted Index | Build | Core IP, need full control |
| Crawler | Build | Custom politeness, priority |
| Query Processing | Build | Core ranking algorithms |
| Cache (Redis) | Buy (ElastiCache) | Managed, reliable |
| Message Queue | Buy (MSK) | Managed Kafka |
| CDN | Buy (CloudFront) | Global edge network |

### Cost per Query Estimate

**Assumptions:**
- 10K queries/second = 26 billion/month
- Monthly cost: $880,000

**Calculation:**
- Cost per query: $880,000 / 26B = $0.000034
- Cost per 1000 queries: $0.034

### At What Scale Does This Become Expensive?

| Scale | Monthly Cost | Cost per 1M queries |
|-------|--------------|---------------------|
| 10K QPS | $880,000 | $34 |
| 50K QPS | $3,000,000 | $23 |
| 100K QPS | $5,000,000 | $19 |

**Economies of scale**: Cost per query decreases due to:
- Better cache hit rates at scale
- More efficient infrastructure utilization
- Volume discounts from cloud providers

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | ALB + least connections | < 1ms added latency |
| Rate Limiting | Redis token bucket | Tier-based limits |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 500ms |
| Security | API keys + input validation | No injection vulnerabilities |
| DR | Multi-region active-passive | RTO < 15 min |
| Cost | $880,000/month | $0.034 per 1K queries |

