# 🔗 Cache Coherency

---

## 0️⃣ Prerequisites

Before diving into cache coherency, you need to understand:

- **Cache**: A fast storage layer for frequently accessed data. Covered in Topics 1-6.
- **Consistency**: Whether different parts of a system see the same data at the same time.
- **Distributed Systems**: Multiple servers working together. Different servers may have different cache states.
- **Race Conditions**: When the outcome depends on the timing of operations.

If you understand that multiple servers can have different cached versions of the same data, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

You have an e-commerce site with 3 servers. A product's price changes from $100 to $80.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE COHERENCY PROBLEM                                 │
│                                                                          │
│   Time: 10:00:00 - Admin updates price: $100 → $80                      │
│                                                                          │
│   Server 1 (updated)    Server 2 (stale)    Server 3 (stale)           │
│   ┌────────────────┐    ┌────────────────┐    ┌────────────────┐        │
│   │ Cache:         │    │ Cache:         │    │ Cache:         │        │
│   │ price = $80    │    │ price = $100   │    │ price = $100   │        │
│   └────────────────┘    └────────────────┘    └────────────────┘        │
│                                                                          │
│   User A                User B                User C                     │
│   sees $80              sees $100             sees $100                  │
│   ✅ Correct            ❌ Wrong              ❌ Wrong                   │
│                                                                          │
│   User B adds to cart at $100                                           │
│   Checkout shows $80                                                     │
│   User B is confused and angry!                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### What is Cache Coherency?

**Cache Coherency** is the discipline of ensuring that multiple caches present a consistent view of data. When data changes, all caches should either:
1. Have the updated value, OR
2. Not have the value (forcing a fresh fetch)

### The Coherency Problem in Distributed Systems

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHY COHERENCY IS HARD                                 │
│                                                                          │
│   Classic Problem: Update arrives at different times                    │
│                                                                          │
│   Timeline:                                                              │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   10:00:00.000  Database updated: price = $80                           │
│                                                                          │
│   10:00:00.001  Invalidation sent to all caches                         │
│                                                                          │
│   10:00:00.002  Server 1 receives invalidation → Cache cleared          │
│                                                                          │
│   10:00:00.003  User on Server 2 requests product                       │
│                 Server 2 hasn't received invalidation yet!              │
│                 Returns stale $100 from cache                           │
│                                                                          │
│   10:00:00.005  Server 2 receives invalidation → Cache cleared          │
│                 (Too late, user already saw $100)                       │
│                                                                          │
│   The window between update and invalidation = Inconsistency window     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Real Examples of Coherency Failures

**Amazon (2014)**: A flash sale started, prices updated in database, but some edge caches still showed old prices. Customers saw different prices on different pages.

**Twitter**: User blocks someone, but block list cache isn't invalidated across all servers. Blocked user can still see some tweets temporarily.

