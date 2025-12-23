# 🏠 Local vs Distributed Cache

---

## 0️⃣ Prerequisites

Before diving into local vs distributed caching, you need to understand:

- **Cache**: A fast storage layer for frequently accessed data. Covered in Topics 1-5.
- **JVM Heap**: The memory area where Java objects live. Limited by `-Xmx` setting.
- **Network Latency**: Time for data to travel between machines. Even within a data center, this is ~0.5-2ms.
- **Serialization**: Converting objects to bytes for transmission. Required for distributed caches.

If you understand that accessing local memory is faster than network calls, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

You're building a product catalog service. You have two options:

**Option A: Local Cache (HashMap in JVM)**
```java
private Map<Long, Product> cache = new ConcurrentHashMap<>();
```

**Option B: Distributed Cache (Redis)**
```java
redisTemplate.opsForValue().get("product:" + id);
```

Which should you choose? The answer is: **it depends**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE TRADEOFF                                          │
│                                                                          │
│   LOCAL CACHE                          DISTRIBUTED CACHE                 │
│   ┌─────────────────────┐              ┌─────────────────────┐          │
│   │  ⚡ 50 nanoseconds  │              │  🌐 0.5-2 milliseconds│         │
│   │  Access time        │              │  Access time         │          │
│   │                     │              │                      │          │
│   │  ❌ Not shared      │              │  ✅ Shared across    │          │
│   │  across servers     │              │  all servers         │          │
│   │                     │              │                      │          │
│   │  ❌ Lost on restart │              │  ✅ Survives restarts│          │
│   │                     │              │                      │          │
│   │  ❌ Limited to JVM  │              │  ✅ Scales to TBs    │          │
│   │  heap size          │              │                      │          │
│   └─────────────────────┘              └─────────────────────┘          │
│                                                                          │
│   Local is 10,000-40,000x faster but has significant limitations        │
└─────────────────────────────────────────────────────────────────────────┘
```

### What Breaks with Wrong Choice?

**Using Only Local Cache**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOCAL CACHE PROBLEM                                   │
│                                                                          │
│   Server 1                    Server 2                                   │
│   ┌──────────────────┐       ┌──────────────────┐                       │
│   │ Cache:           │       │ Cache:           │                       │
│   │ product:123 =    │       │ product:123 =    │                       │
│   │ {price: $99}     │       │ {price: $79}     │  ← Different values!  │
│   └──────────────────┘       └──────────────────┘                       │
│                                                                          │
│   User A sees $99            User B sees $79                            │
│   (Server 1)                 (Server 2)                                 │
│                                                                          │
│   Problems:                                                              │
│   1. Inconsistent data across servers                                   │
│   2. Cache wasted - same data cached N times on N servers              │
│   3. Cold start after deployment - all caches empty                    │
└─────────────────────────────────────────────────────────────────────────┘
```

**Using Only Distributed Cache**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED CACHE PROBLEM                             │
│                                                                          │
│   Server 1     Server 2     Server 3                                    │
│      │            │            │                                        │
│      │  1ms       │  1ms       │  1ms                                   │
│      └────────────┼────────────┘                                        │
│                   │                                                      │
│                   ▼                                                      │
│            ┌──────────────┐                                             │
│            │    Redis     │                                             │
│            └──────────────┘                                             │
│                                                                          │
│   10,000 requests/second × 3 servers = 30,000 Redis calls/second       │
│                                                                          │
│   Problems:                                                              │
│   1. Network latency on every access (1ms vs 50ns)                     │
│   2. Redis becomes bottleneck                                           │
│   3. Network bandwidth consumed                                         │
│   4. Single point of failure                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### Real Examples

**Netflix**: Uses a two-level cache. L1 (local, Guava/Caffeine) for extremely hot data like configuration. L2 (EVCache/Memcached) for shared data like user profiles.

**Facebook**: Uses local caches in front of Memcached for the hottest keys. A single popular user's profile might be accessed millions of times per second.

**Uber**: Uses local caches for surge pricing calculations (needs sub-millisecond latency) and distributed cache for driver/rider matching data.

---

## 2️⃣ Intuition and Mental Model

