# URL Shortener - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for analytics), caching (Redis), and search capabilities. These are the fundamental building blocks that enable the URL shortener to meet its latency and throughput requirements.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant data pipelines. Unlike traditional message queues (RabbitMQ, ActiveMQ), Kafka stores messages durably on disk and allows multiple consumers to read the same data independently.

**What problems does Kafka solve?**

1. **Buffering**: Absorbs traffic spikes without overwhelming downstream services
2. **Decoupling**: Producers and consumers operate independently
3. **Ordering**: Guarantees message order within a partition
4. **Durability**: Messages persist even if consumers are temporarily down
5. **Replay**: Consumers can re-read historical messages

**Core Kafka concepts:**

| Concept | Definition |
|---------|------------|
| **Topic** | A named stream of messages (like a database table) |
| **Partition** | A topic is split into partitions for parallelism |
| **Offset** | A unique ID for each message within a partition |
| **Consumer Group** | A group of consumers that share the work of reading a topic |
| **Broker** | A Kafka server that stores and serves messages |

### B) OUR USAGE: How We Use Kafka Here

**Why async vs sync for analytics?**

The redirect operation is latency-critical (< 50ms target). If we recorded analytics synchronously:
- Database write: +10-20ms
- GeoIP lookup: +5-10ms
- User agent parsing: +2-5ms
- Total added latency: 17-35ms (unacceptable)

By using Kafka, the redirect returns immediately after publishing an event (< 1ms), and analytics processing happens asynchronously.

**Events in our system:**

| Event | Purpose | Volume |
|-------|---------|--------|
| `click-event` | Records each redirect for analytics | 12,000/second peak |

**Topic/Queue Design:**

```
Topic: click-events
Partitions: 12
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
```

**Why 12 partitions?**
- Allows up to 12 parallel consumers
- Matches expected consumer count for throughput
- Can increase later if needed (but not decrease)

**Partition Key Choice:**

We partition by `short_code` hash. This ensures:
- All events for the same URL go to the same partition
- Ordering is preserved per URL (important for accurate click counting)
- Even distribution across partitions

**Hot Partition Risk:**

If a celebrity tweets a viral URL, one partition could receive disproportionate traffic. Mitigations:
1. Use hash of `short_code` (distributes evenly)
2. Monitor partition lag per partition
3. If hot, temporarily increase partition count

**Consumer Group Design:**

```
Consumer Group: analytics-consumers
Consumers: 4 instances (each handles 3 partitions)
```

**Ordering Guarantees:**

- **Per-key ordering**: Guaranteed (all events for same short_code go to same partition)
- **Global ordering**: Not guaranteed (not needed for analytics)

**Offset Management:**

We use manual offset commits to ensure at-least-once delivery:

```java
@KafkaListener(topics = "click-events", groupId = "analytics-consumers")
public void consume(List<ClickEvent> events, Acknowledgment ack) {
    try {
        processEvents(events);
        ack.acknowledge();  // Commit offset only after successful processing
    } catch (Exception e) {
        // Don't acknowledge, will be reprocessed
        log.error("Failed to process events", e);
    }
}
```

**Deduplication Strategy:**

