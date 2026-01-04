# Distributed Cache - Production Deep Dives (Core)

## Overview

This document covers the core production components: consistent hashing with virtual nodes, cache invalidation strategies, eviction policies, and cache stampede prevention.

---

## 1. Consistent Hashing

### A) CONCEPT: What is Consistent Hashing?

Consistent hashing is a distributed hashing scheme that minimizes remapping when nodes are added or removed. It uses a hash ring where:

1. **Hash Ring**: Circular space (0 to 2^64-1)
2. **Node Placement**: Nodes placed on ring by hash(node_id)
3. **Key Placement**: Keys placed on ring by hash(key)
4. **Key Assignment**: Key assigned to first node clockwise

**Example:**
```
Hash Ring: [0 -------- 2^64-1]
Nodes:     [Node1, Node2, Node3]
Keys:      [key1, key2, key3]

key1 → Node1 (first node clockwise)
key2 → Node2
key3 → Node3
```

### B) OUR USAGE: Virtual Nodes Implementation

**Problem:** Without virtual nodes, uneven distribution

**Example:**
```
3 physical nodes
Hash ring: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

Node1 at position 0
Node2 at position 3
Node3 at position 6

Keys distributed:
  [0,1,2] → Node1 (3 keys)
  [3,4,5] → Node2 (3 keys)
  [6,7,8,9] → Node3 (4 keys)  ← Uneven!
```

**Solution: Virtual Nodes**

```java
public class ConsistentHash {
    
    private final TreeMap<Long, String> ring = new TreeMap<>();
    private final int virtualNodesPerPhysical = 150;
    
    public void addNode(String physicalNode) {
        for (int i = 0; i < virtualNodesPerPhysical; i++) {
            String virtualNode = physicalNode + ":" + i;
            long hash = hash(virtualNode);
            ring.put(hash, physicalNode);
        }
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        long keyHash = hash(key);
        Map.Entry<Long, String> entry = ring.ceilingEntry(keyHash);
        
        if (entry == null) {
            // Wrap around to beginning
            entry = ring.firstEntry();
        }
        
        return entry.getValue();
    }
    
    private long hash(String input) {
        return Hashing.murmur3_128()
            .hashString(input, StandardCharsets.UTF_8)
            .asLong();
    }
}
```

**Benefits:**
- **Even Distribution**: 150 virtual nodes per physical = even distribution
- **Smooth Rebalancing**: Only 1/N keys remap when node added/removed
- **Load Balancing**: Distributes load evenly

### C) REAL STEP-BY-STEP SIMULATION: Consistent Hashing Technology-Level

**Normal Flow: Key Lookup**

```
Step 1: Client Requests Key
┌─────────────────────────────────────────────────────────────┐
│ GET cache/user:123                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Hash Key
┌─────────────────────────────────────────────────────────────┐
│ Hash("user:123") = 1234567890123456789                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Find Node on Ring
┌─────────────────────────────────────────────────────────────┐
│ Hash Ring:                                                  │
│   Node1:virtual0 → hash: 1000000000000000000               │
│   Node1:virtual1 → hash: 2000000000000000000               │
│   Node2:virtual0 → hash: 5000000000000000000               │
│   Node2:virtual1 → hash: 6000000000000000000               │
│   Node3:virtual0 → hash: 9000000000000000000               │
│                                                              │
│ Key hash: 1234567890123456789                               │
│ Find first node >= key hash: Node1:virtual1 (2000000...)     │
│ Physical node: Node1                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Connect to Node
┌─────────────────────────────────────────────────────────────┐
│ Connect to Node1                                            │
│ Redis: GET user:123                                         │
│ Result: HIT, value: { "name": "John", "email": "..." }     │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
```

**Node Failure Flow:**

