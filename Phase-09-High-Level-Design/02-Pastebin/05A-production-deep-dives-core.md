# Pastebin - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (if applicable), caching (Redis), and search capabilities. These are the fundamental building blocks that enable Pastebin to meet its latency and storage requirements.

---

## 1. Async, Messaging & Event Flow

### Not Applicable for Core Pastebin

Asynchronous messaging is **not required** for the core Pastebin functionality because:

1. **Synchronous operations suffice**: Paste creation and retrieval are simple CRUD operations
2. **No event-driven workflows**: Unlike URL Shortener with click analytics, Pastebin doesn't need real-time event processing
3. **Low write volume**: 4 QPS for writes doesn't justify Kafka overhead
4. **No downstream consumers**: No separate analytics pipeline needed

**Why synchronous works here:**

```
Paste Creation Flow (Synchronous):
Client → API → Validate → S3 Upload → DB Insert → Cache → Response
Total: ~200-500ms (acceptable for upload operation)

Paste Retrieval Flow (Synchronous):
Client → CDN → (miss) → Cache → (miss) → DB + S3 → Response
Total: ~50-100ms (acceptable for view operation)
```

### What Would Change If Async Were Introduced?

**Scenario 1: Adding View Analytics**

If we needed detailed view analytics (like URL Shortener), we would add:

```
View Event Flow:
User views paste → Paste Service → Kafka (view-events topic)
                                        ↓
                              Analytics Consumer
                                        ↓
                              ClickHouse (analytics)
```

**Scenario 2: Content Processing Pipeline**

If we needed async content processing (malware scanning, syntax detection):

```
Content Processing Flow:
Paste created → DB (status: processing) → Kafka (process-events)
                                              ↓
                                    Content Processor Consumer
                                              ↓
                                    Update DB (status: ready)
```

**Scenario 3: Notification System**

If we needed to notify users of paste views/expirations:

```
Notification Flow:
Paste expires → Cleanup Worker → Kafka (notification-events)
                                        ↓
                                Notification Consumer
                                        ↓
                                Email/Push Service
```

### Current Architecture Decision

For the current requirements (10M pastes/month, 100M views/month), synchronous processing is simpler and sufficient:

| Aspect | Sync (Current) | Async (If Added) |
|--------|----------------|------------------|
| Complexity | Low | High |
| Latency | Acceptable | Lower for writes |
| Infrastructure | Less | Kafka cluster needed |
| Failure modes | Simpler | More complex |
| Cost | Lower | Higher |

**Recommendation**: Start synchronous, add Kafka only when:
- View analytics become a requirement
- Content processing pipeline needed
- Scale exceeds 100M views/month

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis and Caching Patterns?

Redis is an in-memory data structure store that provides sub-millisecond latency for read/write operations. For Pastebin, caching is important because:

- S3 latency is 50-100ms per request
- Popular pastes would overwhelm S3 request limits
- CDN cache misses need fast origin response

**Caching Patterns:**

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **Cache-Aside** | App checks cache, then storage on miss | Read-heavy, can tolerate stale data |
| **Write-Through** | Write to cache and storage together | Need immediate cache consistency |
| **Write-Back** | Write to cache, async to storage | Write-heavy, can lose data |

**We use a hybrid approach:**
- **Write-Through** for metadata (cache immediately on create)
- **Cache-Aside** for content (cache on read, not on write)

**Consistency Tradeoff:**

We accept eventual consistency for cached content:
- If a paste is deleted, cached content may be served for up to 15 minutes
- This is acceptable because deletions are rare
- For immediate invalidation, we explicitly delete from cache

### B) OUR USAGE: How We Use Redis Here

**Exact Keys and Values:**

```
Metadata Cache:
Key:    meta:{paste_id}
Value:  {"id":"abc123","storage_key":"pastes/2024/01/15/abc123.txt.gz","syntax":"python","visibility":"unlisted","expires_at":1735689600}
TTL:    3600 seconds (1 hour)

Content Cache (small pastes only):
Key:    content:{paste_id}
Value:  [gzip compressed content bytes]
TTL:    900 seconds (15 minutes)
Max:    1 MB (larger pastes skip Redis)

Rate Limiting:
Key:    rate:{ip}:{action}:{window}
Value:  count (integer)
TTL:    3600 seconds (1 hour)

User Paste List:
Key:    user:{user_id}:pastes
Value:  List of paste IDs (most recent first)
TTL:    300 seconds (5 minutes)
```