### The Office Supplies Analogy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE OFFICE SUPPLIES ANALOGY                           │
│                                                                          │
│   LOCAL CACHE = Your Desk Drawer                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   ✅ Instant access (reach into drawer)                         │   │
│   │   ✅ No one else can take your stuff                            │   │
│   │   ❌ Limited space (one small drawer)                           │   │
│   │   ❌ Coworker can't borrow your stapler                         │   │
│   │   ❌ If you change desks, drawer is empty                       │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   DISTRIBUTED CACHE = Supply Closet Down the Hall                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   ✅ Everyone can access it                                      │   │
│   │   ✅ Huge storage capacity                                       │   │
│   │   ✅ Survives if you change desks                               │   │
│   │   ❌ Takes 30 seconds to walk there                             │   │
│   │   ❌ Might be crowded (contention)                              │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   HYBRID = Keep frequently used items in drawer,                        │
│            get less common items from closet                            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Local Cache Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOCAL CACHE (Caffeine)                                │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                        JVM HEAP                                  │   │
│   │                                                                  │   │
│   │   ┌───────────────────────────────────────────────────────┐     │   │
│   │   │                   CAFFEINE CACHE                       │     │   │
│   │   │                                                        │     │   │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │     │   │
│   │   │   │   Window    │  │  Probation  │  │  Protected  │  │     │   │
│   │   │   │   (1%)      │  │   (20%)     │  │   (79%)     │  │     │   │
│   │   │   │             │  │             │  │             │  │     │   │
│   │   │   │  New items  │  │  Promotion  │  │ Frequently  │  │     │   │
│   │   │   │  enter here │──▶│  candidates │──▶│ accessed    │  │     │   │
│   │   │   │             │  │             │  │             │  │     │   │
│   │   │   └─────────────┘  └─────────────┘  └─────────────┘  │     │   │
│   │   │                                                        │     │   │
│   │   │   Uses W-TinyLFU algorithm (combines LRU + LFU)       │     │   │
│   │   │   Near-optimal hit rate                                │     │   │
│   │   └───────────────────────────────────────────────────────┘     │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Access Time: ~50 nanoseconds (direct memory access)                   │
│   No serialization needed                                                │
│   No network call                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Distributed Cache Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED CACHE (Redis)                             │
│                                                                          │
│   Application Server                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. Serialize object to JSON/bytes                               │   │
│   │  2. Send over TCP to Redis                                       │   │
│   │  3. Wait for response                                            │   │
│   │  4. Deserialize response                                         │   │
│   └───────────────────────────────┬─────────────────────────────────┘   │
│                                   │                                      │
│                                   │ TCP (1-2ms round trip)              │
│                                   │                                      │
│                                   ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         REDIS SERVER                             │   │
│   │                                                                  │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                    MEMORY                                │   │   │
│   │   │                                                          │   │   │
│   │   │   Hash Table with data                                   │   │   │
│   │   │   "product:123" → "{name: 'Keyboard', price: 99.99}"   │   │   │
│   │   │                                                          │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Access Time: ~500-2000 microseconds (network + serialization)         │
│   Shared across all application servers                                  │
│   Survives application restarts                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Hybrid (Multi-Level) Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MULTI-LEVEL CACHE (L1 + L2)                           │
│                                                                          │
│   Request Flow:                                                          │
│                                                                          │
│   1. Check L1 (local)                                                    │
│      │                                                                   │
│      ├── HIT → Return immediately (50ns)                                │
│      │                                                                   │
│      └── MISS → Check L2 (distributed)                                  │
│                 │                                                        │
│                 ├── HIT → Store in L1, return (1-2ms)                   │
│                 │                                                        │
│                 └── MISS → Query database                               │
│                            │                                             │
│                            └── Store in L2, store in L1, return         │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Server 1          Server 2          Server 3                    │   │
│   │  ┌──────────┐     ┌──────────┐     ┌──────────┐                 │   │
│   │  │ L1 Cache │     │ L1 Cache │     │ L1 Cache │                 │   │
│   │  │ (local)  │     │ (local)  │     │ (local)  │                 │   │
│   │  │ 1000     │     │ 1000     │     │ 1000     │                 │   │
│   │  │ items    │     │ items    │     │ items    │                 │   │
│   │  └────┬─────┘     └────┬─────┘     └────┬─────┘                 │   │
│   │       │                │                │                        │   │
│   │       └────────────────┼────────────────┘                        │   │
│   │                        │                                         │   │
│   │                        ▼                                         │   │
│   │            ┌───────────────────────┐                            │   │
│   │            │      L2 Cache         │                            │   │
│   │            │      (Redis)          │                            │   │
│   │            │   1,000,000 items     │                            │   │
│   │            └───────────┬───────────┘                            │   │
│   │                        │                                         │   │
│   │                        ▼                                         │   │
│   │            ┌───────────────────────┐                            │   │
│   │            │      Database         │                            │   │
│   │            │  100,000,000 items    │                            │   │
│   │            └───────────────────────┘                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Scenario: Product Catalog with 10,000 requests/second