**Banking**: Account balance cached. Deposit and withdrawal happen on different servers. Both servers have stale balance.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Menu Board Analogy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE MENU BOARD ANALOGY                                │
│                                                                          │
│   Imagine a restaurant chain with 3 locations.                          │
│   Each location has a menu board (cache).                               │
│   Head office (database) sets the prices.                               │
│                                                                          │
│   WITHOUT COHERENCY:                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │   Head Office: "Burger is now $8" (was $10)                     │   │
│   │                                                                  │   │
│   │   Location A          Location B          Location C            │   │
│   │   ┌──────────┐       ┌──────────┐       ┌──────────┐           │   │
│   │   │ Burger   │       │ Burger   │       │ Burger   │           │   │
│   │   │ $8       │       │ $10      │       │ $10      │           │   │
│   │   └──────────┘       └──────────┘       └──────────┘           │   │
│   │   (Updated)          (Not yet)          (Not yet)              │   │
│   │                                                                  │   │
│   │   Customer at B orders at $10, expects $8 like they saw online │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   WITH COHERENCY:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │   Head Office: "Burger is now $8"                               │   │
│   │                     │                                            │   │
│   │                     │ IMMEDIATELY broadcasts                     │   │
│   │                     │ "Update your burger price!"                │   │
│   │                     ▼                                            │   │
│   │   ┌──────────────────────────────────────────────────────────┐  │   │
│   │   │              CENTRAL BROADCAST SYSTEM                     │  │   │
│   │   │   All locations receive message within seconds           │  │   │
│   │   │   All update their menu boards simultaneously            │  │   │
│   │   └──────────────────────────────────────────────────────────┘  │   │
│   │                                                                  │   │
│   │   Location A          Location B          Location C            │   │
│   │   ┌──────────┐       ┌──────────┐       ┌──────────┐           │   │
│   │   │ Burger   │       │ Burger   │       │ Burger   │           │   │
│   │   │ $8       │       │ $8       │       │ $8       │           │   │
│   │   └──────────┘       └──────────┘       └──────────┘           │   │
│   │   (All consistent!)                                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Coherency Models

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CACHE COHERENCY MODELS                                │
│                                                                          │
│   STRONG COHERENCY                                                       │
│   ─────────────────                                                     │
│   All caches always have the same value                                 │
│   Write completes only when all caches updated                          │
│   High latency, high consistency                                         │
│                                                                          │
│   Write: [Update DB] → [Update ALL caches] → [Acknowledge]             │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   EVENTUAL COHERENCY                                                     │
│   ──────────────────                                                    │
│   Caches converge to same value eventually                              │
│   Write completes immediately                                            │
│   Low latency, temporary inconsistency                                   │
│                                                                          │
│   Write: [Update DB] → [Acknowledge] → [Async invalidate caches]       │
│                        (returns here)                                    │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│   READ-YOUR-WRITES                                                       │
│   ────────────────                                                      │
│   A user always sees their own writes                                   │
│   Other users may see stale data temporarily                            │
│                                                                          │
│   Write: [Update DB] → [Update LOCAL cache] → [Acknowledge]            │
│                                   ↓                                      │
│                        [Async invalidate other caches]                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Coherency Protocols

**Protocol 1: Invalidation-Based**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INVALIDATION PROTOCOL                                 │
│                                                                          │
│   On Write:                                                              │
│   1. Update database                                                     │
│   2. Send invalidation message to all caches                            │
│   3. Each cache deletes the entry                                       │
│   4. Next read fetches fresh data                                       │
│                                                                          │
│   Server 1          Message Bus          Server 2, 3, ...               │
│      │                  │                      │                        │
│      │  Update DB       │                      │                        │
│      │───────────────────────────────────────▶│                        │
│      │                  │                      │                        │
│      │  INVALIDATE      │                      │                        │
│      │  product:123     │                      │                        │
│      │─────────────────▶│─────────────────────▶│                        │
│      │                  │                      │                        │
│      │                  │           DEL product:123                     │
│      │                  │                      │                        │
│                                                                          │
│   Pros: Simple, low bandwidth                                           │
│   Cons: Next read causes cache miss                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Protocol 2: Update-Based**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    UPDATE PROTOCOL                                       │
│                                                                          │
│   On Write:                                                              │
│   1. Update database                                                     │
│   2. Send new value to all caches                                       │
│   3. Each cache updates its entry                                       │
│                                                                          │
│   Server 1          Message Bus          Server 2, 3, ...               │
│      │                  │                      │                        │
│      │  Update DB       │                      │                        │
│      │───────────────────────────────────────▶│                        │
│      │                  │                      │                        │
│      │  UPDATE          │                      │                        │
│      │  product:123     │                      │                        │
│      │  = {new data}    │                      │                        │
│      │─────────────────▶│─────────────────────▶│                        │
│      │                  │                      │                        │
│      │                  │         SET product:123 = {new data}         │
│      │                  │                      │                        │
│                                                                          │
│   Pros: No cache miss after update                                      │
│   Cons: High bandwidth (sending full objects), ordering issues          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Protocol 3: Version-Based**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VERSION-BASED PROTOCOL                                │
│                                                                          │
│   Each cached entry includes a version number                           │
│                                                                          │
│   Cache Entry: {                                                         │
│     key: "product:123",                                                  │
│     value: {...},                                                        │
│     version: 5                                                           │
│   }                                                                      │
│                                                                          │
│   On Read:                                                               │
│   1. Get from cache                                                      │
│   2. Check version against database (lightweight query)                 │
│   3. If version matches, return cached value                            │
│   4. If version differs, invalidate and fetch fresh                     │
│                                                                          │
│   Pros: No coordination needed, works with network partitions           │
│   Cons: Extra database call to check version                            │
│                                                                          │
│   Optimization: Check version periodically or on suspicious operations  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Scenario: User Updates Profile Picture