**Who reads Redis?**
- Paste Service (on every view request)
- API Gateway (for rate limiting)

**Who writes Redis?**
- Paste Service (on paste creation for metadata)
- Paste Service (on cache miss for content)
- Rate Limiter (on every request)

**TTL Behavior:**

| Cache Type | TTL | Rationale |
|------------|-----|-----------|
| Metadata | 1 hour | Rarely changes, high hit rate needed |
| Content | 15 minutes | Memory-intensive, balance hit rate vs memory |
| Rate limits | 1 hour | Matches rate limit window |
| User lists | 5 minutes | Frequently updated, short cache |

**Why 15 minutes for content?**
- Content is large (avg 100KB)
- Memory is expensive
- 15 min provides good hit rate for viral pastes
- Cold pastes evicted quickly

**Eviction Policy:**

```
maxmemory 16gb
maxmemory-policy allkeys-lru
```

**Why LRU (Least Recently Used)?**
- Automatically evicts cold pastes
- Keeps hot pastes in cache
- No manual management needed

**Invalidation Strategy:**

On paste deletion:
1. Soft delete in database
2. Delete from Redis: `DEL meta:{paste_id}` and `DEL content:{paste_id}`
3. Purge from CDN (async)

On paste update (if allowed):
1. Update in database
2. Delete from Redis (will be repopulated on next read)
3. Purge from CDN

**Why Redis vs Local Cache vs S3 Direct?**

| Option | Latency | Capacity | Consistency |
|--------|---------|----------|-------------|
| Local Cache (Caffeine) | ~0.1ms | Limited by JVM heap | Per-instance, inconsistent |
| Redis | ~1ms | 16GB+ | Shared, consistent |
| S3 Direct | ~50-100ms | Unlimited | Always consistent |

We use Redis as primary cache because:
- Shared across all app instances
- Large capacity (16GB)
- Sub-millisecond latency
- Handles large values (up to 512MB)

We also use local cache (Caffeine) for the hottest 100 pastes metadata to reduce Redis round-trips.

### C) REAL STEP-BY-STEP SIMULATION

**Cache HIT Path (Metadata + Content):**

```
Request: GET /v1/pastes/abc123
    ↓
Paste Service receives request
    ↓
Redis: GET meta:abc123
    ↓
Redis returns: {"id":"abc123","storage_key":"...","syntax":"python",...}
    ↓
Redis: GET content:abc123
    ↓
Redis returns: [compressed content bytes]
    ↓
Decompress and return 200 OK with content
    ↓
Total latency: ~5-10ms
```

**Cache MISS Path (Content):**

```
Request: GET /v1/pastes/xyz789
    ↓
Paste Service receives request
    ↓
Redis: GET meta:xyz789
    ↓
Redis returns: {"id":"xyz789","storage_key":"pastes/2024/01/15/xyz789.txt.gz",...}
    ↓
Redis: GET content:xyz789
    ↓
Redis returns: (nil) - MISS
    ↓
S3: GetObject pastes/2024/01/15/xyz789.txt.gz
    ↓
S3 returns: [compressed content bytes]
    ↓
Check size: 50KB < 1MB, cache it
    ↓
Redis: SET content:xyz789 [bytes] EX 900
    ↓
Decompress and return 200 OK
    ↓
Total latency: ~60-100ms
```

**Cache MISS Path (Metadata + Content):**

```
Request: GET /v1/pastes/def456
    ↓
Paste Service receives request
    ↓
Redis: GET meta:def456
    ↓
Redis returns: (nil) - MISS
    ↓
PostgreSQL: SELECT * FROM pastes WHERE id = 'def456'
    ↓
DB returns: {id, storage_key, syntax, visibility, ...}
    ↓
Redis: SET meta:def456 [json] EX 3600
    ↓
S3: GetObject [storage_key]
    ↓
S3 returns: [compressed content bytes]
    ↓
Redis: SET content:def456 [bytes] EX 900
    ↓
Decompress and return 200 OK
    ↓
Total latency: ~100-150ms
```

**Negative Caching (NOT_FOUND):**

```
Request: GET /v1/pastes/invalid123
    ↓
Redis: GET meta:invalid123 → (nil)
    ↓
PostgreSQL: SELECT ... WHERE id = 'invalid123' → no rows
    ↓
Redis: SET meta:invalid123 "NOT_FOUND" EX 300  (5 minutes)
    ↓
Return 404 Not Found
```

