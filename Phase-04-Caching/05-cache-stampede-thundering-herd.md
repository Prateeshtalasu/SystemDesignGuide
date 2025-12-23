# 🐘 Cache Stampede (Thundering Herd)

---

## 0️⃣ Prerequisites

Before diving into cache stampede, you need to understand:

- **Cache**: A fast storage layer that reduces database load. Covered in Topics 1-4.
- **TTL (Time To Live)**: When cached data expires. Covered in Topic 2.
- **Cache Miss**: When requested data is not in cache, requiring a database query.
- **Concurrency**: Multiple requests happening at the same time.

If you understand that many users can request the same data simultaneously, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

You have a popular product page cached in Redis. The cache entry expires.

**What you expect**:
- One request sees cache miss
- One database query
- Cache is repopulated
- Subsequent requests hit cache

**What actually happens**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE THUNDERING HERD                                   │
│                                                                          │
│   Time: 10:00:00.000 - Cache entry expires                              │
│                                                                          │
│   Time: 10:00:00.001                                                     │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│   │Request 1│  │Request 2│  │Request 3│  │Request 4│  │Request 5│      │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│        │            │            │            │            │            │
│        │  Check     │  Check     │  Check     │  Check     │  Check    │
│        │  cache     │  cache     │  cache     │  cache     │  cache    │
│        ▼            ▼            ▼            ▼            ▼            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                        REDIS                                     │   │
│   │                                                                  │   │
│   │   GET product:123 → (nil)  MISS!                                │   │
│   │   GET product:123 → (nil)  MISS!                                │   │
│   │   GET product:123 → (nil)  MISS!                                │   │
│   │   GET product:123 → (nil)  MISS!                                │   │
│   │   GET product:123 → (nil)  MISS!                                │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│        │            │            │            │            │            │
│        │  Query     │  Query     │  Query     │  Query     │  Query    │
│        │  DB        │  DB        │  DB        │  DB        │  DB       │
│        ▼            ▼            ▼            ▼            ▼            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                       DATABASE                                   │   │
│   │                                                                  │   │
│   │   5 IDENTICAL QUERIES executing simultaneously!                 │   │
│   │                                                                  │   │
│   │   SELECT * FROM products WHERE id = 123                         │   │
│   │   SELECT * FROM products WHERE id = 123                         │   │
│   │   SELECT * FROM products WHERE id = 123                         │   │
│   │   SELECT * FROM products WHERE id = 123                         │   │
│   │   SELECT * FROM products WHERE id = 123                         │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   With 10,000 concurrent users: 10,000 identical database queries!      │
│                                                                          │
│   Result: Database overwhelmed, slow responses, potential crash         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why "Thundering Herd"?

The name comes from a herd of animals all stampeding at once when startled. In computing:
- The "startle" is the cache expiration
- The "herd" is all the concurrent requests
- The "stampede" is all requests hitting the database simultaneously

### What Breaks Without Prevention?

1. **Database Overload**: 
   - Normal: 100 queries/second
   - During stampede: 10,000 queries/second
   - Database can't handle the spike

2. **Cascading Failures**:
   - Database slows down
   - Application threads wait for database
   - Thread pool exhausted
   - Application becomes unresponsive
   - Load balancer marks servers as unhealthy
   - Remaining servers get even more traffic
   - Complete system failure

3. **Wasted Resources**:
   - All queries return the same data
   - Only one result is needed
   - 9,999 queries are pure waste

### Real Examples

**Reddit (Multiple Incidents)**:
- Popular post's cache expires
- Millions of users viewing simultaneously
- Database crushed by identical queries
- Site goes down

**Facebook (2010)**:
- Memcached cluster restart
- All cache entries gone
- Billions of requests hit database
- Had to implement warming strategies

**Twitter (Early Days)**:
- Celebrity tweets went viral
- Cache couldn't keep up with expiration
- "Fail Whale" became famous

---

## 2️⃣ Intuition and Mental Model

### The Concert Ticket Analogy