Since we use at-least-once delivery, duplicates are possible. We handle this by:
1. Using idempotent inserts (UPSERT with unique constraint)
2. Deduplicating by `(short_code, timestamp, ip_hash)` combination
3. Accepting slight over-counting (analytics doesn't need 100% accuracy)

**Outbox Pattern:**

Not used here because:
- Analytics events are not critical (losing some is acceptable)
- We don't need transactional guarantees between URL creation and event publishing
- Simpler architecture without outbox overhead

### C) REAL STEP-BY-STEP SIMULATION

**Event Flow:**

```
User clicks short URL "abc123"
    ↓
URL Service receives GET /abc123
    ↓
Redis lookup: GET url:abc123 → returns original URL
    ↓
Kafka Producer publishes ClickEvent:
    {
      "shortCode": "abc123",
      "timestamp": 1705312800000,
      "ipHash": "a1b2c3...",
      "userAgent": "Mozilla/5.0...",
      "referrer": "twitter.com"
    }
    ↓
Partition selected by hash(abc123) % 12 = partition 7
    ↓
301 Redirect returned to user (< 30ms total)
    ↓
[ASYNC] Consumer polls partition 7
    ↓
[ASYNC] Batch of 500 events processed:
    - GeoIP lookup for each IP
    - User agent parsing
    - Batch INSERT to ClickHouse
    ↓
[ASYNC] Offset committed
    ↓
[ASYNC] Redis counter incremented: INCR clicks:abc123
```

**Failure Scenarios:**

**Q: What if consumer crashes mid-batch?**

A: The offset is not committed, so the batch will be reprocessed by another consumer in the group. This may cause duplicate processing, but our deduplication handles it.

**Q: What if duplicate events occur?**

A: We accept slight over-counting for analytics. For critical metrics, we deduplicate by `(short_code, timestamp, ip_hash)` before aggregation.

**Q: What if a partition becomes hot?**

A: Monitor partition lag. If one partition consistently lags:
1. Add more consumers (up to partition count)
2. If still hot, consider re-partitioning with more partitions
3. Implement backpressure by slowing down producers

**Q: How does retry work?**

A: Producer retries with exponential backoff:
```java
config.put(ProducerConfig.RETRIES_CONFIG, 3);
config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);  // 100ms, 200ms, 400ms
```

Failed events after retries are logged and dropped (analytics loss is acceptable).

**Q: What breaks first and what degrades gracefully?**

A: Order of failure:
1. **Kafka broker down**: Events buffer in producer memory, then fail. Redirects still work.
2. **Consumer lag**: Analytics delayed but redirects unaffected.
3. **All Kafka down**: Events lost, redirects still work (graceful degradation).

### Event Schema

```java
public class ClickEvent {
    private String shortCode;       // Partition key
    private long timestamp;         // Event time
    private String ipHash;          // SHA-256 of IP (privacy)
    private String userAgent;       // Raw user agent string
    private String referrer;        // HTTP Referer header
    private String acceptLanguage;  // For geo inference

    // Populated by consumer
    private String countryCode;
    private String city;
    private String deviceType;
    private String browser;
    private String os;
}
```

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, ClickEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Reliability settings
        config.put(ProducerConfig.ACKS_CONFIG, "1");  // Leader ack only (speed over durability)
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);

        // Batching for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);  // Wait 5ms to batch

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Why acks=1 instead of acks=all?**

- Analytics events are not critical (can lose some)
- Prioritize redirect latency over analytics durability
- 3x replication still provides reasonable durability

### Consumer Implementation

```java
@Service
public class ClickEventConsumer {

    private final ClickRepository clickRepository;
    private final GeoIpService geoIpService;
    private final UserAgentParser userAgentParser;

    @KafkaListener(topics = "click-events", groupId = "analytics-consumers")
    public void consume(List<ClickEvent> events, Acknowledgment ack) {
        try {
            // Enrich events
            List<Click> clicks = events.stream()
                .map(this::enrichEvent)
                .collect(Collectors.toList());

            // Batch insert
            clickRepository.batchInsert(clicks);

            // Update real-time counters
            updateClickCounts(clicks);

            // Commit offset
            ack.acknowledge();

        } catch (Exception e) {
            log.error("Failed to process click events", e);
            // Don't acknowledge, will be reprocessed
        }
    }

    private Click enrichEvent(ClickEvent event) {
        // GeoIP lookup
        GeoLocation geo = geoIpService.lookup(event.getIpHash());

        // User agent parsing
        UserAgent ua = userAgentParser.parse(event.getUserAgent());

        return Click.builder()
            .shortCode(event.getShortCode())
            .clickedAt(Instant.ofEpochMilli(event.getTimestamp()))
            .ipHash(event.getIpHash())
            .referrer(event.getReferrer())
            .countryCode(geo.getCountryCode())
            .city(geo.getCity())
            .deviceType(ua.getDeviceType())
            .browser(ua.getBrowser())
            .os(ua.getOs())
            .build();
    }

    private void updateClickCounts(List<Click> clicks) {
        // Group by short_code and increment counters
        Map<String, Long> counts = clicks.stream()
            .collect(Collectors.groupingBy(Click::getShortCode, Collectors.counting()));

        counts.forEach((shortCode, count) -> {
            // Increment in Redis (real-time)
            redisTemplate.opsForValue().increment("clicks:" + shortCode, count);
        });
    }
}
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis and Caching Patterns?

Redis is an in-memory data structure store that provides sub-millisecond latency for read/write operations. For URL shortening, caching is critical because:
- Database cannot handle 12,000 QPS directly
- Redirect latency target is < 50ms
- Most URLs follow a power-law distribution (few hot URLs, many cold ones)

**Caching Patterns:**

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **Cache-Aside** | App checks cache, then DB on miss | Read-heavy, can tolerate stale data |
| **Write-Through** | Write to cache and DB together | Need immediate cache consistency |
| **Write-Back** | Write to cache, async to DB | Write-heavy, can lose data |

**We use a hybrid approach:**
- **Write-Through** on URL creation (cache immediately)
- **Cache-Aside** on redirect (check cache, fallback to DB)

**Consistency Tradeoff:**

We accept eventual consistency for redirects:
- If a URL is deleted, cached redirects may work for up to 24 hours
- This is acceptable because deleted URLs are rare
- For immediate invalidation, we explicitly delete from cache

### B) OUR USAGE: How We Use Redis Here

**Exact Keys and Values:**

```
Key:    url:{short_code}
Value:  {original_url}|{redirect_type}|{expires_at}
TTL:    86400 seconds (24 hours)