Why cache NOT_FOUND?
- Prevents repeated DB queries for invalid IDs
- Short TTL (5 min) in case paste is created later
- Protects against enumeration attacks

**Failure Scenarios:**

**Q: What if Redis is down?**

A: Circuit breaker activates:

```java
@CircuitBreaker(name = "redis", fallbackMethod = "getFromStorage")
public PasteContent getFromCache(String pasteId) {
    byte[] content = redisTemplate.opsForValue().get("content:" + pasteId);
    return content != null ? decompress(content) : null;
}

public PasteContent getFromStorage(String pasteId, Exception e) {
    log.warn("Redis unavailable, falling back to S3", e);
    return s3Service.getContent(pasteId);
}
```

Impact:
- All 200 QPS hit S3 directly
- S3 can handle 5,500 GET/sec per prefix
- Latency increases from 10ms to 100ms
- System degrades but doesn't fail

**Q: What happens to S3 load and latency?**

A: Without cache:
- S3 load increases 10x (from ~20 QPS to 200 QPS)
- Latency increases from 10ms to 100ms
- May hit S3 request rate limits if traffic spikes
- Cost increases due to more S3 requests

**Q: What is the circuit breaker behavior?**

A: Using Resilience4j:

```yaml
resilience4j.circuitbreaker:
  instances:
    redis:
      slidingWindowSize: 100
      failureRateThreshold: 50  # Open after 50% failures
      waitDurationInOpenState: 30s
      permittedNumberOfCallsInHalfOpenState: 10
```

**Q: Why this TTL (15 minutes for content)?**

A: Balance between:
- Cache hit rate (longer TTL = higher hit rate)
- Memory usage (longer TTL = more memory)
- Content is large (100KB average)

15 minutes provides ~80% cache hit rate with acceptable memory usage.

**Q: Why write-through vs write-back vs cache-aside?**

A: We use:
- **Write-through for metadata**: Paste must be immediately findable after creation
- **Cache-aside for content**: Content is large, only cache on read to avoid wasting memory on pastes that are never viewed

**Q: Negative caching and when to use it?**

A: We cache NOT_FOUND results for:
- Invalid paste IDs (prevents enumeration attacks)
- Expired pastes (prevents repeated DB lookups)
- Deleted pastes (prevents serving stale content)

Short TTL (5 minutes) ensures:
- New pastes become accessible quickly
- Memory isn't wasted on negative entries

### Cache Operations Code

**On Paste Creation (Write-Through for Metadata):**

```java
@Service
public class PasteCacheService {
    
    private final RedisTemplate<String, String> metaRedis;
    private final RedisTemplate<String, byte[]> contentRedis;
    
    public void cacheMetadata(Paste paste) {
        String key = "meta:" + paste.getId();
        String value = objectMapper.writeValueAsString(PasteMetadata.from(paste));
        
        // Calculate TTL: minimum of 1 hour or time until expiration
        long ttlSeconds = 3600;
        if (paste.getExpiresAt() != null) {
            long secondsUntilExpiry = ChronoUnit.SECONDS.between(
                Instant.now(), paste.getExpiresAt());
            ttlSeconds = Math.min(ttlSeconds, secondsUntilExpiry);
        }
        
        metaRedis.opsForValue().set(key, value, ttlSeconds, TimeUnit.SECONDS);
    }
    
    // Note: Content is NOT cached on create (cache-aside pattern)
}
```

**On Paste View (Cache-Aside for Content):**

```java
public PasteContent getContent(String pasteId, String storageKey) {
    String contentKey = "content:" + pasteId;
    
    // 1. Try cache first
    byte[] cached = contentRedis.opsForValue().get(contentKey);
    if (cached != null) {
        metrics.recordCacheHit("content");
        return decompress(cached);
    }
    
    metrics.recordCacheMiss("content");
    
    // 2. Cache miss: fetch from S3
    byte[] compressed = s3Service.getObject(storageKey);
    
    // 3. Only cache if small enough (< 1MB)
    if (compressed.length < 1024 * 1024) {
        contentRedis.opsForValue().set(contentKey, compressed, 
            Duration.ofMinutes(15));
    }
    
    return decompress(compressed);
}
```

**On Paste Deletion (Invalidation):**