```
Step 1: Node2 Fails
┌─────────────────────────────────────────────────────────────┐
│ Health check detects Node2 failure                          │
│ Alert: Node2 down                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Remove Node from Ring
┌─────────────────────────────────────────────────────────────┐
│ Remove Node2 virtual nodes from ring:                        │
│   - Node2:virtual0 (hash: 5000000...)                        │
│   - Node2:virtual1 (hash: 6000000...)                        │
│   - ... (150 virtual nodes removed)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Reassign Keys
┌─────────────────────────────────────────────────────────────┐
│ Keys previously on Node2 reassigned:                        │
│   - key1 (hash: 5500000...) → Node3 (next clockwise)        │
│   - key2 (hash: 6200000...) → Node3                        │
│   - ... (1/3 of keys remapped)                             │
│                                                              │
│ Impact: Only 1/N keys need remapping (N = number of nodes)  │
│ Minimal disruption                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Cache Invalidation

### A) CONCEPT: What is Cache Invalidation?

Cache invalidation removes stale data from cache when source data changes. This ensures cache consistency.

**Strategies:**
1. **TTL-based**: Automatic expiration
2. **Manual**: Explicit invalidation
3. **Write-through**: Invalidate on write
4. **Write-behind**: Invalidate after write

### B) OUR USAGE: Multi-Strategy Invalidation

**TTL-Based Expiration:**

```java
@Service
public class CacheService {
    
    private final RedisTemplate<String, Object> redis;
    
    public void set(String key, Object value, int ttlSeconds) {
        CacheEntry entry = new CacheEntry(
            value, 
            System.currentTimeMillis() + ttlSeconds * 1000
        );
        redis.opsForValue().set(key, entry, Duration.ofSeconds(ttlSeconds));
    }
    
    public Object get(String key) {
        CacheEntry entry = (CacheEntry) redis.opsForValue().get(key);
        if (entry == null) {
            return null;  // Cache miss
        }
        
        if (entry.getExpiresAt() < System.currentTimeMillis()) {
            redis.delete(key);  // Expired, remove
            return null;
        }
        
        return entry.getValue();
    }
}
```

**Manual Invalidation:**

```java
public void invalidate(String key) {
    // Invalidate in primary node
    redis.delete(key);
    
    // Invalidate in replicas (via pub/sub)
    redis.convertAndSend("cache:invalidate", key);
}

