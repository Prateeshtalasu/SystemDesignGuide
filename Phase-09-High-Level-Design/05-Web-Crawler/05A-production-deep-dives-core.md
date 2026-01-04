# Web Crawler - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for URL distribution), caching strategy (robots.txt and DNS), and the key algorithms (URL frontier, duplicate detection, politeness).

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Web Crawler Context?

In a web crawler, Kafka serves as the backbone for distributed URL processing:

1. **URL Distribution**: Routes discovered URLs to appropriate crawler pods
2. **Crawled Documents**: Streams crawled pages to downstream processors
3. **Coordination**: Enables horizontal scaling without central bottleneck

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key |
|-------|---------|------------|-----|
| `discovered-urls` | New URLs to crawl | 15 (one per crawler) | domain_hash |
| `crawled-documents` | Crawled page content | 10 | url_hash |
| `crawl-errors` | Failed crawl attempts | 5 | domain |

**Why Domain-Based Partitioning?**

```java
public class URLPartitioner {
    
    public int partition(String url, int numPartitions) {
        String domain = extractDomain(url);
        return Math.abs(domain.hashCode()) % numPartitions;
    }
}
```

Benefits:
- Same domain always goes to same crawler
- Politeness enforced locally (no coordination)
- robots.txt cached per crawler
- Connection pooling per domain

**Producer Configuration:**