Example:
Key:    url:abc123
Value:  https://example.com/long/path|301|1735689600
TTL:    86400
```

**Why pipe-delimited string instead of Redis Hash?**
- Single GET operation vs HGETALL
- Less memory overhead
- Simpler parsing

**Who reads Redis?**
- URL Service (on every redirect request)

**Who writes Redis?**
- URL Service (on URL creation)
- URL Service (on cache miss, after DB lookup)

**TTL Behavior:**

- Default TTL: 24 hours
- If URL has expiration, TTL = min(24 hours, time until expiration)
- Why 24 hours? Balances cache hit rate vs memory usage

**Eviction Policy:**

```
maxmemory 16gb
maxmemory-policy allkeys-lru
```

**Why LRU (Least Recently Used)?**
- Automatically evicts cold URLs
- Keeps hot URLs in cache
- No manual management needed

**Invalidation Strategy:**

On URL deletion:
1. Soft delete in database
2. Delete from Redis: `DEL url:{short_code}`
3. Purge from CDN (async)

On URL update:
1. Update in database
2. Delete from Redis (will be repopulated on next read)
3. Purge from CDN

**Why Redis vs Local Cache vs DB Indexes?**

| Option | Latency | Capacity | Consistency |
|--------|---------|----------|-------------|
| Local Cache (Caffeine) | ~0.1ms | Limited by JVM heap | Per-instance, inconsistent |
| Redis | ~1ms | 100s of GB | Shared, consistent |
| DB Index | ~10ms | Unlimited | Always consistent |

We use Redis as primary cache because:
- Shared across all app instances
- Large capacity (16GB+)
- Sub-millisecond latency

We also use local cache (Caffeine) for the hottest 1000 URLs to reduce Redis round-trips.

### C) REAL STEP-BY-STEP SIMULATION

**Cache HIT Path:**

```
Request: GET /abc123
    ↓
URL Service receives request
    ↓
Redis: GET url:abc123
    ↓
Redis returns: "https://example.com/path|301|1735689600"
    ↓
Parse response, return 301 redirect
    ↓
Total latency: ~10-20ms
```

**Cache MISS Path:**

```
Request: GET /xyz789
    ↓
URL Service receives request
    ↓
Redis: GET url:xyz789
    ↓
Redis returns: (nil) - MISS
    ↓
PostgreSQL: SELECT original_url, redirect_type FROM urls WHERE short_code = 'xyz789'
    ↓
DB returns: ("https://example.com/other|301")
    ↓
Redis: SET url:xyz789 "https://example.com/other|301|0" EX 86400
    ↓
Return 301 redirect
    ↓
Total latency: ~30-50ms
```

**Negative Caching (NOT_FOUND):**

```
Request: GET /invalid123
    ↓
Redis: GET url:invalid123 → (nil)
    ↓
PostgreSQL: SELECT ... WHERE short_code = 'invalid123' → no rows
    ↓
Redis: SET url:invalid123 "NOT_FOUND" EX 300  (5 minutes)
    ↓