**Setup**:
- 3 application servers
- Local L1 cache (Caffeine) + Distributed L2 cache (Redis)
- User updates profile picture

### Without Coherency Protocol

```
Timeline:
─────────────────────────────────────────────────────────────────────────
10:00:00.000  User on Server 1 uploads new profile picture
              │
              │  Server 1:
              │  - Updates database: user:123.avatar = "new.jpg"
              │  - Updates local L1 cache
              │  - Updates Redis L2 cache
              │
10:00:00.050  Update complete on Server 1
              │
              │  Server 1 L1: avatar = "new.jpg" ✅
              │  Redis L2:    avatar = "new.jpg" ✅
              │  Server 2 L1: avatar = "old.jpg" ❌ (stale!)
              │  Server 3 L1: avatar = "old.jpg" ❌ (stale!)
              │
10:00:00.100  User's friend on Server 2 views profile
              │
              │  Server 2 checks L1: HIT! Returns "old.jpg"
              │  Friend sees old profile picture
              │
10:00:01.000  Server 2 L1 TTL expires
              │
              │  Server 2 L1: (empty)
              │  Server 2 fetches from L2: avatar = "new.jpg"
              │  Now consistent, but 1 second of inconsistency!
─────────────────────────────────────────────────────────────────────────
```

### With Invalidation-Based Coherency

```
Timeline:
─────────────────────────────────────────────────────────────────────────
10:00:00.000  User on Server 1 uploads new profile picture
              │
              │  Server 1:
              │  - Updates database: user:123.avatar = "new.jpg"
              │  - Updates local L1 cache
              │  - Updates Redis L2 cache
              │  - PUBLISHES: "INVALIDATE user:123" to Redis channel
              │
10:00:00.005  Redis broadcasts invalidation to all subscribers
              │
              │  Server 2 receives: INVALIDATE user:123
              │  Server 2: DELETE user:123 from L1
              │
              │  Server 3 receives: INVALIDATE user:123
              │  Server 3: DELETE user:123 from L1
              │
10:00:00.010  All L1 caches cleared
              │
              │  Server 1 L1: avatar = "new.jpg" ✅
              │  Redis L2:    avatar = "new.jpg" ✅
              │  Server 2 L1: (empty, will fetch fresh) ✅
              │  Server 3 L1: (empty, will fetch fresh) ✅
              │
10:00:00.100  User's friend on Server 2 views profile
              │
              │  Server 2 checks L1: MISS
              │  Server 2 fetches from L2: avatar = "new.jpg" ✅
              │  Friend sees NEW profile picture!
─────────────────────────────────────────────────────────────────────────

Inconsistency window: ~10ms (vs ~1 second without protocol)
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Facebook's Cache Coherency

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FACEBOOK'S APPROACH                                   │
│                                                                          │
│   Problem: 1000+ Memcached servers across regions                       │
│                                                                          │
│   Solution: McSqueal + Remote Markers                                   │
│                                                                          │
│   1. McSqueal reads MySQL binlog                                        │
│   2. Extracts changed rows                                               │
│   3. Sends invalidations to all Memcached servers                       │
│                                                                          │
│   Remote Markers (for cross-region):                                    │
│   - When data changes in Region A                                        │
│   - Set a "marker" in Region B's Memcached                              │
│   - Marker says: "This data is being updated, refresh from DB"          │
│   - Prevents returning stale data during replication lag               │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │   Region A (Primary)           Region B (Replica)             │     │
│   │   ┌────────────────┐          ┌────────────────┐             │     │
│   │   │    MySQL       │          │    MySQL       │             │     │
│   │   │   (Primary)    │─────────▶│   (Replica)    │             │     │
│   │   └────────┬───────┘          └────────────────┘             │     │
│   │            │ binlog                    ▲                      │     │
│   │            ▼                           │                      │     │
│   │   ┌────────────────┐                   │                      │     │
│   │   │   McSqueal     │───────────────────┼─────────────────┐   │     │
│   │   └────────┬───────┘    Set remote     │                 │   │     │
│   │            │            marker         │                 │   │     │
│   │            ▼                           │                 ▼   │     │
│   │   ┌────────────────┐          ┌────────────────────────────┐│     │
│   │   │   Memcached A  │          │   Memcached B              ││     │
│   │   │   (invalidated)│          │   (marker: refresh from DB)││     │
│   │   └────────────────┘          └────────────────────────────┘│     │
│   └───────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Netflix's Approach

Netflix uses EVCache with a custom coherency solution:

1. **Zone Affinity**: Requests prefer caches in the same availability zone
2. **Replication**: Data replicated across zones asynchronously
3. **TTL-Based Coherency**: Short TTLs for frequently changing data
4. **Application-Level Versioning**: Include version in cached objects

---

## 6️⃣ How to Implement Cache Coherency in Java

### Invalidation-Based Coherency with Redis Pub/Sub

```java
// CoherentCache.java
package com.example.cache.coherency;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