**Setup**:
- 3 application servers
- 1 million products
- 80% of requests hit 1% of products (hot items)

### Local Only

```
Request for product:123 on Server 1:
─────────────────────────────────────────────────────────────────────────
Time: 0ns     Check local cache
Time: 50ns    Cache HIT! Return product
─────────────────────────────────────────────────────────────────────────
Total: 50 nanoseconds

Same request on Server 2 (first time):
─────────────────────────────────────────────────────────────────────────
Time: 0ns     Check local cache
Time: 50ns    Cache MISS
Time: 50ns    Query database
Time: 50ms    Database returns
Time: 50ms    Store in local cache
Time: 50ms    Return product
─────────────────────────────────────────────────────────────────────────
Total: 50 milliseconds (1000x slower for first access on each server!)

Problem: 3 servers × 10,000 products = potential 30,000 database queries
         instead of 10,000
```

### Distributed Only

```
Request for product:123 on any server:
─────────────────────────────────────────────────────────────────────────
Time: 0ms       Serialize request
Time: 0.1ms     Send to Redis
Time: 0.5ms     Network latency
Time: 0.6ms     Redis processes
Time: 0.7ms     Network latency (return)
Time: 0.8ms     Deserialize response
─────────────────────────────────────────────────────────────────────────
Total: ~1 millisecond

10,000 requests/second × 1ms = 10 seconds of cumulative latency/second
Redis handling 10,000 requests/second (approaching limits)
```

### Hybrid (L1 + L2)

```
Request for HOT product:123:
─────────────────────────────────────────────────────────────────────────
Time: 0ns     Check L1 (local cache)
Time: 50ns    L1 HIT! Return immediately
─────────────────────────────────────────────────────────────────────────
Total: 50 nanoseconds

Request for COLD product:999999 (first time on this server):
─────────────────────────────────────────────────────────────────────────
Time: 0ns       Check L1 (local cache) - MISS
Time: 0.1ms     Check L2 (Redis)
Time: 1ms       L2 HIT! (another server cached it earlier)
Time: 1ms       Store in L1
Time: 1ms       Return product
─────────────────────────────────────────────────────────────────────────
Total: ~1 millisecond (but subsequent requests: 50ns)

Result:
- 80% of requests (hot items): 50ns (L1 hit)
- 19% of requests (warm items): 1ms (L2 hit)
- 1% of requests (cold items): 50ms (database)

Average latency: 0.8 × 0.00005ms + 0.19 × 1ms + 0.01 × 50ms = 0.69ms
vs Local only: Higher database load
vs Distributed only: 1ms average
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Netflix's Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NETFLIX CACHING LAYERS                                │
│                                                                          │
│   Layer 0: In-Process Cache (Guava/Caffeine)                            │
│   ─────────────────────────────────────────────                         │
│   - Config values, feature flags                                         │
│   - TTL: Minutes                                                         │
│   - Size: ~100MB per instance                                           │
│                                                                          │
│   Layer 1: EVCache (Memcached-based)                                    │
│   ─────────────────────────────────────────────                         │
│   - User profiles, viewing history                                       │
│   - TTL: Hours                                                           │
│   - Size: Terabytes                                                      │
│   - Cross-region replication                                             │
│                                                                          │
│   Layer 2: Cassandra                                                     │
│   ─────────────────────────────────────────────                         │
│   - Persistent storage                                                   │
│   - Source of truth                                                      │
│                                                                          │
│   Special: Hollow (local read-only datasets)                            │
│   ─────────────────────────────────────────────                         │
│   - Movie catalog (changes infrequently)                                │
│   - Loaded entirely into memory                                          │
│   - No cache misses!                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Facebook's TAO

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FACEBOOK TAO ARCHITECTURE                             │
│                                                                          │
│   TAO = The Associations and Objects cache                              │
│                                                                          │
│   Request Flow:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   Web Server                                                     │   │
│   │       │                                                          │   │
│   │       ▼                                                          │   │
│   │   TAO Leader (in-region cache)                                  │   │
│   │       │                                                          │   │
│   │       ├── HIT → Return (99%+ of requests)                       │   │
│   │       │                                                          │   │
│   │       └── MISS → TAO Follower (other region)                    │   │
│   │                   │                                              │   │
│   │                   ├── HIT → Return + async populate leader      │   │
│   │                   │                                              │   │
│   │                   └── MISS → MySQL (rare)                       │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Key insight: Leader caches HOT data for that region                   │
│                Follower acts as L2 with global data                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement in Java

### Caffeine (Local Cache)

```java
// CaffeineLocalCache.java
package com.example.cache.local;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;
import com.github.benmanes.caffeine.cache.stats.CacheStats;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.function.Function;