@EventListener
public void handleInvalidation(String key) {
    redis.delete(key);
}
```

**Pattern-Based Invalidation:**

```java
public void invalidatePattern(String pattern) {
    // Use SCAN to find matching keys (non-blocking)
    Set<String> keys = new HashSet<>();
    ScanOptions options = ScanOptions.scanOptions()
        .match(pattern)
        .count(100)
        .build();
    
    Cursor<String> cursor = redis.scan(options);
    while (cursor.hasNext()) {
        keys.add(cursor.next());
    }
    
    // Delete in batches
    if (!keys.isEmpty()) {
        redis.delete(keys.toArray(new String[0]));
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Invalidation Technology-Level

**Normal Flow: TTL Expiration**

```
Step 1: Key Set with TTL
┌─────────────────────────────────────────────────────────────┐
│ SET user:123 { "name": "John" } EX 3600                     │
│ TTL: 1 hour                                                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Key Accessed Before Expiration
┌─────────────────────────────────────────────────────────────┐
│ T+30min: GET user:123                                        │
│ Result: HIT, value: { "name": "John" }                      │
│ TTL remaining: 30 minutes                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Key Expires
┌─────────────────────────────────────────────────────────────┐
│ T+60min: GET user:123                                        │
│ Redis: Key expired, automatically removed                   │
│ Result: MISS                                                 │
│ Action: Fetch from database, cache with new TTL             │
└─────────────────────────────────────────────────────────────┘
```

**Manual Invalidation Flow:**

```
Step 1: Data Updated in Database
┌─────────────────────────────────────────────────────────────┐
│ Database: UPDATE users SET name='Jane' WHERE id=123        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Invalidate Cache
┌─────────────────────────────────────────────────────────────┐
│ Cache: DEL user:123                                         │
│ Cache: PUBLISH cache:invalidate user:123                    │
│ Replicas: Receive invalidation message, delete key           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Next Request
┌─────────────────────────────────────────────────────────────┐
│ GET user:123                                                 │
│ Result: MISS (invalidated)                                  │
│ Action: Fetch from database, cache new value                │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Eviction Policies

### A) CONCEPT: What are Eviction Policies?

Eviction policies determine which keys to remove when cache is full. Common policies:

1. **LRU (Least Recently Used)**: Evict least recently accessed
2. **LFU (Least Frequently Used)**: Evict least frequently accessed
3. **FIFO (First In First Out)**: Evict oldest
4. **Random**: Evict random key

### B) OUR USAGE: LRU Implementation

**Redis LRU Configuration:**

```conf
# redis.conf
maxmemory 32gb
maxmemory-policy allkeys-lru
```

**Java LRU Implementation:**

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public LRUCache(int maxSize) {
        super(16, 0.75f, true);  // accessOrder = true (LRU)
        this.maxSize = maxSize;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
    
    @Override
    public V get(Object key) {
        V value = super.get(key);
        if (value != null) {
            // Move to end (most recently used)
            remove(key);
            put((K) key, value);
        }
        return value;
    }
}
```

**Why LRU?**
- Good for 80/20 access patterns (hot keys stay, cold keys evicted)
- Simple to implement
- Works well for most use cases

---

## 4. Cache Stampede Prevention

### A) CONCEPT: What is Cache Stampede?

Cache stampede (thundering herd) occurs when a cached value expires and many requests simultaneously try to recompute it, overwhelming the backend.

**Scenario:**
```
T+0s: Popular key expires
T+0s: 1,000 requests see cache MISS
T+0s: All 1,000 requests hit database simultaneously
T+1s: Database overwhelmed
```

### B) OUR USAGE: Distributed Locking + Single-Flight

**Implementation:**

```java
@Service
public class CacheStampedePrevention {
    
    private final RedisTemplate<String, Object> redis;
    private final DistributedLock distributedLock;
    
    public Object getOrCompute(String key, Supplier<Object> compute, int ttlSeconds) {
        // 1. Try cache first
        Object value = redis.opsForValue().get(key);
        if (value != null) {
            return value;
        }
        
        // 2. Try to acquire lock
        String lockKey = "lock:" + key;
        boolean acquired = distributedLock.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is computing, wait briefly
            try {
                Thread.sleep(50);
                value = redis.opsForValue().get(key);
                if (value != null) {
                    return value;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fallback: Compute anyway (better than blocking)
            return compute.get();
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            value = redis.opsForValue().get(key);
            if (value != null) {
                return value;
            }
            
            // 4. Compute value (only one thread does this)
            value = compute.get();
            
            // 5. Cache with TTL jitter (prevent simultaneous expiration)
            long baseTtl = ttlSeconds;
            long jitter = (long)(baseTtl * 0.1 * (Math.random() * 2 - 1));  // ±10%
            long finalTtl = baseTtl + jitter;
            
            redis.opsForValue().set(key, value, Duration.ofSeconds(finalTtl));
            
            return value;
            
        } finally {
            distributedLock.unlock(lockKey);
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Cache Stampede Prevention

**Normal Flow: Cache Hit**

```
Step 1: Request Arrives
┌─────────────────────────────────────────────────────────────┐
│ GET cache/user:123                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET user:123                                         │
│ Result: HIT, value: { "name": "John" }                      │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Cached Value
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK, { "name": "John" }                       │
│ Total latency: ~2ms                                         │
└─────────────────────────────────────────────────────────────┘
```

**Cache Stampede Prevention Flow:**

```
Step 1: Cache Expires
┌─────────────────────────────────────────────────────────────┐
│ T+0s: Key user:123 expires                                  │
│ T+0s: 1,000 requests see cache MISS                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Try to Acquire Lock
┌─────────────────────────────────────────────────────────────┐
│ Request 1: Try lock:user:123 → SUCCESS (acquired)          │
│ Request 2-1000: Try lock:user:123 → FAILED (locked)        │
│                                                              │
│ Requests 2-1000: Wait 50ms, retry cache check               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Request 1 Computes Value
┌─────────────────────────────────────────────────────────────┐
│ Request 1: Compute value from database                      │
│   Database: SELECT * FROM users WHERE id=123                 │
│   Latency: ~10ms                                             │
│   Value: { "name": "John" }                                 │
│                                                              │
│ Request 1: Cache value with TTL jitter                      │
│   Redis: SET user:123 {value} EX 3600±360 (jitter)          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Request 1 Releases Lock
┌─────────────────────────────────────────────────────────────┐
│ Request 1: Release lock:user:123                            │
│ Request 1: Return value                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Requests 2-1000 Get Cached Value
┌─────────────────────────────────────────────────────────────┐
│ Requests 2-1000: GET user:123                                │
│ Result: HIT (cached by Request 1)                          │
│ Latency: ~1ms per request                                    │
│                                                              │
│ Result: Only 1 database query instead of 1,000              │
│ Database not overwhelmed                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Consistent Hashing | Virtual nodes | 150 virtual nodes per physical |
| Cache Invalidation | TTL + Manual | TTL jitter (±10%) |
| Eviction Policy | LRU | Redis allkeys-lru |
| Stampede Prevention | Distributed locking | Single-flight pattern |