/**
 * Cache with built-in coherency via Redis Pub/Sub
 */
@Service
@Slf4j
public class CoherentCache<V> {

    private final Cache<String, CacheEntry<V>> l1Cache;
    private final RedisTemplate<String, V> redisTemplate;
    private final CacheCoherencyBroadcaster broadcaster;

    private static final Duration L1_TTL = Duration.ofSeconds(30);
    private static final Duration L2_TTL = Duration.ofMinutes(5);

    public CoherentCache(RedisTemplate<String, V> redisTemplate,
                         CacheCoherencyBroadcaster broadcaster) {
        this.redisTemplate = redisTemplate;
        this.broadcaster = broadcaster;
        
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(L1_TTL)
            .build();
    }

    /**
     * Get value with coherent multi-level caching
     */
    public V get(String key, Supplier<V> loader) {
        // Check L1
        CacheEntry<V> entry = l1Cache.getIfPresent(key);
        if (entry != null) {
            log.debug("L1 HIT for {}", key);
            return entry.getValue();
        }

        // Check L2 (Redis)
        V value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            log.debug("L2 HIT for {}", key);
            l1Cache.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
            return value;
        }

        // Load from source
        log.debug("MISS for {}", key);
        value = loader.get();
        if (value != null) {
            put(key, value);
        }
        return value;
    }

    /**
     * Put value and broadcast for coherency
     */
    public void put(String key, V value) {
        long timestamp = System.currentTimeMillis();
        
        // Update L1
        l1Cache.put(key, new CacheEntry<>(value, timestamp));
        
        // Update L2
        redisTemplate.opsForValue().set(key, value, L2_TTL);
        
        // Broadcast update to other servers
        broadcaster.broadcastUpdate(key, timestamp);
    }

    /**
     * Invalidate and broadcast
     */
    public void invalidate(String key) {
        l1Cache.invalidate(key);
        redisTemplate.delete(key);
        broadcaster.broadcastInvalidation(key);
        log.debug("Invalidated {} and broadcast", key);
    }

    /**
     * Handle invalidation from another server
     */
    public void handleRemoteInvalidation(String key) {
        l1Cache.invalidate(key);
        log.debug("Handled remote invalidation for {}", key);
    }

    /**
     * Handle update notification from another server
     */
    public void handleRemoteUpdate(String key, long timestamp) {
        CacheEntry<V> local = l1Cache.getIfPresent(key);
        
        // Only invalidate if remote update is newer
        if (local == null || local.getTimestamp() < timestamp) {
            l1Cache.invalidate(key);
            log.debug("Invalidated {} due to newer remote update", key);
        }
    }

    /**
     * Cache entry with timestamp for conflict resolution
     */
    private record CacheEntry<V>(V value, long timestamp) {
        public V getValue() { return value; }
        public long getTimestamp() { return timestamp; }
    }
}
```

### Coherency Broadcaster

```java
// CacheCoherencyBroadcaster.java
package com.example.cache.coherency;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