/**
 * Local cache implementation using Caffeine
 * 
 * Caffeine is the successor to Guava Cache with better performance
 * Uses Window TinyLFU eviction policy for near-optimal hit rates
 */
@Component
@Slf4j
public class CaffeineLocalCache<K, V> {

    private final Cache<K, V> cache;

    public CaffeineLocalCache() {
        this.cache = Caffeine.newBuilder()
            // Maximum entries in cache
            .maximumSize(10_000)
            
            // Expire after write (fixed TTL)
            .expireAfterWrite(Duration.ofMinutes(5))
            
            // Expire after last access (sliding TTL)
            // .expireAfterAccess(Duration.ofMinutes(10))
            
            // Enable statistics for monitoring
            .recordStats()
            
            // Listener for evictions (useful for debugging)
            .evictionListener((key, value, cause) -> 
                log.debug("Evicted {} due to {}", key, cause))
            
            // Build the cache
            .build();
    }

    /**
     * Get value, return null if not present
     */
    public V get(K key) {
        return cache.getIfPresent(key);
    }

    /**
     * Get value, compute if absent
     * This is atomic - only one computation for concurrent requests
     */
    public V get(K key, Function<K, V> loader) {
        return cache.get(key, loader);
    }

    /**
     * Put value in cache
     */
    public void put(K key, V value) {
        cache.put(key, value);
    }

    /**
     * Invalidate specific key
     */
    public void invalidate(K key) {
        cache.invalidate(key);
    }

    /**
     * Invalidate all keys
     */
    public void invalidateAll() {
        cache.invalidateAll();
    }

    /**
     * Get cache statistics
     */
    public CacheStats stats() {
        return cache.stats();
    }

    /**
     * Print statistics for monitoring
     */
    public void printStats() {
        CacheStats stats = cache.stats();
        log.info("""
            Cache Statistics:
            - Hit Rate: {:.2f}%
            - Miss Rate: {:.2f}%
            - Load Count: {}
            - Eviction Count: {}
            - Average Load Time: {:.2f}ms
            """,
            stats.hitRate() * 100,
            stats.missRate() * 100,
            stats.loadCount(),
            stats.evictionCount(),
            stats.averageLoadPenalty() / 1_000_000.0
        );
    }
}
```

### Multi-Level Cache (L1 Local + L2 Redis)

```java
// MultiLevelCache.java
package com.example.cache.multilevel;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;
import java.util.function.Supplier;

/**
 * Multi-level cache: L1 (Caffeine local) + L2 (Redis distributed)
 * 
 * L1 provides sub-millisecond access for hot data
 * L2 provides shared cache across all servers
 */
@Service
@Slf4j
public class MultiLevelCache<V> {

    private final Cache<String, V> l1Cache;
    private final RedisTemplate<String, V> redisTemplate;
    
    // Configuration
    private static final int L1_MAX_SIZE = 10_000;
    private static final Duration L1_TTL = Duration.ofMinutes(1);
    private static final Duration L2_TTL = Duration.ofMinutes(30);

    public MultiLevelCache(RedisTemplate<String, V> redisTemplate) {
        this.redisTemplate = redisTemplate;
        
        // Initialize L1 (local cache)
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(L1_MAX_SIZE)
            .expireAfterWrite(L1_TTL)
            .recordStats()
            .build();
    }

    /**
     * Get from multi-level cache
     * 
     * 1. Check L1 (local)
     * 2. Check L2 (Redis)
     * 3. Load from source
     */
    public V get(String key, Supplier<V> loader) {
        // Step 1: Check L1 (local cache)
        V value = l1Cache.getIfPresent(key);
        if (value != null) {
            log.debug("L1 HIT for {}", key);
            return value;
        }
        
        // Step 2: Check L2 (Redis)
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            log.debug("L2 HIT for {}", key);
            // Promote to L1
            l1Cache.put(key, value);
            return value;
        }
        