Return 404 Not Found
```

Why cache NOT_FOUND?
- Prevents repeated DB queries for invalid URLs
- Short TTL (5 min) in case URL is created later
- Protects against enumeration attacks

**Failure Scenarios:**

**Q: What if Redis is down?**

A: Circuit breaker activates:
```java
@CircuitBreaker(name = "redis", fallbackMethod = "getFromDatabase")
public String getFromCache(String shortCode) {
    return redisTemplate.opsForValue().get("url:" + shortCode);
}

public String getFromDatabase(String shortCode, Exception e) {
    log.warn("Redis unavailable, falling back to DB", e);
    return urlRepository.findByShortCode(shortCode);
}
```

Impact:
- All 12,000 QPS hit PostgreSQL directly
- PostgreSQL can handle ~5,000 QPS per replica
- With 2 replicas: 10,000 QPS capacity
- System degrades but doesn't fail completely

**Q: What happens to DB load and latency?**

A: Without cache:
- DB load increases 10x
- Latency increases from 20ms to 50-100ms
- May need to shed load or return errors for some requests

**Q: What is the circuit breaker behavior?**

A: Using Resilience4j:
```java
resilience4j.circuitbreaker:
  instances:
    redis:
      slidingWindowSize: 100
      failureRateThreshold: 50  # Open after 50% failures
      waitDurationInOpenState: 30s
      permittedNumberOfCallsInHalfOpenState: 10
