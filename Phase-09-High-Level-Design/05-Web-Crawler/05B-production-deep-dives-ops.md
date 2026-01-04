# Web Crawler - Production Deep Dives (Operations)

## Overview

This document covers operational concerns: scaling and reliability, monitoring and observability, security considerations, end-to-end simulation, and cost analysis.

---

## 1. Scaling & Reliability

### Load Balancing Approach

**Crawler Distribution:**

The web crawler uses Kafka partitioning for load distribution rather than traditional load balancing:

```
Kafka Topic: discovered-urls (15 partitions)
    ↓
Partition 0 → Crawler Pod 0
Partition 1 → Crawler Pod 1
...
Partition 14 → Crawler Pod 14
```

**Why Kafka-based distribution?**
- Domain affinity (same domain → same crawler)
- Automatic rebalancing on pod failure
- No central coordinator bottleneck

### Rate Limiting Implementation

**Per-Domain Rate Limiting:**

```java
@Service
public class DomainRateLimiter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    public void acquire(String domain) {
        RateLimiter limiter = limiters.computeIfAbsent(domain, 
            d -> RateLimiter.create(getRate(d)));
        limiter.acquire();
    }
    
    private double getRate(String domain) {
        // Get crawl delay from robots.txt
        int delayMs = robotsManager.getCrawlDelay(domain);
        return 1000.0 / delayMs;  // Requests per second
    }
}
```

### Circuit Breaker Placement