        // Step 3: Load from source (database)
        log.debug("MISS for {}, loading from source", key);
        value = loader.get();
        
        if (value != null) {
            // Store in both L1 and L2
            l1Cache.put(key, value);
            redisTemplate.opsForValue().set(key, value, L2_TTL);
        }
        
        return value;
    }

    /**
     * Invalidate from all levels
     */
    public void invalidate(String key) {
        l1Cache.invalidate(key);
        redisTemplate.delete(key);
        log.debug("Invalidated {} from all cache levels", key);
    }

    /**
     * Invalidate L1 only (for distributed invalidation)
     * Called when receiving invalidation message from other servers
     */
    public void invalidateL1(String key) {
        l1Cache.invalidate(key);
        log.debug("Invalidated {} from L1", key);
    }

    /**
     * Put directly (bypassing load)
     */
    public void put(String key, V value) {
        l1Cache.put(key, value);
        redisTemplate.opsForValue().set(key, value, L2_TTL);
    }

    /**
     * Get L1 stats for monitoring
     */
    public String getL1Stats() {
        var stats = l1Cache.stats();
        return String.format("L1 Hit Rate: %.2f%%, Size: %d", 
            stats.hitRate() * 100, l1Cache.estimatedSize());
    }
}
```

### L1 Invalidation via Redis Pub/Sub

```java
// CacheInvalidationBroadcaster.java
package com.example.cache.multilevel;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.stereotype.Service;

/**
 * Broadcasts cache invalidation messages to all servers
 * 
 * When one server invalidates a cache entry, all other servers
 * need to invalidate their L1 caches too.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class CacheInvalidationBroadcaster {

    private final RedisTemplate<String, String> redisTemplate;
    
    private static final String INVALIDATION_CHANNEL = "cache:invalidation";

    /**
     * Broadcast invalidation to all servers
     */
    public void broadcastInvalidation(String cacheKey) {
        log.debug("Broadcasting invalidation for {}", cacheKey);
        redisTemplate.convertAndSend(INVALIDATION_CHANNEL, cacheKey);
    }
}
```

```java
// CacheInvalidationListener.java
package com.example.cache.multilevel;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.stereotype.Component;

/**
 * Listens for cache invalidation messages from other servers
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheInvalidationListener implements MessageListener {

    private final MultiLevelCache<?> multiLevelCache;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String cacheKey = new String(message.getBody());
        log.debug("Received invalidation for {}", cacheKey);
        
        // Only invalidate L1 - L2 (Redis) is already updated
        multiLevelCache.invalidateL1(cacheKey);
    }
}
```

```java
// RedisListenerConfig.java
package com.example.cache.config;

import com.example.cache.multilevel.CacheInvalidationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;

@Configuration
public class RedisListenerConfig {

    @Bean
    public RedisMessageListenerContainer redisContainer(
            RedisConnectionFactory connectionFactory,
            CacheInvalidationListener listener) {
        
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listener, new ChannelTopic("cache:invalidation"));
        return container;
    }
}
```

### Complete Product Service Example

```java
// ProductService.java
package com.example.cache.service;

import com.example.cache.multilevel.CacheInvalidationBroadcaster;
import com.example.cache.multilevel.MultiLevelCache;
import com.example.domain.Product;
import com.example.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {

    private final MultiLevelCache<Product> cache;
    private final CacheInvalidationBroadcaster broadcaster;
    private final ProductRepository productRepository;

    private static final String KEY_PREFIX = "product:";

    /**
     * Get product with multi-level caching
     */
    public Product getProduct(Long productId) {
        String key = KEY_PREFIX + productId;
        
        return cache.get(key, () -> {
            log.info("Loading product {} from database", productId);
            return productRepository.findById(productId).orElse(null);
        });
    }

    /**
     * Update product - invalidate cache across all servers
     */
    public Product updateProduct(Long productId, Product updates) {
        Product saved = productRepository.save(updates);
        
        String key = KEY_PREFIX + productId;
        
        // Invalidate local cache and Redis
        cache.invalidate(key);
        
        // Broadcast to other servers to invalidate their L1
        broadcaster.broadcastInvalidation(key);
        
        return saved;
    }

    /**
     * Delete product - invalidate cache
     */
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);
        
        String key = KEY_PREFIX + productId;
        cache.invalidate(key);
        broadcaster.broadcastInvalidation(key);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Comparison Table