```

**Q: Why this TTL (24 hours)?**

A: Balance between:
- Cache hit rate (longer TTL = higher hit rate)
- Memory usage (longer TTL = more memory)
- Freshness (shorter TTL = fresher data)

24 hours provides ~95% cache hit rate with acceptable memory usage.

**Q: Why write-through vs write-back vs cache-aside?**

A: We use write-through on create because:
- URL must be immediately available after creation
- No risk of data loss (DB is source of truth)
- Simple implementation

We use cache-aside on read because:
- Not all URLs are accessed (no need to pre-warm)
- Natural LRU behavior (hot URLs stay cached)

### Cache Operations Code

**On URL Creation (Write-Through):**

```java
public void cacheUrl(String shortCode, String originalUrl, int redirectType, Long expiresAt) {
    String value = String.format("%s|%d|%d", originalUrl, redirectType,
                                  expiresAt != null ? expiresAt : 0);

    // Calculate TTL: minimum of 24 hours or time until expiration
    long ttlSeconds = 86400; // 24 hours default
    if (expiresAt != null) {
        long secondsUntilExpiry = expiresAt - System.currentTimeMillis() / 1000;
        ttlSeconds = Math.min(ttlSeconds, secondsUntilExpiry);
    }

    redisTemplate.opsForValue().set("url:" + shortCode, value, ttlSeconds, TimeUnit.SECONDS);
}
```

**On Redirect (Cache-Aside with Fallback):**

```java
public UrlMapping getUrl(String shortCode) {
    // 1. Try cache first
    String cached = redisTemplate.opsForValue().get("url:" + shortCode);
    if (cached != null) {
        if ("NOT_FOUND".equals(cached)) {
            return null;  // Cached negative result
        }
        return parseUrlMapping(shortCode, cached);
    }

    // 2. Cache miss: query database
    UrlMapping mapping = urlRepository.findByShortCode(shortCode);
    if (mapping == null) {
        // Cache negative result to prevent repeated DB queries
        redisTemplate.opsForValue().set("url:" + shortCode, "NOT_FOUND", 300, TimeUnit.SECONDS);
        return null;
    }

    // 3. Populate cache
    cacheUrl(shortCode, mapping.getOriginalUrl(), mapping.getRedirectType(), mapping.getExpiresAt());

    return mapping;
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when a cached item expires and many requests simultaneously try to regenerate it, overwhelming the backend (database).

**Scenario:**
```
T+0s:   Cache entry expires for popular URL "abc123"
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database simultaneously
T+1s:   Database overwhelmed, latency spikes
```

**Prevention Strategy: Probabilistic Early Expiration + Locking**

```java
@Service
public class CacheService {
    
    private final RedisTemplate<String, String> redis;
    private final DistributedLock lockService;
    
    private static final double EARLY_EXPIRATION_PROBABILITY = 0.01; // 1%
    
    public UrlMapping getUrl(String shortCode) {
        // 1. Try cache first
        String cached = redisTemplate.opsForValue().get("url:" + shortCode);
        if (cached != null) {
            // Probabilistic early expiration (1% chance)
            if (Math.random() < EARLY_EXPIRATION_PROBABILITY) {
                // Refresh in background (don't block request)
                refreshCacheAsync(shortCode);
            }
            return parseUrlMapping(shortCode, cached);
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:url:" + shortCode;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            // Retry cache (might be populated by now)
            Thread.sleep(50);
            cached = redisTemplate.opsForValue().get("url:" + shortCode);
            if (cached != null) {
                return parseUrlMapping(shortCode, cached);
            }
            // Still miss, fall back to database (acceptable)
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redisTemplate.opsForValue().get("url:" + shortCode);
            if (cached != null) {
                return parseUrlMapping(shortCode, cached);
            }
            
            // 4. Fetch from database (only one thread does this)
            UrlMapping mapping = urlRepository.findByShortCode(shortCode);
            
            if (mapping == null) {
                // Cache negative result
                redisTemplate.opsForValue().set("url:" + shortCode, "NOT_FOUND", 300, TimeUnit.SECONDS);
                return null;
            }
            
            // 5. Populate cache
            cacheUrl(shortCode, mapping.getOriginalUrl(), 
                    mapping.getRedirectType(), mapping.getExpiresAt());
            
            return mapping;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
    
    private void refreshCacheAsync(String shortCode) {
        // Background refresh without blocking
        CompletableFuture.runAsync(() -> {
            UrlMapping mapping = urlRepository.findByShortCode(shortCode);
            if (mapping != null) {
                cacheUrl(shortCode, mapping.getOriginalUrl(),
                        mapping.getRedirectType(), mapping.getExpiresAt());
            }
        });
    }
}
```

**Cache Stampede Simulation:**

```
Scenario: Popular URL cache expires, 10,000 requests arrive

┌─────────────────────────────────────────────────────────────┐
│ T+0ms: Cache entry expires for "abc123"                     │
│ T+0ms: 10,000 requests arrive simultaneously                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Without Protection:                                          │
│ - All 10,000 requests see cache MISS                        │
│ - All 10,000 requests hit database                          │
│ - Database overwhelmed (10,000 concurrent queries)         │
│ - Latency: 5ms → 500ms                                      │
│ - Some requests timeout                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ With Protection (Locking):                                   │
│ - Request 1: Acquires lock, fetches from DB                │
│ - Requests 2-10,000: Lock unavailable, wait 50ms            │
│ - Request 1: Populates cache, releases lock                 │
│ - Requests 2-10,000: Retry cache → HIT                      │
│ - Database: Only 1 query (not 10,000)                      │
│ - Latency: 5ms (cache hit) for 99.99% of requests          │
└─────────────────────────────────────────────────────────────┘
```

**Probabilistic Early Expiration Benefits:**

```
Normal TTL: 24 hours
Early expiration: 1% chance per request

Benefits:
- Cache refreshed before expiration
- No stampede (only 1% of requests trigger refresh)
- Background refresh doesn't block requests
- Cache stays warm
```

### Redis Cluster Configuration

```yaml
# Redis Cluster with 6 nodes (3 primary + 3 replica)
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:6379
        - redis-node-2:6379
        - redis-node-3:6379
        - redis-node-4:6379
        - redis-node-5:6379
        - redis-node-6:6379
      max-redirects: 3
    lettuce:
      pool:
        max-active: 100
        max-idle: 50
        min-idle: 10
```

---

## 3. Search (Elasticsearch / OpenSearch)

### Not Applicable for URL Shortener

Search is **not required** for the URL shortener because:

1. **Primary access pattern is key-value lookup**: Given a short code, return the original URL
2. **No full-text search requirement**: Users don't search for URLs by content
3. **No fuzzy matching needed**: Short codes are exact matches
4. **Analytics queries use SQL**: Aggregations are handled by PostgreSQL/ClickHouse

**What would change if search were added later?**

If we needed to support searching URLs by:
- Original URL content
- Metadata/tags
- Custom alias prefix

We would add:
1. Elasticsearch cluster for indexing URL metadata
2. Indexer service to sync from PostgreSQL to Elasticsearch
3. Search API endpoint with query parsing

But for the core URL shortener use case, this is unnecessary complexity.

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Async Messaging | Kafka | 12 partitions, RF=3, 7-day retention |
| Caching | Redis Cluster | 16GB, LRU eviction, 24h TTL |
| Search | Not applicable | Key-value lookups only |