/**
 * Broadcasts cache coherency messages via Redis Pub/Sub
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class CacheCoherencyBroadcaster {

    private final RedisTemplate<String, String> redisTemplate;
    
    private static final String INVALIDATION_CHANNEL = "cache:coherency:invalidate";
    private static final String UPDATE_CHANNEL = "cache:coherency:update";

    /**
     * Broadcast invalidation to all servers
     */
    public void broadcastInvalidation(String key) {
        redisTemplate.convertAndSend(INVALIDATION_CHANNEL, key);
        log.debug("Broadcast invalidation for {}", key);
    }

    /**
     * Broadcast update with timestamp for conflict resolution
     */
    public void broadcastUpdate(String key, long timestamp) {
        String message = key + ":" + timestamp;
        redisTemplate.convertAndSend(UPDATE_CHANNEL, message);
        log.debug("Broadcast update for {} at {}", key, timestamp);
    }
}
```

### Coherency Listener

```java
// CacheCoherencyListener.java
package com.example.cache.coherency;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.stereotype.Component;

/**
 * Listens for coherency messages and updates local cache
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheCoherencyListener implements MessageListener {

    private final CoherentCache<?> cache;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String body = new String(message.getBody());
        
        if (channel.endsWith(":invalidate")) {
            cache.handleRemoteInvalidation(body);
        } else if (channel.endsWith(":update")) {
            String[] parts = body.split(":");
            if (parts.length == 2) {
                String key = parts[0];
                long timestamp = Long.parseLong(parts[1]);
                cache.handleRemoteUpdate(key, timestamp);
            }
        }
    }
}
```

### Version-Based Coherency

```java
// VersionedCoherentCache.java
package com.example.cache.coherency;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Function;
import java.util.function.Supplier;

/**
 * Version-based cache coherency
 * 
 * Each cached value includes a version.
 * On read, optionally verify version against source.
 */
@Service
@Slf4j
public class VersionedCoherentCache<V extends Versioned> {

    private final Cache<String, V> l1Cache;
    private final RedisTemplate<String, V> redisTemplate;

    public VersionedCoherentCache(RedisTemplate<String, V> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(1))
            .build();
    }

    /**
     * Get with version verification
     * 
     * @param key Cache key
     * @param versionChecker Function to get current version from source
     * @param loader Function to load full value from source
     */
    public V getWithVersionCheck(String key, 
                                  Supplier<Long> versionChecker,
                                  Supplier<V> loader) {
        // Check L1
        V cached = l1Cache.getIfPresent(key);
        
        if (cached != null) {
            // Verify version
            Long currentVersion = versionChecker.get();
            
            if (currentVersion != null && currentVersion.equals(cached.getVersion())) {
                log.debug("L1 HIT with valid version for {}", key);
                return cached;
            } else {
                log.debug("L1 version mismatch for {} (cached: {}, current: {})", 
                         key, cached.getVersion(), currentVersion);
                l1Cache.invalidate(key);
            }
        }

        // Check L2
        cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            Long currentVersion = versionChecker.get();
            if (currentVersion != null && currentVersion.equals(cached.getVersion())) {
                log.debug("L2 HIT with valid version for {}", key);
                l1Cache.put(key, cached);
                return cached;
            } else {
                log.debug("L2 version mismatch for {}", key);
                redisTemplate.delete(key);
            }
        }

        // Load fresh
        log.debug("Loading fresh data for {}", key);
        V fresh = loader.get();
        if (fresh != null) {
            l1Cache.put(key, fresh);
            redisTemplate.opsForValue().set(key, fresh, Duration.ofMinutes(5));
        }
        return fresh;
    }

    /**
     * Get without version check (faster, eventual consistency)
     */
    public V get(String key, Supplier<V> loader) {
        V cached = l1Cache.getIfPresent(key);
        if (cached != null) return cached;

        cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            l1Cache.put(key, cached);
            return cached;
        }

        V fresh = loader.get();
        if (fresh != null) {
            l1Cache.put(key, fresh);
            redisTemplate.opsForValue().set(key, fresh, Duration.ofMinutes(5));
        }
        return fresh;
    }
}