Imagine a concert ticket booth with one seller and thousands of fans:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE TICKET BOOTH PROBLEM                              │
│                                                                          │
│   WITHOUT STAMPEDE PREVENTION:                                           │
│                                                                          │
│   10:00 AM - Tickets go on sale                                         │
│                                                                          │
│   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                              │
│   │Fan 1│ │Fan 2│ │Fan 3│ │Fan 4│ │Fan 5│  ... 10,000 fans             │
│   └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                              │
│      │       │       │       │       │                                   │
│      └───────┴───────┴───────┴───────┘                                   │
│                      │                                                   │
│                      ▼                                                   │
│              ┌───────────────┐                                          │
│              │  ONE SELLER   │  ← Overwhelmed!                          │
│              │  (Database)   │                                          │
│              └───────────────┘                                          │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   WITH STAMPEDE PREVENTION:                                              │
│                                                                          │
│   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                              │
│   │Fan 1│ │Fan 2│ │Fan 3│ │Fan 4│ │Fan 5│                              │
│   └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                              │
│      │       │       │       │       │                                   │
│      │       │       │       │       │                                   │
│      ▼       ▼       ▼       ▼       ▼                                   │
│   ┌─────────────────────────────────────────┐                           │
│   │           QUEUE MANAGER                  │                           │
│   │   "Fan 1 gets to buy. Everyone else,    │                           │
│   │    please wait. I'll announce when      │                           │
│   │    tickets are available."              │                           │
│   └─────────────────────┬───────────────────┘                           │
│                         │                                                │
│                         ▼                                                │
│                 ┌───────────────┐                                       │
│                 │  ONE SELLER   │  ← Handles one request                │
│                 │  (Database)   │                                       │
│                 └───────────────┘                                       │
│                                                                          │
│   Fan 1 buys tickets → Tickets posted on board → Everyone sees them    │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Instead of everyone rushing the seller, one person goes while others wait. Once tickets are "posted" (cached), everyone can see them.

---

## 3️⃣ How It Works Internally