| Service | Circuit Breaker | Fallback |
|---------|-----------------|----------|
| HTTP Fetch | Yes | Mark URL for retry |
| DNS Resolution | Yes | Use cached IP |
| S3 Storage | Yes | Buffer locally |
| Kafka Producer | Yes | Local queue |

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      http-fetch:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      dns:
        slidingWindowSize: 20
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
```

### Timeout and Retry Policies

| Operation | Timeout | Retries | Backoff |
|-----------|---------|---------|---------|
| DNS Lookup | 5s | 2 | 1s |
| HTTP Fetch | 30s | 3 | 1s, 5s, 15s |
| Robots.txt | 10s | 2 | 2s |
| S3 Upload | 60s | 3 | 5s exponential |

### Replication and Failover

**Crawler Pods:**
- 15 pods across 3 AZs
- Kafka consumer group rebalancing on failure
- Stateless design (state in Redis/Kafka)

**Failover:**
- Pod fails → Kafka rebalances partitions
- URLs re-consumed from last committed offset
- No data loss (Kafka retention)

### Graceful Degradation

**Priority of features to drop:**

1. **First**: Low-priority URLs (skip priority 4-5)
2. **Second**: Content storage (crawl but don't store)
3. **Third**: Link extraction (fetch only, no parsing)
4. **Last**: Stop crawling entirely

### Backup and Recovery

**RPO (Recovery Point Objective):** < 1 hour
- Maximum acceptable data loss
- Based on Kafka retention period (1 hour)
- Crawled content stored in S3 with versioning enabled
- URL frontier state persisted to Redis with hourly snapshots

**RTO (Recovery Time Objective):** < 30 minutes
- Maximum acceptable downtime
- Time to restore service via failover to DR region
- Includes spinning up new crawler pods and rebalancing Kafka partitions

**Backup Strategy:**
- **Kafka Retention**: 1 hour retention for URL queue
- **S3 Versioning**: All crawled content stored with versioning
- **Redis Snapshots**: Hourly snapshots of URL frontier state
- **Cross-region Replication**: Async replication to DR region (us-west-2)
- **Database Backups**: Daily full backups, hourly incremental

**Restore Steps:**
1. Detect primary region failure (health checks) - 1 minute
2. Promote DR region to primary - 5 minutes
3. Restore Redis state from latest snapshot - 10 minutes
4. Spin up crawler pods and rebalance Kafka - 10 minutes
5. Verify crawler health and resume crawling - 4 minutes

**Disaster Recovery Testing:**

**Frequency:** Quarterly (every 3 months)

**Pre-Test Checklist:**
- [ ] DR region infrastructure provisioned and healthy
- [ ] Database replication lag < 1 hour
- [ ] Redis snapshots verified in S3
- [ ] Kafka replication lag < 1 hour
- [ ] DNS failover scripts tested
- [ ] Monitoring alerts configured for DR region
- [ ] On-call engineer notified and available

**Test Process (Step-by-Step):**

1. **Pre-Test Baseline (T-30 minutes):**
   - Record current crawl rate (pages/second)
   - Capture URL frontier size
   - Document active crawl jobs
   - Verify Redis state (seen URLs count)
   - Test crawl functionality (10 sample URLs)

2. **Simulate Primary Region Failure (T+0):**
   - Stop all services in us-east-1 (or use chaos engineering tool)
   - Verify health checks fail
   - Confirm traffic routing stops to primary region

3. **Execute Failover Procedure (T+0 to T+30 minutes):**
   - **T+0-1 min:** Detect failure via health checks
   - **T+1-6 min:** Promote DR region (us-west-2) to primary
   - **T+6-7 min:** Update DNS records (Route53 health checks)
   - **T+7-17 min:** Restore Redis state from latest snapshot
   - **T+17-27 min:** Spin up crawler pods and rebalance Kafka partitions
   - **T+27-29 min:** Verify crawler health and resume crawling
   - **T+29-30 min:** Resume traffic to DR region

4. **Post-Failover Validation (T+30 to T+45 minutes):**
   - Verify RTO < 30 minutes: ✅ PASS/FAIL
   - Verify RPO < 1 hour: Check replication lag at failure time
   - Test crawl functionality: Crawl 100 test URLs, verify success
   - Test URL frontier: Verify URLs are being processed
   - Monitor metrics: Crawl rate, error rate return to baseline
   - Verify Redis state: Check seen URLs count matches pre-failover

5. **Data Integrity Verification:**
   - Compare URL frontier size: Pre-failover vs post-failover (should match within 1 hour)
   - Spot check: Verify 100 random URLs in frontier
   - Check Redis state: Verify seen URLs Bloom filter intact
   - Test crawl results: Verify crawled pages stored correctly

6. **Failback Procedure (T+45 to T+60 minutes):**
   - Restore primary region services
   - Sync Redis state from DR to primary
   - Rebalance Kafka partitions
   - Update DNS to route traffic back to primary
   - Monitor for 15 minutes before declaring success

**Validation Criteria:**
- ✅ RTO < 30 minutes: Time from failure to service resumption
- ✅ RPO < 1 hour: Maximum data loss (verified via replication lag)
- ✅ Crawl functionality works: URLs being processed correctly
- ✅ No data loss: URL frontier size matches pre-failover (within 1 hour window)
- ✅ Service resumes within RTO target: All metrics return to baseline
- ✅ Redis state intact: Seen URLs Bloom filter functional

**Post-Test Actions:**
- Document test results in runbook
- Update last test date
- Identify improvements for next test
- Review and update failover procedures if needed

**Last Test:** TBD (to be scheduled)
**Next Test:** [Last Test Date + 3 months]

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Crawl a New Website

**Step-by-step:**

1. **User Action**: Admin submits new domain via API: `POST /v1/crawl-jobs {"domain": "newsite.com"}`
2. **Crawl Scheduler**: 
   - Validates domain
   - Creates crawl job in database
   - Adds seed URLs to URL Frontier: `newsite.com/`, `newsite.com/sitemap.xml`
3. **URL Frontier**: Assigns URLs to crawler workers (distributed queue)
4. **Crawler Worker 1**:
   - Fetches `newsite.com/robots.txt` (cached for 1 hour)
   - Checks robots.txt rules (allowed)
   - Fetches `newsite.com/` (HTTP GET)
   - Downloads HTML (100 KB, 2 seconds)
   - Parses HTML, extracts links
   - Stores content in S3: `s3://crawled-pages/newsite.com/index.html`
   - Updates database: `INSERT INTO pages (url, s3_key, crawled_at)`
   - Extracts new URLs, adds to URL Frontier
5. **Response**: `201 Created` with `{"job_id": "job_123", "status": "running"}`
6. **Background Processing**:
   - Content processor extracts text, metadata
   - Indexer updates search index
   - Analytics job updates crawl statistics
7. **Result**: Website fully crawled, indexed, searchable

**Total time: ~30 minutes** for small site (100 pages)

### Journey 2: Handle Rate-Limited Domain

**Step-by-step:**