```java
public void invalidateCache(String pasteId) {
    // Delete both metadata and content
    metaRedis.delete("meta:" + pasteId);
    contentRedis.delete("content:" + pasteId);
    
    // Async CDN invalidation
    cdnService.invalidateAsync("/raw/" + pasteId);
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede occurs when a cached paste expires and many requests simultaneously try to fetch it from S3, overwhelming the storage service.

**Scenario:**
```
T+0s:   Popular paste cache expires (viral paste)
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit S3 simultaneously
T+1s:   S3 overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Probabilistic Early Expiration**

```java
@Service
public class PasteContentService {
    
    private final RedisTemplate<String, byte[]> redis;
    private final DistributedLock lockService;
    private final S3Service s3Service;
    
    private static final double EARLY_EXPIRATION_PROBABILITY = 0.01; // 1%
    
    public PasteContent getContent(String pasteId, String storageKey) {
        String contentKey = "content:" + pasteId;
        
        // 1. Try cache first
        byte[] cached = redis.opsForValue().get(contentKey);
        if (cached != null) {
            // Probabilistic early expiration (1% chance)
            if (Math.random() < EARLY_EXPIRATION_PROBABILITY) {
                // Refresh in background (don't block request)
                refreshCacheAsync(pasteId, storageKey);
            }
            return decompress(cached);
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:content:" + pasteId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(contentKey);
                if (cached != null) {
                    return decompress(cached);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to S3 (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache
            cached = redis.opsForValue().get(contentKey);
            if (cached != null) {
                return decompress(cached);
            }
            
            // 4. Fetch from S3 (only one thread does this)
            byte[] compressed = s3Service.getObject(storageKey);
            
            // 5. Only cache if small enough (< 1MB)
            if (compressed.length < 1024 * 1024) {
                redis.opsForValue().set(contentKey, compressed, 
                    Duration.ofMinutes(15));
            }
            
            return decompress(compressed);
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
    
    private void refreshCacheAsync(String pasteId, String storageKey) {
        // Background refresh without blocking
        CompletableFuture.runAsync(() -> {
            byte[] compressed = s3Service.getObject(storageKey);
            if (compressed.length < 1024 * 1024) {
                redis.opsForValue().set("content:" + pasteId, compressed, 
                    Duration.ofMinutes(15));
            }
        });
    }
}
```

**Cache Stampede Simulation:**

```
Scenario: Viral paste cache expires, 10,000 requests arrive

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Cache entry expires for "abc12345                      │
│ T+0s: 10,000 requests arrive simultaneously                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Without Protection:                                          │
│ - All 10,000 requests see cache MISS                        │
│ - All 10,000 requests hit S3                               │
│ - S3 overwhelmed (10,000 concurrent GET requests)        │
│ - Latency: 50ms → 2 seconds                                │
│ - Some requests timeout                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ With Protection (Locking):                                  │
│ - Request 1: Acquires lock, fetches from S3                │
│ - Requests 2-10,000: Lock unavailable, wait 50ms          │
│ - Request 1: Populates cache, releases lock                 │
│ - Requests 2-10,000: Retry cache → HIT                      │
│ - S3: Only 1 request (not 10,000)                          │
│ - Latency: 5ms (cache hit) for 99.99% of requests          │
└─────────────────────────────────────────────────────────────┘
```

### Redis Cluster Configuration

```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:6379
        - redis-node-2:6379
        - redis-node-3:6379
      max-redirects: 3
    lettuce:
      pool:
        max-active: 50
        max-idle: 25
        min-idle: 5
```

### Cache Decision Logic

```java
public class CacheStrategy {
    
    private static final long MAX_REDIS_SIZE = 1024 * 1024; // 1 MB
    
    public CacheDecision decide(Paste paste, byte[] content) {
        // Private pastes: no CDN caching, shorter Redis TTL
        if (paste.getVisibility() == Visibility.PRIVATE) {
            return CacheDecision.builder()
                .cdnCacheable(false)
                .redisCacheable(content.length < MAX_REDIS_SIZE)
                .redisTtl(Duration.ofMinutes(5))
                .build();
        }
        
        // Large pastes: skip Redis, only CDN
        if (content.length > MAX_REDIS_SIZE) {
            return CacheDecision.builder()
                .cdnCacheable(true)
                .cdnTtl(Duration.ofHours(1))
                .redisCacheable(false)
                .build();
        }
        
        // Normal pastes: cache everywhere
        return CacheDecision.builder()
            .cdnCacheable(true)
            .cdnTtl(Duration.ofHours(1))
            .redisCacheable(true)
            .redisTtl(Duration.ofMinutes(15))
            .build();
    }
}
```