### The Root Cause

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHY STAMPEDE HAPPENS                                  │
│                                                                          │
│   The Gap Between Check and Set:                                         │
│                                                                          │
│   Request 1:                                                             │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache    │ Query DB │ SET cache │                               │
│   │ (miss)       │          │           │                               │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 2 (arrives during Request 1's DB query):                      │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache    │ Query DB │ SET cache │                               │
│   │ (miss)       │          │           │                               │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 3 (arrives during Request 1's DB query):                      │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache    │ Query DB │ SET cache │                               │
│   │ (miss)       │          │           │                               │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   The problem: All requests see "miss" before any can "set"             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Locking (Mutex)

**The Concept**: Only one request can fetch from database. Others wait for the result.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOCKING SOLUTION                                      │
│                                                                          │
│   Request 1:                                                             │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache │ ACQUIRE LOCK │ Query DB │ SET cache │ RELEASE LOCK │   │
│   │ (miss)    │ (success)    │          │           │              │   │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 2:                                                             │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache │ ACQUIRE LOCK │ ... waiting ...     │ GET cache │       │
│   │ (miss)    │ (blocked)    │                     │ (HIT!)    │       │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 3:                                                             │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache │ ACQUIRE LOCK │ ... waiting ...     │ GET cache │       │
│   │ (miss)    │ (blocked)    │                     │ (HIT!)    │       │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Result: 1 DB query instead of 3                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solution 2: Request Coalescing

**The Concept**: Group identical in-flight requests. Make one database call, share the result.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REQUEST COALESCING                                    │
│                                                                          │
│   In-Flight Request Tracker:                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Key: "product:123"                                              │   │
│   │  Status: LOADING                                                 │   │
│   │  Waiters: [Request2, Request3, Request4, ...]                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Request 1: First to arrive                                            │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ Check tracker │ Not found │ Register as loader │ Query DB │        │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 2: Arrives while Request 1 is loading                         │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ Check tracker │ Found LOADING │ Add self to waiters │ Wait... │    │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Request 1 completes:                                                   │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ Got result │ Notify all waiters │ Cache result │                   │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   All waiters receive the same result without additional DB queries     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solution 3: Probabilistic Early Expiration

**The Concept**: Refresh cache before it expires. Different requests have different "early refresh" probability.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROBABILISTIC EARLY EXPIRATION                        │
│                                                                          │
│   TTL = 60 seconds                                                       │
│                                                                          │
│   Time: 0s    Cache populated                                           │
│   Time: 30s   Request arrives, TTL = 30s remaining                      │
│               Refresh probability = low (30%)                            │
│               → Probably returns cached value                            │
│                                                                          │
│   Time: 50s   Request arrives, TTL = 10s remaining                      │
│               Refresh probability = medium (60%)                         │
│               → Might trigger background refresh                         │
│                                                                          │
│   Time: 58s   Request arrives, TTL = 2s remaining                       │
│               Refresh probability = high (95%)                           │
│               → Very likely triggers background refresh                  │
│                                                                          │
│   Formula: probability = 1 - (remaining_ttl / total_ttl)^β              │
│            where β controls aggressiveness (typically 2-4)              │
│                                                                          │
│   Result: Cache refreshed BEFORE expiration, no stampede!               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solution 4: Background Refresh

**The Concept**: Never let cache expire. Background job refreshes before TTL.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BACKGROUND REFRESH                                    │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    REFRESH SCHEDULER                             │   │
│   │                                                                  │   │
│   │  Hot Keys to Refresh:                                           │   │
│   │  ┌────────────────┬──────────────┬────────────────────┐        │   │
│   │  │ Key            │ TTL Remaining │ Next Refresh       │        │   │
│   │  ├────────────────┼──────────────┼────────────────────┤        │   │
│   │  │ product:123    │ 45s          │ In 30s             │        │   │
│   │  │ product:456    │ 120s         │ In 105s            │        │   │
│   │  │ user:789       │ 10s          │ NOW!               │        │   │
│   │  └────────────────┴──────────────┴────────────────────┘        │   │
│   │                                                                  │   │
│   │  Refresh at: TTL - buffer (e.g., 15 seconds before expiry)     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   User Requests:                                                         │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache │ Always HIT! │ Return immediately │                     │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Background Job (separate thread):                                      │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ Check schedule │ Key needs refresh │ Query DB │ Update cache │     │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   Result: Users never see cache miss for hot keys                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solution 5: Stale-While-Revalidate

**The Concept**: Return stale data immediately while refreshing in background.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STALE-WHILE-REVALIDATE                                │
│                                                                          │
│   Cache Entry Structure:                                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Key: "product:123"                                              │   │
│   │  Value: {...product data...}                                     │   │
│   │  Fresh Until: 10:00:00 (soft expiry)                            │   │
│   │  Stale Until: 10:05:00 (hard expiry)                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Request at 10:02:00 (after soft expiry, before hard expiry):          │
│   ─────────────────────────────────────────────────────────────────────  │
│   │ GET cache │ Data is STALE │ Return stale data │ Trigger refresh │  │
│   │           │ but valid     │ immediately       │ in background   │  │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   User gets response in 2ms (stale but acceptable)                      │
│   Background refresh updates cache for next request                     │
│                                                                          │
│   Timeline:                                                              │
│   ─────────────────────────────────────────────────────────────────────  │
│   10:00:00  Fresh period ends                                           │
│   10:00:00 - 10:05:00  Stale period (return stale + refresh)           │
│   10:05:00  Hard expiry (must refresh synchronously)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

Let's trace through a stampede scenario with and without prevention.

### Scenario: Product Page Cache Expires

**Setup**:
- Product ID: 123
- Cache TTL: 5 minutes
- Concurrent requests: 1000
- Database query time: 100ms

### Without Stampede Prevention

```
Timeline:
─────────────────────────────────────────────────────────────────────────
10:00:00.000  Cache expires for product:123
              │
10:00:00.001  Request 1 arrives
              │ GET product:123 → MISS
              │ Query database...
              │
10:00:00.002  Request 2 arrives
              │ GET product:123 → MISS
              │ Query database...
              │
10:00:00.003  Requests 3-1000 arrive
              │ GET product:123 → MISS (all of them)
              │ Query database... (all of them)
              │
10:00:00.100  Database receives 1000 identical queries
              │ Database CPU: 100%
              │ Query queue backing up
              │
10:00:00.500  Database struggling
              │ Query latency: 500ms (normally 100ms)
              │ Connection pool exhausted
              │
10:00:01.000  Some queries timing out
              │ Application threads blocked
              │ New requests queuing
              │
10:00:02.000  Cascade failure begins
              │ Health checks failing
              │ Load balancer removing servers
─────────────────────────────────────────────────────────────────────────

Result: 1000 database queries, potential system failure
```

### With Locking Prevention

```
Timeline:
─────────────────────────────────────────────────────────────────────────
10:00:00.000  Cache expires for product:123
              │
10:00:00.001  Request 1 arrives
              │ GET product:123 → MISS
              │ SETNX lock:product:123 → SUCCESS (acquired lock)
              │ Query database...
              │
10:00:00.002  Request 2 arrives
              │ GET product:123 → MISS
              │ SETNX lock:product:123 → FAIL (lock exists)
              │ Wait for lock to release...
              │
10:00:00.003  Requests 3-1000 arrive
              │ GET product:123 → MISS
              │ SETNX lock:product:123 → FAIL (all fail)
              │ Wait for lock to release... (all wait)
              │
10:00:00.100  Request 1 gets database response
              │ SET product:123 {...} EX 300
              │ DEL lock:product:123
              │
10:00:00.101  Requests 2-1000 retry
              │ GET product:123 → HIT! (all of them)
              │ Return cached data
─────────────────────────────────────────────────────────────────────────

Result: 1 database query, all requests served successfully
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Facebook's Lease Mechanism

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FACEBOOK'S LEASE MECHANISM                            │
│                                                                          │
│   Instead of simple locking, Facebook uses "leases":                    │
│                                                                          │
│   1. Client requests data                                                │
│   2. Cache miss → Client gets a "lease" (permission to fetch)           │
│   3. Other clients see "lease in progress" → Wait or get stale data    │
│   4. First client fetches from DB, updates cache, releases lease        │
│                                                                          │
│   Lease includes:                                                        │
│   - Lease ID (to verify ownership)                                      │
│   - Expiration (auto-release if client dies)                            │
│   - Stale value (optional, for stale-while-revalidate)                 │
│                                                                          │
│   Benefits:                                                              │
│   - Prevents stampede                                                    │
│   - Handles client failures (lease expires)                             │
│   - Can return stale data instead of waiting                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### Netflix's Request Coalescing

```java
// Netflix uses Hystrix/Resilience4j with request collapsing
// Multiple requests for same data within a time window are batched

// Simplified concept:
public class RequestCollapser {
    private final Map<String, CompletableFuture<Product>> inFlight = new ConcurrentHashMap<>();
    
    public CompletableFuture<Product> getProduct(String productId) {
        return inFlight.computeIfAbsent(productId, id -> {
            CompletableFuture<Product> future = fetchFromDatabase(id);
            future.whenComplete((result, error) -> inFlight.remove(id));
            return future;
        });
    }
}
```

### Real-World Configuration

**Redis Configuration for Stampede Prevention**:

```conf
# redis.conf

# Use probabilistic expiration
# Keys don't all expire at exactly the same time
# Add random jitter to TTL
```

**Application Configuration**:

```yaml
# application.yml
cache:
  stampede-prevention:
    strategy: locking  # or "probabilistic", "stale-while-revalidate"
    lock-timeout: 5s
    lock-key-prefix: "lock:"
    
  stale-while-revalidate:
    enabled: true
    stale-ttl: 60s  # Return stale data for up to 60s while refreshing
```

---

## 6️⃣ How to Implement Stampede Prevention in Java

### Solution 1: Distributed Locking

```java
// StampedeLockingCache.java
package com.example.cache.stampede;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.UUID;
import java.util.function.Supplier;

/**
 * Cache with distributed locking to prevent stampede
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class StampedeLockingCache {

    private final StringRedisTemplate redisTemplate;
    
    private static final Duration LOCK_TIMEOUT = Duration.ofSeconds(10);
    private static final Duration LOCK_WAIT_TIMEOUT = Duration.ofSeconds(5);
    private static final int LOCK_RETRY_DELAY_MS = 50;

    /**
     * Get from cache with stampede prevention via locking
     * 
     * @param key Cache key
     * @param ttl TTL for cached value
     * @param loader Function to load value on cache miss
     * @return Cached or freshly loaded value
     */
    public String getWithLocking(String key, Duration ttl, Supplier<String> loader) {
        // Step 1: Try to get from cache
        String cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            log.debug("Cache HIT for {}", key);
            return cached;
        }
        
        // Step 2: Cache miss - try to acquire lock
        String lockKey = "lock:" + key;
        String lockValue = UUID.randomUUID().toString();
        
        boolean lockAcquired = tryAcquireLock(lockKey, lockValue, LOCK_TIMEOUT);
        
        if (lockAcquired) {
            try {
                // Step 3: Double-check cache (might have been populated while waiting for lock)
                cached = redisTemplate.opsForValue().get(key);
                if (cached != null) {
                    log.debug("Cache HIT after lock acquisition for {}", key);
                    return cached;
                }
                
                // Step 4: Load from source
                log.info("Loading {} from source (lock holder)", key);
                String value = loader.get();
                
                // Step 5: Cache the value
                if (value != null) {
                    redisTemplate.opsForValue().set(key, value, ttl);
                }
                
                return value;
            } finally {
                // Step 6: Release lock
                releaseLock(lockKey, lockValue);
            }
        } else {
            // Step 7: Couldn't acquire lock - wait for cache to be populated
            log.debug("Waiting for cache population for {}", key);
            return waitForCache(key, LOCK_WAIT_TIMEOUT);
        }
    }

    /**
     * Try to acquire distributed lock
     */
    private boolean tryAcquireLock(String lockKey, String lockValue, Duration timeout) {
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, timeout);
        return Boolean.TRUE.equals(acquired);
    }

    /**
     * Release distributed lock (only if we own it)
     */
    private void releaseLock(String lockKey, String lockValue) {
        String script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
            """;
        
        redisTemplate.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(script, Long.class),
            java.util.Collections.singletonList(lockKey),
            lockValue
        );
    }

    /**
     * Wait for cache to be populated by another thread
     */
    private String waitForCache(String key, Duration timeout) {
        long startTime = System.currentTimeMillis();
        long timeoutMs = timeout.toMillis();
        
        while (System.currentTimeMillis() - startTime < timeoutMs) {
            String cached = redisTemplate.opsForValue().get(key);
            if (cached != null) {
                log.debug("Cache populated by another thread for {}", key);
                return cached;
            }
            
            try {
                Thread.sleep(LOCK_RETRY_DELAY_MS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        
        // Timeout - fall back to loading ourselves
        log.warn("Timeout waiting for cache {}, loading directly", key);
        return null;  // Caller should handle null and load from source
    }
}
```

### Solution 2: Request Coalescing with CompletableFuture

```java
// RequestCoalescingCache.java
package com.example.cache.stampede;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.function.Supplier;

/**
 * Cache with request coalescing to prevent stampede
 * 
 * Multiple requests for the same key share a single database query
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class RequestCoalescingCache {

    private final StringRedisTemplate redisTemplate;
    
    // Track in-flight requests
    // Key -> Future that will contain the result
    private final ConcurrentMap<String, CompletableFuture<String>> inFlightRequests = 
        new ConcurrentHashMap<>();

    /**
     * Get from cache with request coalescing
     * 
     * If multiple requests arrive for the same key while loading,
     * they all share the same database query result.
     */
    public CompletableFuture<String> getAsync(String key, Duration ttl, 
                                               Supplier<String> loader) {
        // Step 1: Try to get from cache
        String cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            log.debug("Cache HIT for {}", key);
            return CompletableFuture.completedFuture(cached);
        }
        
        // Step 2: Check if there's already an in-flight request for this key
        // computeIfAbsent is atomic - only one thread creates the future
        return inFlightRequests.computeIfAbsent(key, k -> {
            log.info("Creating loader for {} (first request)", key);
            
            // Create the loading future
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                // Double-check cache
                String doubleCheck = redisTemplate.opsForValue().get(key);
                if (doubleCheck != null) {
                    return doubleCheck;
                }
                
                // Load from source
                log.info("Loading {} from source", key);
                return loader.get();
            });
            
            // When complete, cache the result and remove from in-flight
            future.whenComplete((result, error) -> {
                inFlightRequests.remove(key);
                
                if (error == null && result != null) {
                    redisTemplate.opsForValue().set(key, result, ttl);
                    log.debug("Cached {} after loading", key);
                }
            });
            
            return future;
        });
    }

    /**
     * Synchronous version for simpler usage
     */
    public String get(String key, Duration ttl, Supplier<String> loader) {
        try {
            return getAsync(key, ttl, loader).get();
        } catch (Exception e) {
            log.error("Error getting {}: {}", key, e.getMessage());
            throw new RuntimeException("Cache get failed", e);
        }
    }
}
```

### Solution 3: Probabilistic Early Expiration

```java
// ProbabilisticExpirationCache.java
package com.example.cache.stampede;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Supplier;

/**
 * Cache with probabilistic early expiration (XFetch algorithm)
 * 
 * As TTL approaches expiration, probability of refresh increases.
 * This spreads out refreshes and prevents all requests from
 * hitting database at exactly the same time.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class ProbabilisticExpirationCache {

    private final StringRedisTemplate redisTemplate;
    
    // Beta parameter controls how aggressive early expiration is
    // Higher beta = more aggressive early expiration
    private static final double BETA = 2.0;

    /**
     * Get with probabilistic early expiration
     * 
     * @param key Cache key
     * @param ttl Total TTL for the key
     * @param loader Function to load value
     * @return Cached or refreshed value
     */
    public String getWithProbabilisticExpiration(String key, Duration ttl, 
                                                  Supplier<String> loader) {
        String cached = redisTemplate.opsForValue().get(key);
        
        if (cached != null) {
            // Check if we should probabilistically refresh
            Long remainingTtl = redisTemplate.getExpire(key);
            
            if (remainingTtl != null && remainingTtl > 0) {
                double totalTtlSeconds = ttl.getSeconds();
                double remainingRatio = remainingTtl / totalTtlSeconds;
                
                // XFetch algorithm: probability increases as TTL decreases
                // P(refresh) = 1 - remainingRatio^beta
                double refreshProbability = 1.0 - Math.pow(remainingRatio, BETA);
                
                if (ThreadLocalRandom.current().nextDouble() < refreshProbability) {
                    log.info("Probabilistic early refresh triggered for {} " +
                             "(remaining: {}s, probability: {:.2f})", 
                             key, remainingTtl, refreshProbability);
                    
                    // Refresh in background, return current value
                    refreshInBackground(key, ttl, loader);
                }
            }
            
            return cached;
        }
        
        // Cache miss - load synchronously
        log.info("Cache MISS for {}, loading from source", key);
        String value = loader.get();
        
        if (value != null) {
            redisTemplate.opsForValue().set(key, value, ttl);
        }
        
        return value;
    }

    /**
     * Refresh cache in background thread
     */
    private void refreshInBackground(String key, Duration ttl, Supplier<String> loader) {
        // Use virtual threads (Java 21+) or executor service
        Thread.startVirtualThread(() -> {
            try {
                String freshValue = loader.get();
                if (freshValue != null) {
                    redisTemplate.opsForValue().set(key, freshValue, ttl);
                    log.debug("Background refresh completed for {}", key);
                }
            } catch (Exception e) {
                log.error("Background refresh failed for {}: {}", key, e.getMessage());
            }
        });
    }
}
```

### Solution 4: Stale-While-Revalidate

```java
// StaleWhileRevalidateCache.java
package com.example.cache.stampede;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.Data;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;
import java.util.function.Supplier;

/**
 * Cache with stale-while-revalidate pattern
 * 
 * Returns stale data immediately while refreshing in background.
 * User gets fast response, cache gets updated for next request.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class StaleWhileRevalidateCache {

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    /**
     * Wrapper that includes freshness metadata
     */
    @Data
    public static class CacheEntry {
        private String value;
        private Instant createdAt;
        private Instant freshUntil;  // Soft expiry
        private Instant staleUntil;  // Hard expiry
        
        public boolean isFresh() {
            return Instant.now().isBefore(freshUntil);
        }
        
        public boolean isStale() {
            Instant now = Instant.now();
            return now.isAfter(freshUntil) && now.isBefore(staleUntil);
        }
        
        public boolean isExpired() {
            return Instant.now().isAfter(staleUntil);
        }
    }

    /**
     * Get with stale-while-revalidate
     * 
     * @param key Cache key
     * @param freshDuration How long data is considered fresh
     * @param staleDuration How long stale data can be served
     * @param loader Function to load fresh data
     * @return Value (fresh or stale)
     */
    public String getWithSwr(String key, Duration freshDuration, 
                             Duration staleDuration, Supplier<String> loader) {
        
        String entryJson = redisTemplate.opsForValue().get(key);
        
        if (entryJson != null) {
            try {
                CacheEntry entry = objectMapper.readValue(entryJson, CacheEntry.class);
                
                if (entry.isFresh()) {
                    // Fresh data - return immediately
                    log.debug("Cache FRESH for {}", key);
                    return entry.getValue();
                }
                
                if (entry.isStale()) {
                    // Stale data - return immediately but refresh in background
                    log.info("Cache STALE for {}, returning stale and refreshing", key);
                    refreshInBackground(key, freshDuration, staleDuration, loader);
                    return entry.getValue();
                }
                
                // Expired - need synchronous refresh
                log.info("Cache EXPIRED for {}", key);
            } catch (JsonProcessingException e) {
                log.error("Failed to parse cache entry: {}", e.getMessage());
            }
        }
        
        // Cache miss or expired - load synchronously
        return loadAndCache(key, freshDuration, staleDuration, loader);
    }

    /**
     * Load from source and cache with SWR metadata
     */
    private String loadAndCache(String key, Duration freshDuration, 
                                Duration staleDuration, Supplier<String> loader) {
        String value = loader.get();
        
        if (value != null) {
            CacheEntry entry = new CacheEntry();
            entry.setValue(value);
            entry.setCreatedAt(Instant.now());
            entry.setFreshUntil(Instant.now().plus(freshDuration));
            entry.setStaleUntil(Instant.now().plus(freshDuration).plus(staleDuration));
            
            try {
                String entryJson = objectMapper.writeValueAsString(entry);
                // Set Redis TTL to stale duration (hard expiry)
                Duration totalTtl = freshDuration.plus(staleDuration);
                redisTemplate.opsForValue().set(key, entryJson, totalTtl);
                log.debug("Cached {} with SWR (fresh: {}, stale: {})", 
                         key, freshDuration, staleDuration);
            } catch (JsonProcessingException e) {
                log.error("Failed to cache entry: {}", e.getMessage());
            }
        }
        
        return value;
    }

    /**
     * Refresh in background
     */
    private void refreshInBackground(String key, Duration freshDuration, 
                                     Duration staleDuration, Supplier<String> loader) {
        Thread.startVirtualThread(() -> {
            try {
                loadAndCache(key, freshDuration, staleDuration, loader);
                log.debug("Background refresh completed for {}", key);
            } catch (Exception e) {
                log.error("Background refresh failed for {}: {}", key, e.getMessage());
            }
        });
    }
}
```

### Unified Cache Service with Multiple Strategies

```java
// StampedePreventionCacheService.java
package com.example.cache.stampede;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

/**
 * Unified cache service with configurable stampede prevention
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class StampedePreventionCacheService {

    private final StampedeLockingCache lockingCache;
    private final RequestCoalescingCache coalescingCache;
    private final ProbabilisticExpirationCache probabilisticCache;
    private final StaleWhileRevalidateCache swrCache;

    public enum Strategy {
        LOCKING,
        COALESCING,
        PROBABILISTIC,
        STALE_WHILE_REVALIDATE
    }

    /**
     * Get from cache with specified stampede prevention strategy
     */
    public String get(String key, Duration ttl, Supplier<String> loader, Strategy strategy) {
        return switch (strategy) {
            case LOCKING -> lockingCache.getWithLocking(key, ttl, loader);
            case COALESCING -> coalescingCache.get(key, ttl, loader);
            case PROBABILISTIC -> probabilisticCache.getWithProbabilisticExpiration(key, ttl, loader);
            case STALE_WHILE_REVALIDATE -> swrCache.getWithSwr(key, ttl, Duration.ofMinutes(1), loader);
        };
    }

    /**
     * Convenience method with default strategy (locking)
     */
    public String get(String key, Duration ttl, Supplier<String> loader) {
        return get(key, ttl, loader, Strategy.LOCKING);
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Strategy Comparison

| Strategy | Latency | Complexity | Consistency | Best For |
|----------|---------|------------|-------------|----------|
| Locking | Higher (waiters blocked) | Medium | Strong | Critical data |
| Coalescing | Medium | Medium | Strong | High concurrency |
| Probabilistic | Low | Low | Eventual | Hot keys |
| Stale-While-Revalidate | Lowest | Medium | Eventual | User-facing APIs |
| Background Refresh | Lowest | High | Eventual | Predictable hot keys |

### Common Mistakes

**1. Lock Without Timeout**

```java
// WRONG: Lock can be held forever if process dies
redisTemplate.opsForValue().setIfAbsent("lock:" + key, "1");

// RIGHT: Always set expiration
redisTemplate.opsForValue().setIfAbsent("lock:" + key, "1", Duration.ofSeconds(10));
```

**2. Not Handling Lock Holder Failure**

```java
// WRONG: If lock holder crashes, lock stays forever
if (acquireLock(key)) {
    loadData();  // What if this throws exception?
    releaseLock(key);
}

// RIGHT: Use try-finally
if (acquireLock(key)) {
    try {
        loadData();
    } finally {
        releaseLock(key);
    }
}
```

**3. Releasing Wrong Lock**

```java
// WRONG: Might release someone else's lock
redisTemplate.delete("lock:" + key);

// RIGHT: Only release if we own it
String script = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    end
    return 0
    """;
redisTemplate.execute(script, List.of("lock:" + key), myLockValue);
```

**4. Infinite Waiting**

```java
// WRONG: Wait forever for cache
while (cache.get(key) == null) {
    Thread.sleep(100);
}

// RIGHT: Set timeout
long deadline = System.currentTimeMillis() + 5000;
while (cache.get(key) == null && System.currentTimeMillis() < deadline) {
    Thread.sleep(100);
}
```

---

## 8️⃣ When NOT to Use Each Strategy

### Locking: Don't Use When
- Latency is critical (waiting adds latency)
- Lock contention would be very high
- Simple probabilistic approach is sufficient

### Request Coalescing: Don't Use When
- Requests are for different data (no coalescing benefit)
- Memory for tracking in-flight requests is limited
- System is single-threaded

### Probabilistic: Don't Use When
- Strong consistency is required
- Cache miss is very expensive (might trigger multiple refreshes)
- TTL is very short (not enough time for probability to work)

### Stale-While-Revalidate: Don't Use When
- Data must always be fresh (financial, security)
- Stale data could cause incorrect behavior
- Users would notice stale data

---

## 9️⃣ Comparison: Choosing the Right Strategy

### Decision Guide

```
                        What's your priority?
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
      Consistency         Latency            Simplicity
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │   Locking   │    │    SWR or   │    │Probabilistic│
    │     or      │    │ Background  │    │   Early     │
    │ Coalescing  │    │   Refresh   │    │ Expiration  │
    └─────────────┘    └─────────────┘    └─────────────┘
```

### Real-World Choices

| Company | Use Case | Strategy | Why |
|---------|----------|----------|-----|
| **Facebook** | User profiles | Lease (locking variant) | Consistency matters |
| **Netflix** | Recommendations | Stale-while-revalidate | Latency critical |
| **Twitter** | Tweet cache | Request coalescing | High concurrency |
| **Amazon** | Product catalog | Background refresh | Predictable hot items |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is cache stampede and why is it a problem?**

A: Cache stampede (or thundering herd) occurs when a popular cache entry expires and many concurrent requests all see the cache miss simultaneously. Instead of one request fetching from the database and caching the result, hundreds or thousands of requests all query the database at once. This can overwhelm the database, cause slow responses, and potentially crash the system. The solution is to ensure only one request fetches from the database while others wait for the result.

**Q: What's the simplest way to prevent cache stampede?**

A: The simplest approach is distributed locking. When a cache miss occurs, the first request acquires a lock (using Redis SETNX with expiration), fetches from the database, caches the result, and releases the lock. Other requests that see the cache miss try to acquire the lock, fail, and wait for the cache to be populated. This ensures only one database query regardless of concurrent requests.

### L5 (Mid-Level) Questions

**Q: Compare locking vs request coalescing for stampede prevention.**

A: Both prevent stampede but work differently. Locking uses a distributed lock, where one request holds the lock and others poll/wait. It's simpler but adds latency for waiters. Request coalescing tracks in-flight requests in memory. Multiple requests for the same key share a single CompletableFuture. When the first request completes, all waiters get the result. Coalescing is more efficient (no polling) but requires more memory and only works within a single application instance. For distributed systems, locking is more common. For high-concurrency single instances, coalescing is better.

**Q: How would you implement stale-while-revalidate?**

A: I'd store cache entries with two expiration times: fresh-until (soft expiry) and stale-until (hard expiry). On cache hit: if fresh, return immediately. If stale (past fresh-until but before stale-until), return the stale value immediately but trigger a background refresh. If expired (past stale-until), refresh synchronously. This gives users fast responses while keeping data reasonably fresh. The tradeoff is users might see slightly outdated data, which is acceptable for most use cases like product catalogs or user profiles.

### L6 (Senior) Questions

**Q: Design a cache warming strategy for a system that experiences predictable traffic spikes (e.g., daily at 9 AM).**

A: I'd implement a multi-layered approach:

**1. Predictive Warming (30 minutes before spike)**:
- Analyze historical access patterns to identify hot keys
- Background job pre-fetches these keys from database
- Set TTL to extend past the spike period

**2. Probabilistic Early Refresh (during normal operation)**:
- Use XFetch algorithm to refresh keys before expiration
- Spread refreshes over time to avoid synchronized expiration

**3. Stale-While-Revalidate (as safety net)**:
- If a key does expire during spike, serve stale data
- Refresh in background
- Users get fast response, cache stays warm

**4. Monitoring and Adaptation**:
- Track cache hit rate in real-time
- If hit rate drops, increase warming aggressiveness
- Alert if database query rate spikes

**5. Circuit Breaker**:
- If database is overwhelmed, fail fast
- Return cached data even if expired
- Gradually restore normal operation

This approach handles both predictable spikes (warming) and unexpected load (SWR + circuit breaker).

---

## 1️⃣1️⃣ One Clean Mental Summary

Cache stampede occurs when a popular cache entry expires and many concurrent requests all hit the database simultaneously, potentially overwhelming it. **Locking** ensures only one request fetches from the database while others wait. **Request coalescing** groups concurrent requests to share a single database query. **Probabilistic early expiration** refreshes cache before it expires, spreading load over time. **Stale-while-revalidate** returns stale data immediately while refreshing in background, prioritizing latency. **Background refresh** proactively refreshes hot keys before expiration. Choose based on your priorities: locking for consistency, SWR for latency, probabilistic for simplicity. Most production systems use a combination of these strategies.