1. **Crawler Worker**: Attempts to fetch `popular-site.com/page1`
2. **Target Server**: Returns `429 Too Many Requests` with `Retry-After: 60`
3. **Crawler Worker**:
   - Detects rate limit response
   - Calculates backoff: 60 seconds
   - Re-queues URL with delay: `URL_Frontier.delay(url, 60s)`
   - Moves to next URL in queue
4. **After 60 seconds**: URL becomes available again
5. **Crawler Worker**: Retries fetch, succeeds
6. **Result**: Page crawled successfully after respecting rate limits

**Total time: Original 2s + 60s delay = 62 seconds**

### Failure & Recovery Walkthrough

**Scenario: Crawler Worker Crash During Download**

**RTO (Recovery Time Objective):** < 5 minutes (worker restart + URL re-queue)  
**RPO (Recovery Point Objective):** 0 (URLs re-queued, no data loss)

**Timeline:**

```
T+0s:    Worker 5 downloading large page (50 MB)
T+30s:   Worker 5 crashes (OOM or network timeout)
T+30s:   URL Frontier detects worker heartbeat timeout
T+35s:   URL Frontier marks URLs as "in-progress" by Worker 5 as "failed"
T+40s:   URLs re-queued with priority boost
T+45s:   Worker 6 picks up re-queued URLs
T+2min:  Worker 6 successfully downloads page
T+3min:  Page stored, processing continues
```

**What degrades:**
- URLs assigned to crashed worker delayed by ~3 minutes
- Other workers continue normally
- No data loss (URLs re-queued)

**What stays up:**
- 99 other workers continue crawling
- URL Frontier continues processing
- Database and S3 unaffected

**What recovers automatically:**
- Kubernetes restarts crashed worker
- URL Frontier re-queues failed URLs
- New worker picks up work
- No manual intervention required

**What requires human intervention:**
- Investigate root cause of crash
- Review worker resource allocation
- Check for systemic issues