/**
 * Interface for versioned entities
 */
interface Versioned {
    Long getVersion();
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Coherency Model Comparison

| Model | Consistency | Latency | Complexity | Use Case |
|-------|-------------|---------|------------|----------|
| Strong | Highest | Highest | High | Financial data |
| Eventual | Medium | Low | Medium | Most web apps |
| Read-Your-Writes | Good for users | Low | Medium | User-facing apps |
| Version-Based | High | Medium | Medium | Critical but not real-time |

### Common Mistakes

**1. Not Handling Broadcast Failures**

```java
// WRONG: Assumes broadcast always succeeds
public void update(String key, V value) {
    database.save(key, value);
    cache.put(key, value);
    broadcaster.broadcast(key);  // What if this fails?
}

// RIGHT: Use TTL as safety net
public void update(String key, V value) {
    database.save(key, value);
    cache.put(key, value, Duration.ofMinutes(1));  // Short TTL
    try {
        broadcaster.broadcast(key);
    } catch (Exception e) {
        log.error("Broadcast failed, relying on TTL for coherency", e);
    }
}
```

**2. Ignoring Message Ordering**

```java
// WRONG: Process messages without checking order
public void handleUpdate(String key, V value) {
    cache.put(key, value);  // Might overwrite newer value!
}

// RIGHT: Use timestamps or versions
public void handleUpdate(String key, V value, long timestamp) {
    V current = cache.get(key);
    if (current == null || current.getTimestamp() < timestamp) {
        cache.put(key, value);
    }
}
```

**3. Thundering Herd on Invalidation**

```java
// WRONG: All servers query DB after invalidation
public void handleInvalidation(String key) {
    cache.invalidate(key);
    // Next 1000 requests all hit database!
}

// RIGHT: Use locking or coalescing
public void handleInvalidation(String key) {
    cache.invalidate(key);
    // On next request, use stampede prevention
}
```

---

## 8️⃣ When NOT to Use Strong Coherency

- **Read-heavy, write-light workloads**: Eventual consistency is sufficient
- **Non-critical data**: User preferences, recommendations
- **High-latency tolerance**: Background jobs, analytics
- **Cost-sensitive**: Strong coherency requires more infrastructure

---

## 9️⃣ Interview Follow-Up Questions WITH Answers

### L4 Questions

**Q: What is cache coherency?**

A: Cache coherency ensures that multiple caches show a consistent view of data. When data changes in the source (database), all caches should either have the updated value or not have the value at all. Without coherency, different users might see different values for the same data, leading to confusion and bugs.

### L5 Questions

**Q: How would you implement cache coherency in a distributed system?**

A: I'd use invalidation-based coherency with pub/sub messaging. When data changes, the updating server publishes an invalidation message. All other servers subscribe and invalidate their local caches. For conflict resolution, I'd include timestamps with messages. Short TTLs on local caches act as a safety net if messages are lost. For critical data, I might add version checking where the cache verifies the version against the database before returning data.

### L6 Questions

**Q: Design a cache coherency system for a global social network with users in multiple regions.**

A: I'd design a hierarchical system: (1) Local caches (L1) on each server with 30-second TTL. (2) Regional Redis cluster (L2) per region. (3) Primary database in one region, replicas in others. For coherency: writes go to primary region, which broadcasts invalidations globally via Kafka. Regional L2 caches receive invalidations within 100ms. Local L1 invalidations follow. For cross-region reads, I'd use "stale read" markers during replication lag. Users always read from their region for low latency, accepting that they might see slightly stale data (seconds) during writes. For critical operations (like changing privacy settings), I'd use synchronous cross-region invalidation.

---

## 🔟 One Clean Mental Summary

Cache coherency ensures all caches show consistent data. Invalidation-based protocols delete cache entries when data changes, forcing fresh fetches. Update-based protocols push new values to all caches. Version-based protocols verify cache freshness against the source. In distributed systems, use pub/sub messaging (Redis, Kafka) to broadcast invalidation messages. Include timestamps for conflict resolution. Use short TTLs as a safety net when messages fail. Choose your coherency model based on consistency requirements: strong for financial data, eventual for most web apps.