| Aspect | Local Cache | Distributed Cache | Multi-Level |
|--------|-------------|-------------------|-------------|
| **Latency** | ~50ns | ~1ms | 50ns (L1 hit) to 1ms |
| **Consistency** | Weak | Strong | Eventual |
| **Capacity** | Limited (heap) | Large (TB) | Both |
| **Failure Impact** | None | High | Medium |
| **Complexity** | Low | Medium | High |
| **Best For** | Hot data, config | Shared data | High-scale systems |

### Common Mistakes

**1. L1 TTL Too Long**

```java
// WRONG: L1 TTL same as L2
.expireAfterWrite(Duration.ofMinutes(30))  // Too long for L1!

// RIGHT: L1 should be much shorter
.expireAfterWrite(Duration.ofMinutes(1))   // Quick turnover
```

**2. Not Handling L1 Invalidation**

```java
// WRONG: Only invalidate Redis
public void updateProduct(Product product) {
    repository.save(product);
    redisTemplate.delete("product:" + product.getId());
    // L1 on other servers still has old data!
}

// RIGHT: Broadcast invalidation
public void updateProduct(Product product) {
    repository.save(product);
    cache.invalidate("product:" + product.getId());
    broadcaster.broadcastInvalidation("product:" + product.getId());
}
```

**3. L1 Too Large**

```java
// WRONG: L1 eats all your heap
.maximumSize(10_000_000)  // 10 million items = potential OOM

// RIGHT: Size L1 appropriately
.maximumSize(10_000)      // Keep it small, let L2 handle the rest
// Or use weight-based sizing
.maximumWeight(100_000_000)  // 100MB max
.weigher((k, v) -> estimateSize(v))
```

---

## 8️⃣ When NOT to Use Each Type

### Local Cache: Don't Use When

- Data must be consistent across servers immediately
- Cache size would exceed available heap
- Data is rarely accessed (wasted memory on each server)

### Distributed Cache: Don't Use When

- Sub-millisecond latency is required
- Network reliability is poor
- Data is server-specific (not shared)

### Multi-Level: Don't Use When

- System is simple (single server)
- Eventual consistency is unacceptable
- Operational complexity is a concern

---

## 9️⃣ Interview Follow-Up Questions WITH Answers

### L4 Questions

**Q: What's the difference between local and distributed cache?**

A: Local cache (like Caffeine) stores data in the application's memory (JVM heap). Access is extremely fast (~50 nanoseconds) but data isn't shared between servers. Distributed cache (like Redis) stores data on separate cache servers. Access is slower (~1 millisecond) due to network latency, but data is shared across all application servers. Local is best for hot data where speed is critical; distributed is best for shared state and large datasets.

### L5 Questions

**Q: How do you keep L1 caches synchronized across servers?**

A: I use pub/sub messaging. When one server updates data, it publishes an invalidation message to a Redis channel. All other servers subscribe to this channel and invalidate their L1 caches when they receive the message. This provides eventual consistency. For stronger consistency, I could use shorter L1 TTLs or version-based invalidation where the cache key includes a version number.

### L6 Questions

**Q: Design a caching strategy for a social media feed with 100M users.**

A: I'd use a three-level approach: (1) L1 local cache for the current user's feed and hot celebrity profiles with 30-second TTL. (2) L2 regional Redis cluster for user profiles and recent posts. (3) L3 global Redis for less frequently accessed data. For feed generation, I'd pre-compute feeds for active users and store in Redis. For inactive users, compute on-demand. Hot keys (celebrities) would be replicated across Redis nodes. I'd use pub/sub for L1 invalidation and implement circuit breakers to handle Redis failures gracefully.

---

## 🔟 One Clean Mental Summary

Local caches (Caffeine) provide sub-millisecond access but aren't shared between servers. Distributed caches (Redis) are shared but add network latency. Multi-level caching combines both: L1 local cache handles hot data with instant access, L2 distributed cache handles shared data. The key challenge with multi-level caching is keeping L1 caches synchronized. Use pub/sub or short TTLs for L1 invalidation. Size L1 small (thousands of items), let L2 handle the bulk. Monitor hit rates at each level to tune effectively.

