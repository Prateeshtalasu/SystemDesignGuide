# Typeahead / Autocomplete - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis. These topics ensure the typeahead system operates reliably in production at scale.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Layer 4 (TCP) Load Balancing with NLB:**

```
Internet → CloudFront (CDN) → NLB → Suggestion Servers
```

**Why NLB over ALB?**
- Lower latency (no HTTP parsing)
- Higher throughput (millions of connections)
- Simple round-robin is sufficient
- Health checks still work

**Load Balancing Algorithm:** Round-robin with health checks

```yaml
# NLB Target Group Configuration
target_group:
  protocol: TCP
  port: 8080
  health_check:
    protocol: HTTP
    path: /health
    interval: 10s
    timeout: 5s
    healthy_threshold: 2
    unhealthy_threshold: 2
  deregistration_delay: 30s
```

### Rate Limiting Implementation

**Algorithm: Token Bucket (In-Memory)**

For extreme throughput (500K QPS), Redis-based rate limiting adds too much latency. We use in-memory token bucket per server:

```java
@Component
public class InMemoryRateLimiter {
    
    private final Cache<String, AtomicLong> buckets = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.MINUTES)
        .maximumSize(1_000_000)
        .build();
    
    public boolean isAllowed(String clientId, int limit) {
        AtomicLong counter = buckets.get(clientId, k -> new AtomicLong(0));
        long current = counter.incrementAndGet();
        return current <= limit;
    }
}
```

**Rate Limit Tiers:**

| Client Type | Limit | Window | Enforcement |
|-------------|-------|--------|-------------|
| Anonymous | 100 req/min | Per IP | In-memory |
| Authenticated | 1000 req/min | Per user | In-memory |
| API Partner | 10000 req/min | Per API key | In-memory |

**Why in-memory instead of Redis?**
- Redis adds 1-5ms latency
- At 500K QPS, that's 500K extra Redis calls/second
- In-memory is approximate but good enough
- Each server enforces independently

### Circuit Breaker Placement

**Services with Circuit Breakers:**

| Service | Circuit Breaker | Fallback Behavior |
|---------|-----------------|-------------------|
| S3 (index download) | Yes | Keep serving old index |
| Zookeeper | Yes | Keep serving current index |
| Kafka (logging) | Yes | Drop log events |

**Configuration (Resilience4j):**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      s3:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      zookeeper:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      kafka:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| Trie lookup | N/A | N/A | In-memory, no timeout |
| S3 download | 60s | 3 | 5s exponential |
| Zookeeper read | 5s | 3 | 1s exponential |
| Kafka produce | 100ms | 0 | None (fire-and-forget) |

### Replication and Failover

**Suggestion Servers:**
- 60 servers across 3 AZs (20 per AZ)
- Each server has full Trie copy
- No leader/follower, all equal
- Any server can handle any request

**Index Distribution:**
- S3 with cross-region replication
- Each region has local copy
- Servers download from nearest S3

**Failover:**
- Server fails → LB routes to others
- AZ fails → DNS removes AZ
- Region fails → Route53 routes to other region

### Graceful Degradation Strategies

**Priority of features to drop under stress:**

1. **First to drop**: Personalization
   - Impact: Generic suggestions only
   - Trigger: User context lookup slow

2. **Second to drop**: Kafka logging
   - Impact: Analytics delayed
   - Trigger: Kafka unavailable

3. **Third to drop**: Trending boost
   - Impact: Slightly stale suggestions
   - Trigger: Pipeline delayed

4. **Last resort**: Return empty suggestions
   - Impact: No autocomplete
   - Trigger: Trie corrupted or not loaded

### Disaster Recovery