---

## 3. Database (PostgreSQL)

### A) CONCEPT: Why PostgreSQL for Pastebin?

PostgreSQL is used for paste metadata storage because:
- **ACID guarantees**: Ensures paste creation/deletion is atomic
- **Relational queries**: Fast lookups by ID, user, expiration
- **Indexing**: Efficient queries with proper indexes
- **Transactions**: Ensures consistency for paste operations

### B) OUR USAGE: How We Use PostgreSQL

**Schema:**
- `pastes` table: Stores paste metadata (id, storage_key, syntax, visibility, etc.)
- `users` table: User accounts (if authentication added)
- Indexes: Primary key on `id`, index on `user_id`, index on `expires_at`

**Operations:**
- **Create**: INSERT into pastes table
- **Read**: SELECT by id (primary key lookup)
- **Update**: UPDATE paste metadata
- **Delete**: Soft delete (set deleted_at timestamp)

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Create Paste**

```
Request: POST /v1/pastes
    ↓
Paste Service receives request
    ↓
Validate input (content size, syntax, etc.)
    ↓
PostgreSQL: BEGIN TRANSACTION
    ↓
PostgreSQL: INSERT INTO pastes (id, storage_key, content_hash, ...) 
            VALUES ('abc123', 'pastes/2024/01/15/abc123.txt.gz', 'sha256...', ...)
    ↓
PostgreSQL: COMMIT TRANSACTION
    ↓
S3: Upload content to storage_key
    ↓
Redis: SET meta:abc123 [metadata] EX 3600
    ↓
Return 201 Created with paste metadata
    ↓
Total latency: ~200-500ms (mostly S3 upload)
```

**Normal Flow: Read Paste**

```
Request: GET /v1/pastes/abc123
    ↓
Paste Service receives request
    ↓
Redis: GET meta:abc123 → (nil) MISS
    ↓
PostgreSQL: SELECT * FROM pastes WHERE id = 'abc123' AND deleted_at IS NULL
    ↓
PostgreSQL returns: {id: 'abc123', storage_key: '...', syntax: 'python', ...}
    ↓
Redis: SET meta:abc123 [metadata] EX 3600
    ↓
S3: GetObject [storage_key]
    ↓
Return 200 OK with paste content
    ↓
Total latency: ~100-150ms
```

**Failure Flow: Database Connection Timeout**

```
Request: GET /v1/pastes/abc123
    ↓
Paste Service receives request
    ↓
Redis: GET meta:abc123 → (nil) MISS
    ↓
PostgreSQL: SELECT * FROM pastes WHERE id = 'abc123'
    ↓
PostgreSQL connection timeout after 5 seconds
    ↓
Circuit breaker opens (after 50% failure rate)
    ↓
Fallback: Return 503 Service Unavailable
    ↓
Retry with exponential backoff: 1s, 2s, 4s
    ↓
If DB recovers: Circuit breaker half-open, allow 10 test requests
    ↓
If successful: Circuit breaker closes, normal operation resumes
```

**Duplicate/Idempotency Handling:**

```
Request: POST /v1/pastes (with Idempotency-Key: key_xyz)
    ↓
Check Redis for idempotency key: GET idempotency:key_xyz
    ↓
If found: Return cached response (200 OK with existing paste)
    ↓
If not found:
    ↓
PostgreSQL: BEGIN TRANSACTION
    ↓
PostgreSQL: SELECT id FROM pastes WHERE idempotency_key = 'key_xyz' FOR UPDATE
    ↓
If row exists: Return existing paste (409 Conflict or 200 OK)
    ↓
If no row: INSERT INTO pastes (id, idempotency_key, ...) VALUES (...)
    ↓
PostgreSQL: COMMIT TRANSACTION
    ↓
Cache response in Redis with idempotency key
    ↓
Return 201 Created
```

**Traffic Spike Handling: Hot Key (Popular Paste)**

```
Scenario: Celebrity shares paste link, 10,000 requests/second
    ↓
All requests: GET /v1/pastes/abc123
    ↓
First request: Redis MISS → PostgreSQL SELECT → Cache result
    ↓
Subsequent 9,999 requests: Redis HIT (sub-millisecond)
    ↓
PostgreSQL load: 1 query (not 10,000)
    ↓
Redis handles 10,000 QPS easily (single key)
    ↓
No database overload
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
RPO: 0 (synchronous replication)
```