```java
@Configuration
public class CrawlerKafkaConfig {
    
    @Bean
    public ProducerFactory<String, DiscoveredURL> urlProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durability settings (URLs are valuable)
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Kafka Technology-Level

**Normal Flow: URL Discovery and Distribution**

```
Step 1: Crawler Fetches Page
┌─────────────────────────────────────────────────────────────┐
│ Crawler Pod 3: Fetches https://example.com/page1            │
│ Parser: Extracts links: [/page2, /page3, https://other.com/x]│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Process Each Discovered URL
┌─────────────────────────────────────────────────────────────┐
│ For each link:                                               │
│ 1. Normalize URL                                             │
│ 2. Check bloom filter (Redis): Duplicate?                   │
│ 3. Calculate priority                                        │
│ 4. Determine partition: hash(domain) % 15                   │
│    - example.com → partition 3                              │
│    - other.com → partition 7                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Publish to Kafka
┌─────────────────────────────────────────────────────────────┐
│ Kafka Producer:                                              │
│   Topic: discovered-urls                                     │
│   Partition: 3 (for example.com URLs)                       │
│   Key: "example.com"                                         │
│   Message: {                                                 │
│     "url": "https://example.com/page2",                     │
│     "priority": 8,                                           │
│     "discovered_from": "https://example.com/page1",         │
│     "depth": 2                                               │
│   }                                                          │
│   Latency: < 1ms                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Crawler Consumes URLs
┌─────────────────────────────────────────────────────────────┐
│ Crawler Pod 3 (assigned to partition 3):                    │
│ - Polls partition 3                                         │
│ - Receives batch of 100 URLs                                │
│ - All URLs from example.com (same domain)                   │
│ - Enforces politeness (crawl delay per domain)             │
│ - Fetches URLs sequentially                                 │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Crawler Pod Crash**

```
Scenario: Crawler pod crashes while processing URLs

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Crawler Pod 3 processing partition 3                   │
│ T+30s: Pod crashes (OOM, hardware failure)                 │
│ T+30s: Kafka offset NOT committed                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+60s: Kafka consumer group rebalance                      │
│ - Coordinator detects pod 3 left group                      │
│ - Reassigns partition 3 to pod 8                            │
│ - Pod 8 starts from last committed offset                   │
│ - Receives URLs again (offset not committed)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Pod 8 Processing:                                            │
│ - Receives URLs from partition 3                            │
│ - Checks bloom filter: Some URLs already crawled            │
│ - Skips duplicates (idempotent)                             │
│ - Crawls new URLs                                            │
│ - Commits offset after successful crawl                     │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same URL discovered multiple times (duplicate links)

Solution: Bloom filter + database deduplication
1. Bloom filter: Fast duplicate check (may have false positives)
2. Database: Final deduplication (no false positives)
3. URL hash: Unique identifier per URL

Example:
- URL discovered twice → Bloom filter check → Database check → Deduplicated
```

**Traffic Spike Handling:**

```
Scenario: Large site discovered, millions of URLs to crawl

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 URLs/second discovered                        │
│ Spike: 100,000 URLs/second (large site discovered)          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Buffering:                                             │
│ - 100K URLs/second buffered in Kafka                        │
│ - 7-day retention = 60B URLs capacity                      │
│ - No URL loss                                                │
│                                                              │
│ Crawler Scaling:                                             │
│ - Auto-scale: 15 → 100 crawler pods                         │
│ - Each pod: 1,000 URLs/second                               │
│ - Throughput: 100K URLs/second                               │
│                                                              │
│ Result:                                                      │
│ - URLs queued in Kafka (no loss)                            │
│ - Crawling continues at steady rate                         │
│ - All URLs eventually crawled                                │
└─────────────────────────────────────────────────────────────┘
```

**Hot Partition Mitigation:**

```
Problem: Popular domain has millions of URLs, all go to one partition

Mitigation:
1. Partition by domain (distributes across domains)
2. Multiple crawler pods per partition (if needed)
3. Monitor partition lag per partition

If hot partition occurs:
- Partition receives 10K URLs/second (large domain)
- Crawlers auto-scale to handle load
- Politeness enforced per domain (no overload)
```

**Recovery Behavior:**

```
Auto-healing:
- Crawler crash → Kafka rebalance → URLs reassigned → Retry
- Network timeout → Retry with exponential backoff → Automatic
- Crawl failure → Log error, continue with next URL → Automatic

Human intervention:
- Persistent crawl failures → Check target site status
- High error rate → Adjust crawl delay or investigate
- Partition imbalance → Rebalance partitions
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Crawler Context?

Redis provides critical caching for crawler operations:

1. **Robots.txt Cache**: Avoid re-fetching robots.txt
2. **DNS Cache**: Reduce DNS lookup latency
3. **Domain State**: Track crawl timing per domain
4. **Bloom Filter**: Distributed URL deduplication

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Robots.txt | `robots:{domain}` | 24 hours | Store parsed rules |
| DNS | `dns:{domain}` | 1 hour | IP address cache |
| Domain State | `domain:state:{domain}` | None | Last crawl time, delay |
| Bloom Filter | `bloom:urls` | None | URL deduplication |

**Robots.txt Caching:**

```java
@Service
public class RobotsCacheService {
    
    private final RedisTemplate<String, RobotsRules> redis;
    private static final Duration TTL = Duration.ofHours(24);
    
    public RobotsRules getRules(String domain) {
        String key = "robots:" + domain;
        RobotsRules cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Fetch and parse robots.txt
        RobotsRules rules = fetchAndParse(domain);
        
        // Cache with TTL
        redis.opsForValue().set(key, rules, TTL);
        
        return rules;
    }
}
```

**Domain State Management:**

```java
@Service
public class DomainStateService {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean canCrawl(String domain) {
        String key = "domain:state:" + domain;
        String lastCrawlStr = (String) redis.opsForHash().get(key, "last_crawl");
        String delayStr = (String) redis.opsForHash().get(key, "delay_ms");
        
        if (lastCrawlStr == null) {
            return true;  // Never crawled
        }
        
        long lastCrawl = Long.parseLong(lastCrawlStr);
        int delay = delayStr != null ? Integer.parseInt(delayStr) : 1000;
        
        return System.currentTimeMillis() - lastCrawl >= delay;
    }
    
    public void recordCrawl(String domain) {
        String key = "domain:state:" + domain;
        redis.opsForHash().put(key, "last_crawl", 
            String.valueOf(System.currentTimeMillis()));
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Redis Technology-Level

**Normal Flow: Robots.txt and DNS Caching**

```
Step 1: Crawler Needs to Fetch URL
┌─────────────────────────────────────────────────────────────┐
│ URL: https://example.com/page1                              │
│ Domain: example.com                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Robots.txt Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET robots:example.com                                │
│ Result: HIT (cached from previous crawl)                   │
│ Value: {                                                     │
│   "user-agent": "*",                                        │
│   "disallow": ["/admin", "/private"],                       │
│   "crawl-delay": 1,                                         │
│   "cached_at": "2024-01-20T09:00:00Z"                      │
│ }                                                            │
│ Latency: ~1ms                                                │
│ Decision: /page1 is allowed                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Check DNS Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET dns:example.com                                  │
│ Result: HIT                                                  │
│ Value: "93.184.216.34"                                      │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Check Crawl Delay
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET domain:last_crawl:example.com                     │
│ Result: "1705312800000" (last crawl timestamp)             │
│ Current: 1705312900000 (10 seconds later)                  │
│ Crawl delay: 1 second (from robots.txt)                    │
│ Decision: Can crawl (10s > 1s delay)                        │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Redis Cache Miss**

```
Scenario: New domain, no cached robots.txt or DNS

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: Redis: GET robots:newdomain.com → MISS              │
│ T+1ms: Fetch robots.txt from newdomain.com                │
│ T+200ms: robots.txt fetched                                │
│ T+201ms: Parse and cache in Redis (24-hour TTL)            │
│ T+202ms: Redis: GET dns:newdomain.com → MISS              │
│ T+203ms: DNS lookup: newdomain.com → 1.2.3.4              │
│ T+250ms: Cache DNS in Redis (1-hour TTL)                  │
│ T+251ms: Proceed with crawl                                 │
│ Total latency: ~250ms (vs ~1ms with cache)                 │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same robots.txt cached multiple times

Solution: Redis SET is idempotent
1. SET robots:example.com {data} EX 86400
2. If key exists, overwrites (idempotent)
3. TTL refreshed on each set

Example:
- robots.txt cached twice → Same Redis key → Overwrites → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: Millions of URLs discovered, many cache lookups

┌─────────────────────────────────────────────────────────────┐
│ Normal: 1,000 cache lookups/second                          │
│ Spike: 100,000 cache lookups/second                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Redis Handling:                                               │
│ - GET operations: O(1) per operation                        │
│ - Pipeline operations: Batch 100 operations                  │
│ - Connection pooling: Reuse connections                     │
│ - Result: 100K ops/second handled by Redis cluster         │
│                                                              │
│ Cache Hit Rate:                                              │
│ - robots.txt: 95% hit rate (24-hour TTL)                    │
│ - DNS: 90% hit rate (1-hour TTL)                           │
│ - Result: Most lookups served from cache                    │
└─────────────────────────────────────────────────────────────┘
```

**Hot Key Mitigation:**

```
Problem: Popular domain's robots.txt accessed frequently

Mitigation:
1. Long TTL: 24-hour TTL reduces refresh frequency
2. Read replicas: Distribute reads across replicas
3. Local cache: Cache hottest robots.txt in crawler memory

If hot key occurs:
- Monitor key access patterns
- Add read replicas
- Consider pre-warming cache for popular domains
```

**Recovery Behavior:**

```
Auto-healing:
- Redis node failure → Cluster promotes replica → Automatic
- Network timeout → Retry → Automatic
- Cache miss → Fetch and cache → Automatic

Human intervention:
- Cluster-wide failure → Manual failover to DR region
- Persistent hot keys → Scale Redis or optimize access pattern
- Cache corruption → Clear and rebuild cache
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when a cached robots.txt or DNS entry expires and many crawler threads simultaneously try to fetch it, overwhelming the target server.

**Scenario:**
```
T+0s:   Popular domain's robots.txt cache expires
T+0s:   1,000 crawler threads need to crawl that domain
T+0s:   All 1,000 threads see cache MISS
T+0s:   All 1,000 threads hit target server simultaneously
T+1s:   Target server overwhelmed, politeness violated
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one thread fetches robots.txt or DNS data:

1. **Distributed locking**: Only one thread fetches from target server on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class RobotsCacheService {
    
    private final RedisTemplate<String, RobotsRules> redis;
    private final DistributedLock lockService;
    
    public RobotsRules getRules(String domain) {
        String key = "robots:" + domain;
        
        // 1. Try cache first
        RobotsRules cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:robots:" + domain;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to default rules (allow all) if cache unavailable
            return RobotsRules.allowAll();
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Fetch from target server (only one thread does this)
            RobotsRules rules = fetchAndParse(domain);
            
            // 5. Populate cache with TTL jitter (add ±10% random jitter to 24h TTL)
            Duration baseTtl = Duration.ofHours(24);
            long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
            Duration ttl = baseTtl.plusSeconds(jitter);
            
            redis.opsForValue().set(key, rules, ttl);
            
            return rules;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time
- Maintains politeness to target servers

---

## 3. Database (PostgreSQL)

### A) CONCEPT: Why PostgreSQL for Web Crawler?

PostgreSQL is used for:
- **URL Deduplication**: Final check after bloom filter (no false positives)
- **Crawl Metadata**: Track which URLs have been crawled, when, status
- **Document Metadata**: Store document information for indexing pipeline
- **Crawl Jobs**: Manage crawl job configurations and schedules

### B) OUR USAGE: How We Use PostgreSQL

**Schema:**
- `crawled_urls` table: Tracks all crawled URLs (deduplication)
- `crawl_jobs` table: Crawl job configurations
- `documents` table: Document metadata for indexing
- Indexes: Primary key on `url_hash`, index on `domain`, index on `crawled_at`

**Operations:**
- **URL Deduplication**: Check if URL already crawled
- **Insert Crawl Record**: Record successful crawl
- **Update Status**: Update crawl status (success/failure)
- **Query by Domain**: Get all URLs for a domain

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: URL Deduplication**

```
Step 1: URL Discovered
┌─────────────────────────────────────────────────────────────┐
│ Parser extracts: https://example.com/page1                  │
│ Normalize: example.com/page1                                │
│ Hash: SHA256(url) → abc123def456...                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Bloom Filter Check (Fast, Probabilistic)
┌─────────────────────────────────────────────────────────────┐
│ Redis: BF.EXISTS crawled_urls_bloom abc123def456...        │
│ Result: false (not seen)                                     │
│ Note: Bloom filter may have false positives, but no false   │
│       negatives. If it says "exists", we still check DB.    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Database Check (Accurate, Final)
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT url_hash FROM crawled_urls               │
│              WHERE url_hash = 'abc123def456...'              │
│ Result: 0 rows (not crawled)                                 │
│ Decision: URL is new, proceed to crawl                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: After Successful Crawl
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                               │
│ PostgreSQL: INSERT INTO crawled_urls                         │
│              (url_hash, url, domain, crawled_at, status)    │
│              VALUES ('abc123...', 'https://...',            │
│                      'example.com', NOW(), 'success')       │
│ PostgreSQL: INSERT INTO documents                           │
│              (url_hash, title, content_hash, size_bytes)    │
│              VALUES ('abc123...', 'Page Title', 'sha256...',│
│                      524288)                                │
│ PostgreSQL: COMMIT TRANSACTION                              │
│                                                              │
│ Redis: BF.ADD crawled_urls_bloom abc123def456...           │
│ (Update bloom filter for future fast checks)               │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Database Connection Timeout**

```
Step 1: URL Deduplication Attempt
┌─────────────────────────────────────────────────────────────┐
│ Redis Bloom Filter: false (not seen)                        │
│ PostgreSQL: SELECT url_hash FROM crawled_urls WHERE ...    │
│ Connection timeout after 5 seconds                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Circuit Breaker Opens
┌─────────────────────────────────────────────────────────────┐
│ Circuit breaker: 50% failure rate detected                  │
│ State: OPEN (rejecting requests)                            │
│ Fallback: Skip database check, rely on bloom filter only    │
│ Risk: May crawl duplicate URLs (acceptable degradation)     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Retry with Exponential Backoff
┌─────────────────────────────────────────────────────────────┐
│ T+10s: Retry connection (backoff: 1s)                       │
│ T+20s: Retry connection (backoff: 2s)                       │
│ T+40s: Retry connection (backoff: 4s)                       │
│ T+80s: Retry connection (backoff: 8s)                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Recovery
┌─────────────────────────────────────────────────────────────┐
│ T+120s: Connection successful                                │
│ Circuit breaker: Half-open state (allow 10 test requests)   │
│ T+125s: 10 requests succeed                                  │
│ Circuit breaker: CLOSED (normal operation)                  │
│ Resume full deduplication                                    │
└─────────────────────────────────────────────────────────────┘
```

**Duplicate/Idempotency Handling:**

```
Scenario: Same URL Discovered Twice
┌─────────────────────────────────────────────────────────────┐
│ Discovery 1:                                                │
│   Bloom filter: false → DB check: not found → Crawl        │
│   DB insert: url_hash='abc123...'                          │
│                                                              │
│ Discovery 2 (5 minutes later, different crawler):          │
│   Bloom filter: true (may be false positive)                │
│   DB check: SELECT ... WHERE url_hash='abc123...'          │
│   Result: 1 row found                                       │
│   Decision: Skip crawl (already crawled)                    │
│   No duplicate crawl performed                              │
└─────────────────────────────────────────────────────────────┘
```

**Traffic Spike Handling: Hot Domain**

```
Scenario: Popular Domain Discovered (10,000 URLs)
┌─────────────────────────────────────────────────────────────┐
│ T+0s:   10,000 URLs from example.com discovered             │
│ T+1s:   All URLs queued in Kafka (partitioned by domain)    │
│ T+2s:   Crawler starts processing URLs                      │
│         - Each URL: Bloom filter check → DB check           │
│         - DB queries: 10,000 SELECT statements               │
│                                                              │
│ Problem: Database connection pool exhausted                 │
│ Solution: Connection pooling (max 100 connections)          │
│           Batch queries: SELECT url_hash FROM crawled_urls  │
│                          WHERE url_hash IN (?, ?, ...)      │
│                          (batch of 1000 URLs)               │
│                                                              │
│ Result: 10 batch queries instead of 10,000 individual       │
│         queries (100x reduction)                            │
└─────────────────────────────────────────────────────────────┘
```

**Recovery Behavior: Database Primary Failure**

```
T+0s:    PostgreSQL primary fails (disk failure)
    ↓
T+5s:    Health check detects failure
    ↓
T+10s:   Automatic failover to replica
    ↓
T+15s:   Replica promoted to primary
    ↓
T+20s:   Application reconnects to new primary
    ↓
T+25s:   Normal operations resume
    ↓
RTO: < 30 seconds
RPO: 0 (synchronous replication for critical data)
```

---

## 4. Search - NOT APPLICABLE

The web crawler doesn't implement search functionality. It's a data collection system that feeds into a search engine's indexing pipeline.

---

## 5. URL Frontier Deep Dive

### Two-Level Queue Architecture

```java
public class URLFrontier {
    
    // Front queues: priority-based
    private Map<Integer, Queue<String>> frontQueues;  // priority -> domains
    
    // Back queues: domain-based (for politeness)
    private Map<String, Queue<URLEntry>> backQueues;  // domain -> URLs
    
    // Domain metadata
    private Map<String, DomainState> domainStates;
    
    public URLEntry getNext() {
        // 1. Select priority level (weighted random)
        int priority = selectPriority();
        Queue<String> frontQueue = frontQueues.get(priority);
        
        // 2. Find a domain that's ready to crawl
        for (String domain : frontQueue) {
            DomainState state = domainStates.get(domain);
            
            if (state.isReadyToCrawl()) {
                // 3. Get URL from domain's back queue
                Queue<URLEntry> backQueue = backQueues.get(domain);
                URLEntry url = backQueue.poll();
                
                // 4. Update domain state
                state.markCrawled();
                
                // 5. If domain queue empty, remove from front queue
                if (backQueue.isEmpty()) {
                    frontQueue.remove(domain);
                }
                
                return url;
            }
        }
        
        return null;  // No URLs ready
    }
}
```

### Priority Calculation

```java
public class PriorityCalculator {
    
    public int calculatePriority(String url, URLMetadata metadata) {
        double score = 0.0;
        
        // 1. Domain importance (PageRank-based)
        String domain = extractDomain(url);
        double domainScore = domainRanks.getOrDefault(domain, 0.1);
        score += domainScore * 0.4;
        
        // 2. URL depth (closer to root = higher priority)
        int depth = metadata.getDepth();
        double depthScore = 1.0 / (1 + depth);
        score += depthScore * 0.2;
        
        // 3. URL pattern (news, article = higher)
        if (isNewsUrl(url)) {
            score += 0.2;
        }
        
        // 4. Freshness need
        if (metadata.getLastCrawled() == null) {
            score += 0.2;  // Never crawled = high priority
        }
        
        // Convert to priority level (1-5)
        if (score > 0.8) return 1;
        if (score > 0.6) return 2;
        if (score > 0.4) return 3;
        if (score > 0.2) return 4;
        return 5;
    }
}
```

---

## 5. Duplicate Detection

### URL-Level Deduplication (Bloom Filter)

```java
public class URLDeduplicator {
    
    private BloomFilter<String> bloomFilter;
    
    public URLDeduplicator(long expectedUrls, double falsePositiveRate) {
        // 10B URLs, 0.1% FP rate = ~18 GB
        this.bloomFilter = BloomFilter.create(
            Funnels.stringFunnel(StandardCharsets.UTF_8),
            expectedUrls,
            falsePositiveRate
        );
    }
    
    public boolean isDuplicate(String url) {
        String normalizedUrl = normalizeUrl(url);
        String urlHash = hash(normalizedUrl);
        return bloomFilter.mightContain(urlHash);
    }
    
    public void markSeen(String url) {
        String normalizedUrl = normalizeUrl(url);
        String urlHash = hash(normalizedUrl);
        bloomFilter.put(urlHash);
    }
}
```

### Content-Level Deduplication (SimHash)

```java
public class ContentDeduplicator {
    
    private static final int HASH_BITS = 64;
    private static final int MAX_HAMMING_DISTANCE = 3;
    
    public long computeSimHash(String content) {
        String[] words = content.toLowerCase().split("\\s+");
        
        // Create 3-word shingles
        Set<String> shingles = new HashSet<>();
        for (int i = 0; i <= words.length - 3; i++) {
            shingles.add(words[i] + " " + words[i+1] + " " + words[i+2]);
        }
        
        // Compute SimHash
        int[] vector = new int[HASH_BITS];
        for (String shingle : shingles) {
            long hash = MurmurHash3.hash64(shingle.getBytes());
            for (int i = 0; i < HASH_BITS; i++) {
                vector[i] += ((hash & (1L << i)) != 0) ? 1 : -1;
            }
        }
        
        // Convert to fingerprint
        long fingerprint = 0;
        for (int i = 0; i < HASH_BITS; i++) {
            if (vector[i] > 0) {
                fingerprint |= (1L << i);
            }
        }
        return fingerprint;
    }
    
    public boolean isNearDuplicate(long hash1, long hash2) {
        return Long.bitCount(hash1 ^ hash2) <= MAX_HAMMING_DISTANCE;
    }
}
```

---

## 6. Politeness Enforcement

### Robots.txt Implementation

```java
public class RobotsManager {
    
    private Map<String, RobotsRules> cache = new ConcurrentHashMap<>();
    
    public RobotsRules getRules(String domain) {
        RobotsRules cached = cache.get(domain);
        if (cached != null && !cached.isExpired()) {
            return cached;
        }
        
        try {
            String robotsUrl = "https://" + domain + "/robots.txt";
            HttpResponse response = httpClient.fetch(robotsUrl, 10000);
            
            if (response.getStatusCode() == 200) {
                RobotsRules rules = parseRobotsTxt(response.getBody());
                cache.put(domain, rules);
                return rules;
            } else if (response.getStatusCode() == 404) {
                return RobotsRules.allowAll();
            }
        } catch (Exception e) {
            log.warn("Failed to fetch robots.txt for {}", domain);
        }
        
        return RobotsRules.defaultRules();
    }
}
```

### Crawl Delay Enforcement

```java
public class PolitenessEnforcer {
    
    private Map<String, Long> lastCrawlTimes = new ConcurrentHashMap<>();
    private static final int DEFAULT_DELAY_MS = 1000;
    
    public void waitForPoliteness(String domain) throws InterruptedException {
        Long lastCrawl = lastCrawlTimes.get(domain);
        int delay = getCrawlDelay(domain);
        
        if (lastCrawl != null) {
            long elapsed = System.currentTimeMillis() - lastCrawl;
            long waitTime = delay - elapsed;
            
            if (waitTime > 0) {
                Thread.sleep(waitTime);
            }
        }
    }
    
    public void recordCrawl(String domain) {
        lastCrawlTimes.put(domain, System.currentTimeMillis());
    }
}
```

---

## 7. HTTP Fetching

### Async HTTP Client

```java
public class AsyncFetcher {
    
    private final HttpClient httpClient;
    private final Semaphore concurrencyLimit;
    
    public CompletableFuture<CrawlResult> fetch(String url) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                concurrencyLimit.acquire();
                return doFetch(url);
            } finally {
                concurrencyLimit.release();
            }
        });
    }
    
    private CrawlResult doFetch(String url) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("User-Agent", USER_AGENT)
            .header("Accept", "text/html")
            .header("Accept-Encoding", "gzip")
            .timeout(Duration.ofSeconds(30))
            .GET()
            .build();
        
        HttpResponse<byte[]> response = httpClient.send(request, 
            HttpResponse.BodyHandlers.ofByteArray());
        
        return CrawlResult.success(url, response.statusCode(), 
            response.body(), response.headers());
    }
}
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| URL Distribution | Kafka | 15 partitions, domain-based key |
| Robots Cache | Redis | 24-hour TTL |
| Domain State | Redis | Per-domain tracking |
| URL Dedup | Bloom Filter | 18 GB, 0.1% FP rate |
| Content Dedup | SimHash | 64-bit, hamming ≤ 3 |
| Politeness | Per-domain delays | Default 1s, respect robots.txt |
| HTTP Fetching | Async NIO | 30s timeout, gzip support |