**Cascading failure prevention:**
- Worker isolation (one crash doesn't affect others)
- URL re-queuing prevents work loss
- Circuit breakers prevent retry storms
- Rate limiting prevents overwhelming targets

### Multi-Region Deployment

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRIMARY REGION: us-east-1                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │  │
│  │  │ Crawler     │  │ Crawler     │  │ Crawler     │           │  │
│  │  │ Service     │  │ Service     │  │ Service     │           │  │
│  │  │ (15 pods)   │  │ (15 pods)   │  │ (15 pods)   │           │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │  │
│  │         │                 │                 │                  │  │
│  │  ┌──────┴─────────────────┴─────────────────┴──────┐          │  │
│  │  │  PostgreSQL (URL frontier, crawl results)       │          │  │
│  │  │  ──async repl──> PostgreSQL Replica (DR)         │          │  │
│  │  └──────────────────────────────────────────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Redis Cluster (seen URLs Bloom filter, robots.txt cache)│ │  │
│  │  │  ──async repl──> Redis Replica (DR)                     │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  Kafka Cluster (crawl jobs, results)                     │ │  │
│  │  │  ──async repl──> Kafka Mirror (DR)                      │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  S3 Bucket (crawled pages storage)                        │ │  │
│  │  │  ──CRR──> S3 Bucket (DR region)                          │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                    ──async replication──                            │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    DR REGION: us-west-2                               │
│                              │                                       │
│  ┌───────────────────────────▼─────────────────────────────────────┐ │
│  │  PostgreSQL Replica (async replication from us-east-1)          │ │
│  │  Redis Replica (async replication + snapshots)                   │ │
│  │  Kafka Mirror (async replication from us-east-1)                 │ │
│  │  Crawler Service (5 pods, minimal for DR readiness)              │ │
│  │  S3 Bucket (CRR destination, read-only until failover)         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**Replication Strategy:**

1. **Database Replication:**
   - **Synchronous:** Primary → Replica in same region for zero data loss
   - **Asynchronous:** Primary → DR region (us-west-2) for disaster recovery
   - **Replication Lag Target:** < 1 hour for DR region
   - **Conflict Resolution:** Not applicable (single-writer model)

2. **Redis Replication:**
   - **Redis Replication:** Async replication to DR region
   - **Snapshots:** Hourly snapshots to S3 for faster recovery
   - **Cache Warm-up:** On failover, restore from latest snapshot

3. **Kafka Replication:**
   - **Kafka MirrorMaker:** Async replication of crawl jobs and results
   - **Replication Lag:** < 1 hour (acceptable for crawl jobs)
   - **Partition Rebalancing:** Automatic on failover

4. **S3 Cross-Region Replication (CRR):**
   - **Replication:** Async replication of crawled pages to DR region
   - **Replication Lag:** Typically < 5 minutes for S3 CRR
   - **Access:** DR region S3 bucket read-only until failover

**Cross-Region Challenges & Solutions:**

| Challenge | Impact | Solution |
|-----------|--------|----------|
| **Replication Lag** | Stale URL frontier in DR region | Acceptable for DR (RPO < 1 hour), use sync replica for reads in primary |
| **Redis State Sync** | Bloom filter may be stale | Use snapshots for faster recovery, accept slight duplication on failover |
| **Kafka Lag** | Crawl jobs may be delayed | Acceptable for DR, jobs will process eventually |
| **S3 CRR Lag** | Crawled pages may not be immediately available | Acceptable for DR (RPO < 5 min), critical pages can be re-crawled |
| **Cost** | 2x infrastructure cost | DR region uses smaller instances, only scaled up during failover |
| **Data Consistency** | Eventual consistency across regions | Acceptable for DR scenario, strong consistency within region |

**Failover Scenarios:**

1. **Single Pod Failure:**
   - Impact: Minimal (other pods handle crawling)
   - Recovery: Automatic (Kubernetes restarts pod)
   - RTO: < 2 minutes (automatic)

2. **Entire Region Failure:**
   - Impact: Complete service outage in primary region
   - Recovery: Manual failover to DR region
   - RTO: < 30 minutes (manual process)
   - RPO: < 1 hour (async replication lag)

3. **Network Partition:**
   - Impact: Primary region isolated, DR region may have stale data
   - Recovery: Manual decision (split-brain prevention)
   - Strategy: Prefer data consistency over availability (CP system)

**Cost Implications:**

- **Primary Region:** $50,000/month (full infrastructure)
- **DR Region:** $8,000/month (minimal infrastructure, 16% of primary)
- **S3 CRR:** $200/month (cross-region replication)
- **Replication Bandwidth:** $500/month (cross-region data transfer)
- **Total Multi-Region Cost:** $58,700/month (17% increase)

**Benefits:**
- **Availability:** 99.99% → 99.999% (reduces downtime from 52 min/year to 5 min/year)
- **Disaster Recovery:** Survives region-wide outages
- **Compliance:** Meets regulatory requirements for geographic redundancy
- **Data Durability:** S3 CRR provides additional backup layer

**Trade-offs:**
- **Cost:** 17% increase for DR capability
- **Complexity:** More complex operations (failover procedures, multi-region monitoring)
- **Latency:** Cross-region replication adds 50-100ms latency (acceptable for async)

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
For crawler service:
- 8 CPU cores per pod
- High write rate (URL frontier updates, crawl results)
- Calculated: (8 × 2) + 1 = 17 connections minimum
- With safety margin for concurrent crawls: 25 connections per pod
- 15 crawler pods × 25 = 375 max connections to database
- Database max_connections: 500 (20% headroom)
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
For URL frontier cache:
- 8 CPU cores per pod
- High read/write rate (URL frontier operations)
- Calculated: 30 connections per pod
- 15 crawler pods × 30 = 450 max connections
- Redis cluster: 6 nodes, 100 connections per node capacity
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
   - PgBouncer pool: 100 connections
   - App pools can share PgBouncer connections efficiently

---

## 2. Monitoring & Observability

### Key Metrics

**Crawl Metrics:**

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Crawl rate | 1000/s | < 800/s | < 500/s |
| Success rate | > 95% | < 90% | < 80% |
| P99 fetch latency | < 20s | > 25s | > 30s |
| Frontier size | < 500M | > 700M | > 1B |

**Resource Metrics:**

| Resource | Warning | Critical |
|----------|---------|----------|
| CPU | > 70% | > 85% |
| Memory | > 80% | > 90% |
| Network | > 60% | > 80% |

### Metrics Collection

```java
@Component
public class CrawlerMetrics {
    
    private final MeterRegistry registry;
    
    private final Counter pagesCrawled;
    private final Counter pagesSuccessful;
    private final Counter pagesFailed;
    private final Counter bytesDownloaded;
    private final Timer fetchLatency;
    private final AtomicLong frontierSize;
    
    public CrawlerMetrics(MeterRegistry registry) {
        this.pagesCrawled = Counter.builder("crawler.pages.total")
            .register(registry);
        
        this.fetchLatency = Timer.builder("crawler.fetch.latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.frontierSize = new AtomicLong(0);
        Gauge.builder("crawler.frontier.size", frontierSize, AtomicLong::get)
            .register(registry);
    }
    
    public void recordCrawl(CrawlResult result) {
        pagesCrawled.increment();
        if (result.isSuccess()) {
            pagesSuccessful.increment();
            bytesDownloaded.increment(result.getContentLength());
        } else {
            pagesFailed.increment();
        }
        fetchLatency.record(result.getDuration(), TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Low Crawl Rate | < 800/s for 5 min | Warning | Investigate |
| Very Low Crawl Rate | < 500/s for 2 min | Critical | Page on-call |
| High Failure Rate | > 10% for 5 min | Warning | Check targets |
| Frontier Growing | > 700M URLs | Warning | Scale crawlers |

### Health Checks

```java
@Component
public class CrawlerHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check Kafka connectivity
        boolean kafkaHealthy = kafkaTemplate.send("health-check", "ping")
            .get(5, TimeUnit.SECONDS) != null;
        details.put("kafka", kafkaHealthy ? "UP" : "DOWN");
        
        // Check Redis connectivity
        boolean redisHealthy = redis.getConnectionFactory()
            .getConnection().ping() != null;
        details.put("redis", redisHealthy ? "UP" : "DOWN");
        
        // Check crawl rate
        double crawlRate = metrics.getCrawlRate();
        details.put("crawl_rate", crawlRate);
        
        if (!kafkaHealthy || !redisHealthy) {
            return Health.down().withDetails(details).build();
        }
        
        if (crawlRate < 500) {
            return Health.down()
                .withDetails(details)
                .withDetail("error", "Crawl rate too low")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

---

## 3. Security Considerations

### Input Validation

```java
@Service
public class URLValidator {
    
    private static final int MAX_URL_LENGTH = 2048;
    private static final Set<String> ALLOWED_SCHEMES = Set.of("http", "https");
    
    public boolean isValid(String url) {
        if (url == null || url.length() > MAX_URL_LENGTH) {
            return false;
        }
        
        try {
            URI uri = new URI(url);
            
            // Check scheme
            if (!ALLOWED_SCHEMES.contains(uri.getScheme())) {
                return false;
            }
            
            // Check for private IPs (SSRF prevention)
            if (isPrivateIP(uri.getHost())) {
                return false;
            }
            
            return true;
        } catch (URISyntaxException e) {
            return false;
        }
    }
    
    private boolean isPrivateIP(String host) {
        // Check for localhost, 10.x.x.x, 192.168.x.x, etc.
        return host.equals("localhost") ||
               host.startsWith("10.") ||
               host.startsWith("192.168.") ||
               host.startsWith("127.");
    }
}
```

### Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **SSRF** | Block private IPs, validate URLs |
| **Crawler Traps** | Limit depth, detect patterns |
| **Malicious Content** | Sandbox parsing, size limits |
| **Legal Issues** | Respect robots.txt, rate limits |

---

## 4. Simulation (End-to-End User Journeys)

### Journey 1: Crawling a New Domain

```
1. Seed URL added: https://example.com
    ↓
2. Check bloom filter → Not seen
    ↓
3. Fetch robots.txt → Allow all, delay 1s
    ↓
4. Cache robots.txt in Redis (24h TTL)
    ↓
5. DNS lookup → 93.184.216.34
    ↓
6. Cache DNS in Redis (1h TTL)
    ↓
7. HTTP GET https://example.com
    ↓
8. Parse HTML, extract 50 links
    ↓
9. Store document in S3
    ↓
10. Produce 50 URLs to Kafka
    ↓
11. Update bloom filter
    ↓
12. Record domain state (last crawl time)
    ↓
13. Wait 1 second (politeness)
    ↓
14. Consume next URL from same domain
```

### Journey 2: Handling Rate Limiting (429)

```
1. Crawler fetches https://example.com/page100
    ↓
2. Server returns 429 Too Many Requests
   Retry-After: 3600
    ↓
3. Mark domain as rate-limited
    ↓
4. Increase domain delay to 1 hour
    ↓
5. Skip all example.com URLs for 1 hour
    ↓
6. Continue with other domains
    ↓
7. After 1 hour, resume example.com
```

### Failure & Recovery

**Scenario: Crawler Pod Crash**

**RTO (Recovery Time Objective):** < 1 minute (pod restart by K8s)  
**RPO (Recovery Point Objective):** 0 (no data loss, URLs in queue persist)

**Runbook Steps:**

1. **Detection (T+0s to T+5s):**
   - Monitor: Kubernetes pod status "CrashLoopBackOff"
   - Alert: PagerDuty alert triggered at T+5s
   - Verify: Check pod logs for crash reason
   - Confirm: Pod crash confirmed

2. **Immediate Response (T+5s to T+15s):**
   - Action: Kafka detects consumer heartbeat timeout
   - Fallback: Kafka triggers consumer group rebalance
   - Impact: Partition 3 temporarily unassigned
   - Monitor: Verify other pods continue crawling

3. **Recovery (T+15s to T+30s):**
   - T+15s: Kafka assigns partition 3 to Pod 4
   - T+20s: Pod 4 starts consuming from last committed offset
   - T+25s: Pod 4 resumes crawling URLs from partition 3
   - T+30s: Crawling fully resumed, no URL loss

4. **Post-Recovery (T+30s to T+300s):**
   - Monitor: Watch pod health for 5 minutes
   - Verify: No additional crashes
   - Document: Log incident, investigate root cause
   - Follow-up: Fix underlying issue if recurring

**Prevention:**
- Resource limits in Kubernetes (prevent OOM)
- Health checks every 30 seconds
- Automatic pod restart on failure

### Failure Scenario: Redis Cache Down

**Detection:**
- How to detect: Grafana alert "Redis cluster down"
- Alert thresholds: 3 consecutive health check failures
- Monitoring: Check Redis cluster status endpoint

**Impact:**
- Affected services: URL frontier state, robots.txt cache, DNS cache
- User impact: Crawler continues (degraded performance)
- Degradation: Slower URL deduplication, repeated robots.txt fetches

**Recovery Steps:**
1. **T+0s**: Alert received, on-call engineer paged
2. **T+30s**: Verify Redis cluster status via AWS console
3. **T+60s**: Check Redis primary node health
4. **T+90s**: If primary down, promote replica to primary (automatic)
5. **T+120s**: Verify Redis cluster healthy
6. **T+150s**: Crawlers reconnect to Redis
7. **T+180s**: Cache begins repopulating

**RTO:** < 3 minutes
**RPO:** 0 (no data loss, cache rebuilds from database)

**Prevention:**
- Redis cluster with automatic failover
- Health checks every 10 seconds
- Fallback to database for critical operations

### Failure Scenario: Database Primary Failure

**Detection:**
- How to detect: Database connection errors in logs
- Alert thresholds: > 10% error rate for 30 seconds
- Monitoring: PostgreSQL primary health check endpoint

**Impact:**
- Affected services: URL deduplication, document metadata storage
- User impact: Crawler continues (may crawl duplicates temporarily)
- Degradation: Duplicate detection disabled until recovery

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

### Failure Scenario: Kafka Lag (URL Queue Backup)

**Detection:**
- How to detect: Grafana alert "Kafka consumer lag > 10M messages"
- Alert thresholds: Lag > 10M messages for 10 minutes
- Monitoring: Check Kafka consumer lag metrics

**Impact:**
- Affected services: URL processing pipeline
- User impact: URLs queued but not crawled (delayed)
- Degradation: System continues crawling, slower processing

**Recovery Steps:**
1. **T+0s**: Alert received, Kafka lag detected
2. **T+30s**: Check consumer group status, identify slow consumers
3. **T+60s**: Scale up crawler pods (add 5 more pods)
4. **T+120s**: New pods join consumer group, begin processing
5. **T+300s**: Lag begins decreasing
6. **T+600s**: Lag returns to normal (< 100K messages)

**RTO:** < 10 minutes
**RPO:** 0 (messages in Kafka, no loss)

**Prevention:**
- Auto-scaling based on Kafka lag
- Monitor consumer processing rate
- Alert on lag thresholds

### High-Load / Contention Scenario: Large Site Crawl Storm

```
Scenario: Crawling a massive site (e.g., Wikipedia) with millions of pages
Time: Initial crawl of large domain

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Seed URL added: https://en.wikipedia.org/wiki/Main   │
│ - Domain: wikipedia.org (millions of pages)                 │
│ - Expected: 10M+ URLs to crawl                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+0-1hour: URL Discovery Phase                             │
│ - Crawler discovers 100K URLs from main page               │
│ - Produces to Kafka: 100K URLs/second                      │
│ - Kafka partitions: Distributes across 15 partitions        │
│ - Bloom filter updates: 100K URLs/second                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Load Handling:                                             │
│                                                              │
│ 1. Kafka Backpressure:                                    │
│    - 15 partitions × 100K URLs = 1.5M URLs/sec            │
│    - Kafka buffers messages (7-day retention)              │
│    - Consumers process at own pace (1000 pages/sec)        │
│                                                              │
│ 2. Rate Limiting Protection:                              │
│    - Per-domain delay: 1 second between requests          │
│    - Limits wikipedia.org to 1 page/second per crawler     │
│    - 15 crawlers = 15 pages/second max for domain          │
│    - Prevents overwhelming target server                   │
│                                                              │
│ 3. Bloom Filter Capacity:                                 │
│    - Bloom filter: 10B entries capacity                   │
│    - Current usage: ~1B URLs seen                         │
│    - 100K URLs/second = 8.6B/day                          │
│    - False positive rate increases over time               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Crawl rate: 15 pages/second (respects rate limits)      │
│ - Kafka buffers: 100K URLs/second (no loss)               │
│ - Total crawl time: ~7 days for 10M pages                 │
│ - No service degradation                                   │
│ - Target server not overwhelmed                            │
└─────────────────────────────────────────────────────────────┘
```

### Edge Case Scenario: Infinite Loop Detection

```
Scenario: Crawler encounters circular link structure
Edge Case: Page A links to B, B links to C, C links back to A

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Crawler fetches Page A                              │
│ - URL: https://example.com/page-a                         │
│ - Extracts link: https://example.com/page-b               │
│ - Adds to queue (not seen in bloom filter)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+1s: Crawler fetches Page B                              │
│ - URL: https://example.com/page-b                         │
│ - Extracts link: https://example.com/page-c               │
│ - Adds to queue                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+2s: Crawler fetches Page C                              │
│ - URL: https://example.com/page-c                         │
│ - Extracts link: https://example.com/page-a (LOOP!)       │
│ - Bloom filter check: Already seen (false positive or real)│
│                                                              │
│ Problem: Infinite loop if bloom filter allows              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Solution: Depth Limiting + Cycle Detection                │
│                                                              │
│ 1. Depth Tracking:                                        │
│    - Track crawl depth from seed URL                      │
│    - Max depth: 10 levels (configurable)                  │
│    - Page A (depth 1) → Page B (depth 2) → Page C (depth 3)│
│    - If depth > 10, skip URL                              │
│                                                              │
│ 2. Cycle Detection:                                       │
│    - Maintain path history for current crawl path         │
│    - If URL appears in path, skip (cycle detected)        │
│    - Path: [A, B, C, A] → Cycle detected, skip A         │
│                                                              │
│ 3. Domain-Level Loop Detection:                           │
│    - Count URLs per domain in queue                       │
│    - If domain has > 100K URLs, limit to top 100K         │
│    - Prevents single domain from consuming all resources   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - Infinite loops prevented                                 │
│ - Crawl depth bounded                                      │
│ - Resources distributed across domains                     │
│ - No crawler trap exploitation                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Analysis

### Major Cost Drivers

| Component | Monthly Cost | % of Total |
|-----------|--------------|------------|
| Crawler EC2 (15 × c5.2xlarge) | $15,000 | 35% |
| Kafka (MSK 3-broker) | $8,000 | 19% |
| Redis (ElastiCache) | $3,000 | 7% |
| S3 Storage (100 TB) | $2,300 | 5% |
| Data Transfer | $10,000 | 23% |
| PostgreSQL (RDS) | $2,000 | 5% |
| Other | $2,700 | 6% |
| **Total** | **$43,000** | 100% |

### Cost Optimization Strategies

**Current Monthly Cost:** $43,000
**Target Monthly Cost:** $30,000 (30% reduction)

**Top 3 Cost Drivers:**
1. **Data Transfer:** $10,000/month (23%) - Largest single component
2. **Crawler EC2:** $15,000/month (35%) - Multiple crawler instances
3. **Kafka (MSK):** $8,000/month (19%) - Message queue infrastructure

**Optimization Strategies (Ranked by Impact):**

1. **Spot Instances for Crawlers (60% savings):**
   - **Current:** On-demand c5.2xlarge instances (15 × $1,000 = $15K)
   - **Optimization:** Use Spot instances (60% savings)
   - **Savings:** $9,000/month (60% of $15K)
   - **Trade-off:** Possible interruptions (acceptable for crawlers, can resume)

2. **Data Transfer Optimization (40% savings):**
   - **Current:** High data transfer costs from crawling
   - **Optimization:** 
     - Use CloudFront for popular pages (reduce origin requests)
     - Compress responses before transfer
     - Use regional endpoints to reduce cross-region transfer
   - **Savings:** $4,000/month (40% of $10K)
   - **Trade-off:** Slight complexity increase

3. **S3 Intelligent Tiering (30% storage savings):**
   - **Current:** All pages in standard storage
   - **Optimization:** 
     - Enable S3 Intelligent-Tiering (automatic cost optimization)
     - Move pages > 30 days to Infrequent Access
     - Move pages > 90 days to Glacier
   - **Savings:** $690/month (30% of $2.3K storage)
   - **Trade-off:** Slower access to old pages (acceptable for archive)

4. **Reserved Capacity (40% savings on stable workloads):**
   - **Current:** On-demand instances for Kafka and Redis
   - **Optimization:** 1-year Reserved Instances
   - **Savings:** $4,400/month (40% of $11K Kafka+Redis)
   - **Trade-off:** Less flexibility, but acceptable for stable workloads

5. **Right-sizing:**
   - **Current:** Over-provisioned for safety
   - **Optimization:** 
     - Monitor actual usage for 1 month
     - Reduce instance sizes where possible
     - Use auto-scaling more aggressively
   - **Savings:** $1,500/month (10% of $15K compute)
   - **Trade-off:** Need careful monitoring to avoid performance issues

**Total Potential Savings:** $19,590/month (46% reduction)
**Optimized Monthly Cost:** $23,410/month

**Cost per Operation Breakdown:**

| Operation | Current Cost | Optimized Cost | Reduction |
|-----------|--------------|----------------|-----------|
| Page Crawl | $0.000017 | $0.000009 | 47% |
| Page Storage (per GB/month) | $0.023 | $0.016 | 30% |
| URL Processing | $0.00001 | $0.000005 | 50% |

**Implementation Priority:**
1. **Phase 1 (Month 1):** Spot instances, Data transfer optimization → $13K savings
2. **Phase 2 (Month 2):** Reserved capacity, S3 optimization → $5.09K savings
3. **Phase 3 (Month 3):** Right-sizing → $1.5K savings

**Monitoring & Validation:**
- Track cost reduction weekly
- Monitor crawl rate (ensure no degradation)
- Monitor storage costs (target < $1.6K/month)
- Review and adjust quarterly

### Cost per Page

**Calculation:**
- 1000 pages/second = 2.6 billion pages/month
- Cost: $43,000/month
- **Cost per page: $0.000017**
- **Cost per million pages: $17**

---

## Summary

| Aspect | Implementation | Key Metric |
|--------|----------------|------------|
| Distribution | Kafka partitioning | 15 partitions |
| Rate Limiting | Per-domain | Respect robots.txt |
| Circuit Breakers | Resilience4j | 50% failure threshold |
| Monitoring | Prometheus + Grafana | 1000 pages/s target |
| Security | URL validation, SSRF prevention | Block private IPs |
| DR | Multi-region | RTO < 30 min |
| Cost | $43,000/month | $17 per million pages |