**Multi-Region Deployment:**
- Primary: us-east-1
- Secondary: eu-west-1
- Tertiary: ap-south-1

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Suggestion  │  │ Suggestion  │  │ Suggestion  │           │  │
│  │  │ Service     │  │ Service     │  │ Service     │           │  │
│  │  │ (20 pods)   │  │ (20 pods)   │  │ (20 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  Trie Index (in-memory, 100 GB per pod)          │          │  │
│  │  │  ──async sync──> S3 (index snapshots)            │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Query Database (PostgreSQL) - Query frequency data      │ │  │
│  │  │  ──async repl──> Query DB Replica (DR region)            │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async replication──                            │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    SECONDARY REGION: eu-west-1                       │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  Suggestion Service (20 pods) - Active, serving EU traffic       │ │
│  │  Trie Index (synced from S3, updated hourly)                      │ │
│  │  Query DB Replica (async replication from us-east-1)             │ │
│  │  Route53 latency-based routing (EU users → eu-west-1)            │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Trie Index Replication:**
   - **Primary:** us-east-1 generates Trie index from query database
   - **Replication:** Index snapshots uploaded to S3 every hour
   - **Secondary/Tertiary:** Download latest index from S3, load into memory
   - **Replication Lag:** < 1 hour (acceptable for autocomplete)
   - **Conflict Resolution:** Not applicable (read-only Trie, single source of truth)

2. **Query Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → Secondary/Tertiary regions for disaster recovery
   - **Replication Lag Target:** < 1 hour for DR regions
   - **Conflict Resolution:** Not applicable (single-writer model)

3. **Traffic Routing:**
   - **Route53 Latency-Based Routing:** Users routed to nearest region
   - **Health Checks:** Unhealthy regions automatically removed from routing
   - **Failover:** On primary failure, traffic automatically routes to secondary

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Trie Index Sync Lag** | Stale suggestions in secondary regions | Acceptable for autocomplete (1 hour lag), users see recent queries |
| **Replication Lag** | Stale query frequency data | Acceptable for DR (RPO < 1 hour), use sync replica for reads in primary |
| **Latency** | Higher latency for cross-region writes | Only primary region accepts writes, DR regions read-only until failover |
| **Cost** | 3x infrastructure cost (active-active) | Each region serves local traffic, reduces latency for users |
| **Data Consistency** | Eventual consistency across regions | Acceptable for autocomplete (suggestions don't need real-time accuracy) |
| **Index Size** | 100 GB Trie per region (expensive memory) | Use Trie compression, shared memory across pods where possible |

**Failover Scenarios:**

1. **Single Region Failure:**
   - Impact: Users in that region experience higher latency (routed to nearest region)
   - Recovery: Automatic (Route53 health checks)
   - RTO: < 2 minutes (automatic DNS propagation)
   - RPO: < 1 hour (Trie index sync lag)

2. **Primary Region Failure:**
   - Impact: Query frequency updates stop, but suggestions still work
   - Recovery: Manual promotion of secondary region to primary
   - RTO: < 5 minutes (manual process)
   - RPO: < 1 hour (Trie index sync lag)

3. **Network Partition:**
   - Impact: Regions isolated, each serves local traffic independently
   - Recovery: Automatic (Route53 routes to available regions)
   - Strategy: Prefer availability over consistency (AP system)

**Cost Implications:**

- **Primary Region:** $300,000/month (full infrastructure)
- **Secondary Region:** $300,000/month (active-active, same capacity)
- **Tertiary Region:** $300,000/month (active-active, same capacity)
- **Total Multi-Region Cost:** $900,000/month (3x increase)

**Benefits:**
- **Latency:** Reduced latency for users (served from nearest region)
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Global Scale:** Serves users worldwide with low latency

**Trade-offs:**
- **Cost:** 3x increase for active-active deployment
- **Complexity:** More complex operations (Trie sync, multi-region monitoring)
- **Consistency:** Eventual consistency (acceptable for autocomplete)

**RTO/RPO:**
- RPO: ~1 hour (index update frequency)
- RTO: ~5 minutes (DNS failover)

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Trie index replicated to DR region (within 1 hour lag)
- [ ] Database replication lag < 1 hour
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current traffic metrics (QPS, latency, error rate)
   - Capture Trie index version and size
   - Document active query count
   - Verify cache hit rates
   - Test autocomplete accuracy (100 sample queries)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in primary region (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+5 minutes):**
   - **T+0-30s:** Detect failure via health checks
   - **T+30s-1min:** Update DNS records (Route53 health checks)
   - **T+1-2min:** Verify Trie index available in DR region (check version)
   - **T+2-3min:** Warm up cache with popular queries
   - **T+3-4min:** Verify all services healthy in DR region
   - **T+4-5min:** Resume traffic to DR region

4. **Post-Failover Validation (T+5 to T+20 minutes):**
   - Verify RTO < 5 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 hour: Check Trie index version timestamp
   - Test autocomplete: Run 1,000 sample queries, verify accuracy >95%
   - Test query performance: Verify latency < 50ms p99
   - Monitor metrics: QPS, latency, error rate return to baseline
   - Verify Trie integrity: Check Trie structure, no corruption

5. **Data Integrity Verification:**
   - Compare query corpus: Pre-failover vs post-failover (should match within 1 hour)
   - Spot check: Verify 100 random queries return correct suggestions
   - Check Trie structure: Verify no corruption, all paths valid
   - Test edge cases: Empty queries, special characters, long queries

6. **Failback Procedure (T+20 to T+30 minutes):**
   - Restore primary region services
   - Sync Trie index from DR to primary
   - Verify index version matches
   - Update DNS to route traffic back to primary
   - Monitor for 10 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 5 minutes: Time from failure to service resumption
- ✅ RPO < 1 hour: Maximum data loss (verified via Trie index timestamp)
- ✅ Autocomplete queries return correct results: >95% accuracy
- ✅ No data loss: Query corpus matches pre-failover (within 1 hour window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Trie structure intact: No corruption, all paths valid

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

### Database Connection Pool Configuration

**Connection Pool Settings (HikariCP):**

```yaml
spring:
  datasource:
    hikari:
      # Pool sizing
      maximum-pool-size: 15          # Max connections per application instance
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
For typeahead service:
- 4 CPU cores per pod
- Read-heavy (Trie lookups are in-memory, DB for metadata only)
- Calculated: (4 × 2) + 1 = 9 connections minimum
- With safety margin: 15 connections per pod
- 50 pods × 15 = 750 max connections to database
- Database max_connections: 1000 (20% headroom)
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
   - PgBouncer pool: 50 connections
   - App pools can share PgBouncer connections efficiently

### What Breaks First Under Load

**Order of failure at 10x traffic (5M QPS):**

1. **Network bandwidth** (First)
   - Symptom: Packet loss, timeouts
   - Each server: 10Gbps max
   - Fix: Add more servers, optimize response size

2. **CPU** (Second)
   - Symptom: Increased latency
   - Trie lookup is CPU-bound
   - Fix: Add more servers, optimize code

3. **Memory** (Third)
   - Symptom: GC pauses
   - Trie takes 400GB, leaves 100GB for JVM
   - Fix: Larger instances, optimize Trie

4. **CDN capacity** (Fourth)
   - Symptom: CDN throttling
   - Fix: Increase CDN limits, add origins

---

## 2. Monitoring & Observability

### Key Metrics (Golden Signals)

**Latency:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| P50 | < 15ms | > 20ms | > 30ms |
| P95 | < 30ms | > 40ms | > 50ms |
| P99 | < 50ms | > 70ms | > 100ms |
| P99.9 | < 100ms | > 150ms | > 200ms |

**Traffic:**

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Requests/sec | 500K | > 700K | > 900K |
| CDN hit rate | 40% | < 30% | < 20% |

**Errors:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Error rate (5xx) | < 0.001% | > 0.01% | > 0.1% |
| Empty results | < 5% | > 10% | > 20% |

**Saturation:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 60% | > 80% |
| Memory | > 80% | > 90% |
| Network | > 70% | > 85% |

### Metrics Collection

```java
@Component
public class TypeaheadMetrics {
    
    private final MeterRegistry registry;
    
    // Latency histogram
    private final Timer suggestionLatency;
    
    // Counters
    private final Counter totalRequests;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Counter emptyResults;
    
    // Gauges
    private final AtomicLong trieSize;
    private final AtomicLong indexVersion;
    
    public TypeaheadMetrics(MeterRegistry registry) {
        this.suggestionLatency = Timer.builder("suggestions.latency")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99, 0.999)
            .register(registry);
        
        this.totalRequests = Counter.builder("suggestions.requests")
            .tag("status", "total")
            .register(registry);
        
        this.emptyResults = Counter.builder("suggestions.empty")
            .register(registry);
        
        this.trieSize = registry.gauge("trie.size.bytes", new AtomicLong());
        this.indexVersion = registry.gauge("index.version", new AtomicLong());
    }
    
    public void recordRequest(long latencyMs, boolean cacheHit, int resultCount) {
        suggestionLatency.record(latencyMs, TimeUnit.MILLISECONDS);
        totalRequests.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
        
        if (resultCount == 0) {
            emptyResults.increment();
        }
    }
}
```

### Alerting Thresholds and Escalation

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| High Latency P99 | > 70ms for 5 min | Warning | Ticket |
| High Latency P99 | > 100ms for 2 min | Critical | Page on-call |
| Error Rate | > 0.01% for 5 min | Warning | Ticket |
| Error Rate | > 0.1% for 1 min | Critical | Page on-call |
| Empty Results | > 10% for 10 min | Warning | Ticket |
| Index Stale | > 2 hours | Warning | Ticket |
| Index Stale | > 4 hours | Critical | Page on-call |

### Logging Strategy

**Structured Logging Format:**

```java
@Slf4j
@RestController
public class SuggestionController {
    
    @GetMapping("/v1/suggestions")
    public ResponseEntity<SuggestionResponse> getSuggestions(
            @RequestParam String q,
            @RequestParam(defaultValue = "10") int limit,
            HttpServletRequest request) {
        
        long startTime = System.nanoTime();
        
        List<Suggestion> suggestions = suggestionService.getSuggestions(q, limit);
        
        long latencyUs = (System.nanoTime() - startTime) / 1000;
        
        // Only log slow requests or samples
        if (latencyUs > 50000 || ThreadLocalRandom.current().nextInt(1000) == 0) {
            log.info("suggestion.request",
                kv("query", q.substring(0, Math.min(q.length(), 20))),  // Truncate for privacy
                kv("latency_us", latencyUs),
                kv("result_count", suggestions.size()),
                kv("cache_hit", false),  // CDN handles caching
                kv("client_ip_hash", hashIp(request.getRemoteAddr()))
            );
        }
        
        return ResponseEntity.ok(new SuggestionResponse(q, suggestions));
    }
}
```

**Log Sampling:**
- Log 0.1% of requests (1 in 1000)
- Log 100% of slow requests (> 50ms)
- Log 100% of errors

### Distributed Tracing

**Trace ID Propagation:**

```java
@Component
public class TracingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        String traceId = ((HttpServletRequest) request).getHeader("X-Trace-ID");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString().substring(0, 8);  // Short trace ID
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

**Span Boundaries:**
1. HTTP Request (root span)
2. Query normalization (child span)
3. Trie lookup (child span)
4. Response serialization (child span)

### Health Checks

```java
@Component
public class TypeaheadHealthIndicator implements HealthIndicator {
    
    private final TrieLoader trieLoader;
    private final IndexVersionManager versionManager;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check Trie is loaded
        Trie trie = trieLoader.getCurrentTrie();
        if (trie == null) {
            return Health.down()
                .withDetail("error", "Trie not loaded")
                .build();
        }
        
        details.put("trie_loaded", true);
        details.put("index_version", versionManager.getCurrentVersion());
        details.put("node_count", trie.getNodeCount());
        
        // Check latency
        long testLatency = measureTestQuery();
        details.put("test_latency_us", testLatency);
        
        if (testLatency > 10000) {  // > 10ms
            return Health.down()
                .withDetails(details)
                .withDetail("error", "High latency")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
    
    private long measureTestQuery() {
        long start = System.nanoTime();
        trieLoader.getCurrentTrie().getSuggestions("test", 10);
        return (System.nanoTime() - start) / 1000;
    }
}
```

**Kubernetes Probes:**

```yaml
livenessProbe:
  httpGet:
    path: /health/liveness
    port: 8080
  initialDelaySeconds: 60  # Wait for Trie to load
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/readiness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 5
  failureThreshold: 3
```

### Dashboards

**Golden Signals Dashboard:**
- Request rate (total, by region)
- Latency percentiles (P50, P95, P99, P99.9)
- Error rate (by type)
- CDN hit rate

**System Health Dashboard:**
- CPU, memory, network per server
- Index version across servers
- Trie size and node count
- GC pause times

---

## 3. Security Considerations

### Authentication Mechanism

**API Key for Partners:**

```java
@Service
public class ApiKeyService {
    
    public Optional<ApiPartner> validateApiKey(String apiKey) {
        if (apiKey == null || !apiKey.startsWith("ta_")) {
            return Optional.empty();
        }
        
        String hash = DigestUtils.sha256Hex(apiKey);
        return apiKeyRepository.findByKeyHash(hash);
    }
}
```

**Anonymous Access:**
- Allowed with rate limiting
- IP-based rate limiting
- No authentication required

### Authorization and Access Control

**Permission Model:**

| Client Type | Access Level | Rate Limit |
|-------------|--------------|------------|
| Anonymous | Public suggestions only | 100/min |
| Authenticated | Public + personalized | 1000/min |
| API Partner | Full API access | 10000/min |

### Data Encryption

**At Rest:**
- S3: SSE-S3 encryption
- Index files: Not encrypted (public data)

**In Transit:**
- All traffic: TLS 1.3
- Internal: TLS within VPC

### Input Validation and Sanitization

```java
@Service
public class QueryValidator {
    
    private static final int MAX_QUERY_LENGTH = 200;
    private static final Pattern VALID_CHARS = Pattern.compile("^[a-zA-Z0-9 ]+$");
    
    public ValidationResult validate(String query) {
        // Length check
        if (query == null || query.isEmpty()) {
            return ValidationResult.invalid("Query required");
        }
        
        if (query.length() > MAX_QUERY_LENGTH) {
            return ValidationResult.invalid("Query too long");
        }
        
        // Minimum length
        if (query.length() < 2) {
            return ValidationResult.invalid("Query too short");
        }
        
        // Character validation (simplified)
        if (!VALID_CHARS.matcher(query).matches()) {
            return ValidationResult.invalid("Invalid characters");
        }
        
        return ValidationResult.valid();
    }
}
```

### PII Handling and Compliance

**GDPR Compliance:**

1. **No PII in suggestions**: Queries containing PII are filtered
2. **Log anonymization**: IP addresses hashed
3. **Data retention**: Logs deleted after 90 days
4. **Right to deletion**: User search history can be purged

**PII Detection:**

```java
public class PiiDetector {
    
    private static final Pattern EMAIL = Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    private static final Pattern PHONE = Pattern.compile("\\d{3}[-.]?\\d{3}[-.]?\\d{4}");
    private static final Pattern SSN = Pattern.compile("\\d{3}-\\d{2}-\\d{4}");
    
    public boolean containsPii(String query) {
        return EMAIL.matcher(query).find() ||
               PHONE.matcher(query).find() ||
               SSN.matcher(query).find();
    }
}
```

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **DDoS** | CDN, rate limiting, auto-scaling |
| **Query injection** | Input validation, no SQL/code execution |
| **Data scraping** | Rate limiting, CAPTCHA for suspicious patterns |
| **Cache poisoning** | Validate responses, signed URLs |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: User Types Search Query

**Step-by-step:**

1. **User Action**: Types "how" in search box
2. **Client**: Debounce timer starts (150ms)
3. **User Action**: Types "how to" (within 150ms)
4. **Client**: Debounce timer resets
5. **User Action**: Pauses typing (150ms passes)
6. **Client**: Sends `GET /v1/suggestions?q=how+to`
7. **CDN**: Cache check for "how to" → HIT (40% chance)
8. **If CDN HIT**: Return cached response (< 10ms)
9. **If CDN MISS**: Forward to origin
10. **Load Balancer**: Route to healthy server
11. **Suggestion Service**:
    - Normalize query: "how to" → "how to"
    - Trie lookup: find("how to") → 0.5ms
    - Return pre-computed suggestions
12. **Response**: JSON with top 10 suggestions
13. **CDN**: Cache response for 5 minutes
14. **Client**: Display suggestions dropdown

**Total latency:**
- CDN HIT: 5-10ms
- CDN MISS: 20-40ms

### Journey 2: Index Update Propagation

**Step-by-step:**

1. **Pipeline**: Completes building v124 Trie
2. **Pipeline**: Uploads to S3: `s3://typeahead-index/v124/trie.bin`
3. **Pipeline**: Updates Zookeeper: `/typeahead/current_version = "v124"`
4. **Server 1**: Zookeeper watcher triggered
5. **Server 1**: Downloads v124 from S3 (60 seconds)
6. **Server 1**: Deserializes Trie (30 seconds)
7. **Server 1**: Atomic swap to new Trie
8. **Server 1**: Now serving v124
9. **Servers 2-60**: Staggered update (10 at a time)
10. **All servers**: On v124 within 10 minutes

**During update:**
- Some servers on v123, some on v124
- Suggestions may differ slightly
- Acceptable for ~10 minute window

### Failure & Recovery Walkthrough

**Scenario: Server Memory Exhaustion**

**RTO (Recovery Time Objective):** < 2 minutes (pod restart + cache warm-up)  
**RPO (Recovery Point Objective):** 0 (no data loss, Trie rebuilt from database)

**Timeline:**

```
T+0s:    Server 15 starts loading new index
T+30s:   Memory usage: 450GB / 512GB
T+60s:   Memory usage: 500GB / 512GB
T+90s:   GC pressure increases, latency spikes
T+100s:  Health check fails (latency > 100ms)
T+110s:  LB removes Server 15 from rotation
T+120s:  Server 15 OOM killed
T+130s:  Kubernetes restarts pod
T+180s:  New pod starts, begins loading index
T+300s:  Index loaded, pod marked ready
T+310s:  LB adds pod back to rotation
```

**What degrades:**
- 1/60 capacity lost for ~3 minutes
- Other servers handle extra load
- Latency may increase slightly

**What stays up:**
- 59 other servers serving requests
- CDN cache still working
- No user-visible errors

**What recovers automatically:**
- Kubernetes restarts failed pod
- Pod loads index and rejoins
- LB adds pod back

**What requires human intervention:**
- Investigate root cause
- Potentially increase instance size
- Review memory usage patterns

**Runbook Steps:**

1. **Detection (T+0s to T+100s):**
   - Monitor: Check Grafana dashboard for memory usage > 90%
   - Alert: PagerDuty alert triggered at T+100s
   - Verify: SSH to server, check `free -h` and `top`
   - Confirm: Memory exhaustion confirmed

2. **Immediate Response (T+100s to T+120s):**
   - Action: Kubernetes automatically removes pod from service
   - Fallback: Load balancer routes traffic to remaining 59 servers
   - Impact: 1.67% capacity loss (acceptable)
   - Monitor: Verify other servers handle increased load

3. **Recovery (T+120s to T+310s):**
   - T+130s: Kubernetes scheduler assigns new pod
   - T+180s: Pod starts, begins loading Trie index from database
   - T+300s: Index fully loaded (500M entries)
   - T+310s: Health check passes, pod added to service
   - Verification: Check metrics show pod serving traffic

4. **Post-Recovery (T+310s to T+600s):**
   - Monitor: Watch memory usage for 5 minutes
   - Verify: No additional failures
   - Document: Log incident in runbook
   - Follow-up: Schedule post-mortem if recurring

**Prevention:**
- Set memory limits in Kubernetes (prevent OOM)
- Add memory alerts at 80% threshold
- Consider larger instance sizes if pattern continues

### Failure Scenario: Redis Cache Down

**Detection:**
- How to detect: Grafana alert "Redis cluster down"
- Alert thresholds: 3 consecutive health check failures
- Monitoring: Check Redis cluster status endpoint

**Impact:**
- Affected services: All suggestion servers (cache miss rate 100%)
- User impact: Latency increases from 10ms to 50ms (still acceptable)
- Degradation: System continues operating, slower responses

**Recovery Steps:**
1. **T+0s**: Alert received, on-call engineer paged
2. **T+30s**: Verify Redis cluster status via AWS console
3. **T+60s**: Check Redis primary node health
4. **T+90s**: If primary down, promote replica to primary (automatic)
5. **T+120s**: Verify Redis cluster healthy
6. **T+150s**: Servers reconnect to Redis
7. **T+180s**: Cache begins warming up
8. **T+300s**: Cache hit rate returns to normal (95%)

**RTO:** < 5 minutes
**RPO:** 0 (no data loss, cache rebuilds from database)

**Prevention:**
- Redis cluster with automatic failover
- Health checks every 10 seconds
- Circuit breaker falls back to database on Redis failure

### Failure Scenario: Database Primary Failure

**Detection:**
- How to detect: Database connection errors in logs
- Alert thresholds: > 10% error rate for 30 seconds
- Monitoring: PostgreSQL primary health check endpoint

**Impact:**
- Affected services: Index update pipeline (can't write new data)
- User impact: Suggestions continue (read-only from Trie)
- Degradation: New queries not indexed until recovery

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
- Regular failover testing (quarterly)

### High-Load / Contention Scenario: Breaking News Query Spike

```
Scenario: Major event occurs, millions search for same query simultaneously
Time: Breaking news event (e.g., "earthquake", "election results")

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Breaking news event triggers millions of searches     │
│ - Query: "earthquake" (trending query)                      │
│ - 50,000 requests/second hit system                         │
│ - Expected: ~100 requests/second normal traffic            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-10s: CDN Cache Miss Storm                              │
│ - CDN cache: MISS (query not cached, first time trending)   │
│ - All requests forward to origin (50K req/sec)              │
│ - Load balancer distributes across 60 servers               │
│ - Each server: ~833 requests/second                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Origin Load Handling:                                       │
│                                                              │
│ 1. Suggestion Service (per server):                        │
│    - Trie lookup: find("earthquake") → 0.5ms              │
│    - Returns pre-computed suggestions (in-memory)          │
│    - No database queries (Trie is in-memory)               │
│    - Latency: ~2-5ms per request                           │
│                                                              │
│ 2. CDN Caching:                                            │
│    - First 100 requests miss, forward to origin            │
│    - Origin responds with suggestions                       │
│    - CDN caches response (5min TTL)                        │
│    - Next 49,900 requests: CDN HIT (< 10ms latency)       │
│                                                              │
│ 3. Capacity:                                               │
│    - Each server handles ~1,000 req/sec comfortably       │
│    - 60 servers = 60,000 req/sec capacity                  │
│    - 50K req/sec = 83% utilization (safe)                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                      │
│ - All 50K req/sec handled successfully                     │
│ - 99.8% served from CDN after warm-up                      │
│ - P99 latency: < 50ms (well under 100ms target)           │
│ - No service degradation                                   │
│ - No database load (Trie is in-memory)                     │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Unicode Query Normalization

```
Scenario: User types query with special characters that normalize differently
Edge Case: Query normalization produces unexpected results

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: User types "café" (with accent)                     │
│ - Client sends: GET /v1/suggestions?q=café                │
│ - Query encoded as: "caf%C3%A9" (UTF-8 URL encoding)       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Server Receives Query                              │
│ - URL decode: "caf%C3%A9" → "café"                        │
│ - Normalize: "café" → "cafe" (strip accents)              │
│ - Trie lookup: find("cafe") → Returns suggestions          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: User Types "cafe" (without accent)                 │
│ - Query: "cafe"                                            │
│ - Normalize: "cafe" → "cafe" (no change)                   │
│ - Trie lookup: find("cafe") → Same suggestions             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Edge Case: Mixed Unicode Characters                        │
│ - User types: "café shop" (mixed)                          │
│ - Normalize: "café shop" → "cafe shop"                     │
│ - Trie lookup: find("cafe shop") → Works correctly         │
│                                                              │
│ Problem: User sees different suggestions as they type      │
│ - "café" → normalized to "cafe" → suggestions change       │
│ - User experience: Suggestions "jump" unexpectedly         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Client-Side Normalization Consistency            │
│                                                              │
│ 1. Client normalizes before sending:                       │
│    - "café" → "cafe" (before API call)                    │
│    - Consistent behavior regardless of input method        │
│                                                              │
│ 2. Server also normalizes (defense in depth):              │
│    - Double-normalization is idempotent                    │
│    - Ensures consistency even if client buggy              │
│                                                              │
│ 3. Trie Index Contains Both Forms:                        │
│    - Index: Both "cafe" and "café" point to same results  │
│    - Covers all input variations                           │
│                                                              │
│ Result:                                                     │
│ - Consistent suggestions regardless of input method        │
│ - No "jumping" suggestions                                 │
│ - Better user experience                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Compute (60 × r5.16xlarge) | $150,000 | 50% |
| CDN (CloudFront) | $80,000 | 27% |
| Data Pipeline (EMR) | $30,000 | 10% |
| Storage (S3) | $5,000 | 2% |
| Network (data transfer) | $20,000 | 7% |
| Kafka (MSK) | $10,000 | 3% |
| Other | $5,000 | 1% |
| **Total** | **$300,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $300,000
**Target Monthly Cost:** $200,000 (33% reduction)

**Top 3 Cost Drivers:**
1. **Compute (EC2):** $150,000/month (50%) - Largest single component
2. **CDN (CloudFront):** $80,000/month (27%) - High traffic volume
3. **Network (Data Transfer):** $20,000/month (7%) - Cross-region replication

**Optimization Strategies (Ranked by Impact):**

1. **Reserved Instances (40% savings on compute):**
   - **Current:** On-demand r5.16xlarge instances (60 × $2,500 = $150K)
   - **Optimization:** 1-year Reserved Instances for 80% of compute
   - **Savings:** $60,000/month (40% of $150K)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

2. **Trie Compression & Memory Optimization:**
   - **Current:** 100 GB Trie per pod, requires r5.16xlarge (512 GB RAM)
   - **Optimization:** 
     - Implement Trie compression (reduce size by 30%)
     - Use smaller instance types (r5.8xlarge sufficient after compression)
     - Shared memory across pods using memory-mapped files
   - **Savings:** $45,000/month (30% of $150K compute)
   - **Trade-off:** Slight performance impact (acceptable, still < 50ms)

3. **CDN Optimization (30% savings):**
   - **Current:** 1-hour TTL, 60% cache hit rate
   - **Optimization:** 
     - Increase TTL to 24 hours for popular queries
     - Implement cache warming for top 100K queries
     - Use CloudFront price classes (reduce data transfer costs)
   - **Savings:** $24,000/month (30% of $80K)
   - **Trade-off:** Slightly stale suggestions (acceptable for autocomplete)

4. **Spot Instances for Pipeline Workers:**
   - **Current:** On-demand instances for EMR pipeline
   - **Optimization:** Use Spot instances (70% savings)
   - **Savings:** $21,000/month (70% of $30K pipeline)
   - **Trade-off:** Possible interruptions (acceptable for batch jobs)

5. **Regional Optimization:**
   - **Current:** Equal capacity in all 3 regions
   - **Optimization:** 
     - Reduce capacity in low-traffic regions (ap-south-1)
     - Use smaller instances in secondary regions
   - **Savings:** $15,000/month (10% of $150K compute)
   - **Trade-off:** Slightly higher latency for some users

6. **Network Optimization:**
   - **Current:** High cross-region data transfer costs
   - **Optimization:** 
     - Reduce Trie sync frequency (hourly → every 2 hours)
     - Compress Trie snapshots before transfer
   - **Savings:** $6,000/month (30% of $20K network)
   - **Trade-off:** Slightly stale Trie in secondary regions (acceptable)

**Total Potential Savings:** $171,000/month (57% reduction)
**Optimized Monthly Cost:** $129,000/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Autocomplete Query | $0.00023 | $0.00010 | 57% |
| Trie Index Update | $0.50 | $0.21 | 58% |
| Data Pipeline Run | $100 | $30 | 70% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Reserved Instances, Trie Compression → $105K savings
2. **Phase 2 (Month 2):** CDN optimization, Spot instances → $45K savings
3. **Phase 3 (Month 3):** Regional optimization, Network optimization → $21K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor Trie memory usage (target < 70 GB per pod)
- Monitor cache hit rates (target >80%)
- Monitor performance metrics (ensure latency < 50ms p99)
- Review and adjust quarterly

### Build vs Buy Decisions

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Suggestion Service | Build | Core business logic |
| Trie Implementation | Build | Custom optimizations needed |
| Data Pipeline | Buy (EMR) | Managed Spark, less ops |
| CDN | Buy (CloudFront) | Global edge network |
| Coordination | Buy (Zookeeper) | Proven, reliable |

### At What Scale Does This Become Expensive?

| Scale | Monthly Cost | Cost per 1M requests |
|-------|--------------|----------------------|
| 500K QPS | $300,000 | $0.23 |
| 1M QPS | $500,000 | $0.19 |
| 5M QPS | $1,500,000 | $0.12 |

**Economies of scale**: Cost per request decreases due to:
- Better CDN cache hit rates
- More efficient server utilization
- Volume discounts

### Cost per User Estimate

**Assumptions:**
- 100M daily active users
- 10 searches per user per day
- 5 suggestions requests per search

**Calculation:**
- Total requests: 100M × 10 × 5 = 5 billion/day
- Monthly requests: 150 billion
- Cost: $300,000/month
- Cost per user: $0.003/month
- Cost per request: $0.000002

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Load Balancing | NLB, round-robin | < 1ms added latency |
| Rate Limiting | In-memory token bucket | 100 req/min anonymous |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | P99 < 50ms |
| Security | Input validation, rate limiting | No injection vulnerabilities |
| DR | Multi-region active-active | RTO < 5 min |
| Cost | $300,000/month | $0.003 per user |