## 4. Search (Elasticsearch / OpenSearch)

### Not Applicable for Core Pastebin

Search is **not required** for the core Pastebin functionality because:

1. **Primary access pattern is key-value lookup**: Given a paste ID, return the content
2. **No full-text search requirement**: Users access pastes via direct URL
3. **Public paste listing is simple**: Recent public pastes sorted by time (PostgreSQL handles this)
4. **No fuzzy matching needed**: Paste IDs are exact matches

**What would change if search were added later?**

If we needed to support searching pastes by:
- Content keywords
- Code snippets
- Syntax/language filters
- User-defined tags

We would add:

### Hypothetical Search Architecture

**A) CONCEPT: What is Elasticsearch?**

Elasticsearch is a distributed search engine built on Apache Lucene. It uses inverted indexes to enable fast full-text search across large datasets.

**Inverted Index Concept:**

```
Document: "def hello(): print('world')"

Inverted Index:
  "def"    → [paste_id_1, paste_id_5, paste_id_9]
  "hello"  → [paste_id_1, paste_id_3]
  "print"  → [paste_id_1, paste_id_2, paste_id_7]
  "world"  → [paste_id_1, paste_id_4]
```

**B) HYPOTHETICAL USAGE:**

```
Index: pastes
Mapping:
{
  "properties": {
    "id": { "type": "keyword" },
    "title": { "type": "text", "analyzer": "standard" },
    "content": { "type": "text", "analyzer": "code_analyzer" },
    "syntax": { "type": "keyword" },
    "created_at": { "type": "date" },
    "visibility": { "type": "keyword" }
  }
}
```

**Indexing Flow:**

```
Paste created → DB Insert → Kafka (paste-created event)
                                   ↓
                          Search Indexer Consumer
                                   ↓
                          Elasticsearch Index
```

**Query Flow:**

```
User search "def hello python"
    ↓
Search API: GET /v1/search?q=def+hello&syntax=python
    ↓
Elasticsearch query:
{
  "query": {
    "bool": {
      "must": [
        { "match": { "content": "def hello" } }
      ],
      "filter": [
        { "term": { "syntax": "python" } },
        { "term": { "visibility": "public" } }
      ]
    }
  }
}
    ↓
Return matching paste IDs
    ↓
Fetch paste metadata from PostgreSQL
    ↓
Return search results
```

**C) HYPOTHETICAL FAILURE SCENARIOS:**

- **Stale index**: Paste deleted but still in search results
  - Solution: Soft delete in DB, filter by deleted_at in search
- **Indexer lag**: New pastes not searchable immediately
  - Solution: Accept eventual consistency (search is not critical path)
- **Reindexing**: Schema change requires full reindex
  - Solution: Blue-green index deployment

### Current Architecture Decision

For the current requirements, search is not needed:

| Feature | Without Search | With Search |
|---------|----------------|-------------|
| Access by ID | ✓ PostgreSQL | ✓ PostgreSQL |
| Recent public | ✓ PostgreSQL | ✓ PostgreSQL |
| Full-text search | ✗ | ✓ Elasticsearch |
| Code search | ✗ | ✓ Elasticsearch |
| Complexity | Low | High |
| Cost | Lower | Higher |

**Recommendation**: Add Elasticsearch only when:
- Full-text search becomes a product requirement
- Code search/discovery feature requested
- Scale exceeds PostgreSQL's LIKE query performance

---

## Summary

| Component | Technology | Status | Key Configuration |
|-----------|------------|--------|-------------------|
| Async Messaging | N/A | Not needed | Synchronous suffices |
| Caching | Redis Cluster | Active | 16GB, LRU, 15-min content TTL |
| Search | N/A | Not needed | Key-value lookups only |

### Why This Architecture?

1. **Simplicity**: No Kafka or Elasticsearch overhead for current scale
2. **Cost-effective**: Fewer managed services to maintain
3. **Sufficient performance**: 200 QPS easily handled by PostgreSQL + Redis + S3
4. **Extensible**: Can add Kafka/ES when requirements demand

### When to Evolve?

| Trigger | Action |
|---------|--------|
| View analytics needed | Add Kafka + ClickHouse |
| Full-text search needed | Add Elasticsearch |
| > 1000 QPS | Scale Redis, add more app pods |
| > 10000 QPS | Consider sharding PostgreSQL |

