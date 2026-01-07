# 🗃️ LRU Cache - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the LRU Cache.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Node` | Hold key-value and list pointers | Node structure changes |
| `LRUCache` | Manage cache with LRU policy | Cache policy changes |
| `ConcurrentLRUCache` | Add thread safety | Concurrency strategy changes |
| `EvictionListener` | Define eviction callback contract | Callback signature changes |

**SRP in Action:**

```java
// Node ONLY stores data and links
public class Node<K, V> {
    K key;
    V value;
    Node<K, V> prev;
    Node<K, V> next;
}

// LRUCache handles caching logic
public class LRUCache<K, V> {
    // HashMap for O(1) lookup
    // LinkedList for O(1) reordering
    // Eviction logic
}

// ConcurrentLRUCache ONLY adds thread safety
public class ConcurrentLRUCache<K, V> {
    private final LRUCache<K, V> cache;  // Delegates to LRUCache
    private final ReentrantReadWriteLock lock;  // Adds locking
}
```

**Why this separation?**
- Node doesn't know about caching
- LRUCache doesn't handle threading
- Each class has one reason to change

---

### 2. Open/Closed Principle (OCP)

**Adding New Eviction Policies:**

```java
// Current: LRU (Least Recently Used)
// To add LFU (Least Frequently Used):

public class LFUCache<K, V> {
    private final Map<K, Node<K, V>> cache;
    private final Map<K, Integer> frequencies;
    private final Map<Integer, LinkedList<K>> frequencyBuckets;
    
    public V get(K key) {
        // Increment frequency
        // Move to next frequency bucket
    }
}
```

**Adding New Eviction Listeners:**

```java
// No changes to LRUCache needed!

// Logging listener
cache.addEvictionListener((key, value) -> 
    logger.info("Evicted: {} = {}", key, value));

// Persistence listener
cache.addEvictionListener((key, value) -> 
    database.save(key, value));

// Metrics listener
cache.addEvictionListener((key, value) -> 
    metrics.increment("cache.evictions"));
```

**Adding TTL (Time-To-Live) Support:**

```java
public class TTLNode<K, V> extends Node<K, V> {
    private final long expirationTime;
    
    public boolean isExpired() {
        return System.currentTimeMillis() > expirationTime;
    }
}

public class TTLLRUCache<K, V> extends LRUCache<K, V> {
    @Override
    public V get(K key) {
        TTLNode<K, V> node = (TTLNode<K, V>) cache.get(key);
        if (node != null && node.isExpired()) {
            remove(key);
            return null;
        }
        return super.get(key);
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**Testing LSP:**

```java
// Any cache implementation should work the same way
public void testCache(Map<String, Integer> cache) {
    cache.put("A", 1);
    cache.put("B", 2);
    
    assertEquals(1, cache.get("A"));
    assertEquals(2, cache.get("B"));
}

// All these should work:
testCache(new LRUCache<>(10));
testCache(new LinkedHashMapLRUCache<>(10));
testCache(Collections.synchronizedMap(new LRUCache<>(10)));
```

**LSP with LinkedHashMap:**

```java
// LinkedHashMapLRUCache extends LinkedHashMap
// It can be used anywhere a Map is expected

Map<String, Integer> cache = new LinkedHashMapLRUCache<>(100);
cache.put("key", 123);  // Works like any Map
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
// EvictionListener is minimal - one method
@FunctionalInterface
public interface EvictionListener<K, V> {
    void onEviction(K key, V value);
}
```

**If we needed more callbacks:**

```java
// BAD: Fat interface
public interface CacheListener<K, V> {
    void onEviction(K key, V value);
    void onPut(K key, V value);
    void onGet(K key, V value);
    void onRemove(K key, V value);
    void onClear();
}

// GOOD: Segregated interfaces
public interface EvictionListener<K, V> {
    void onEviction(K key, V value);
}

public interface AccessListener<K, V> {
    void onAccess(K key, V value);
}

public interface ModificationListener<K, V> {
    void onPut(K key, V value);
    void onRemove(K key, V value);
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// ConcurrentLRUCache depends on concrete LRUCache
public class ConcurrentLRUCache<K, V> {
    private final LRUCache<K, V> cache;  // Concrete dependency
}
```

**Better with DIP:**

```java
// Define abstraction
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    V remove(K key);
    int size();
    void clear();
}

// LRUCache implements Cache
public class LRUCache<K, V> implements Cache<K, V> { }

// ConcurrentCache wraps any Cache
public class ConcurrentCache<K, V> implements Cache<K, V> {
    private final Cache<K, V> delegate;  // Abstraction!
    private final ReadWriteLock lock;
    
    public ConcurrentCache(Cache<K, V> delegate) {
        this.delegate = delegate;
        this.lock = new ReentrantReadWriteLock();
    }
}

// Now can wrap any cache implementation
Cache<String, Integer> lruCache = new LRUCache<>(100);
Cache<String, Integer> lfuCache = new LFUCache<>(100);

Cache<String, Integer> concurrentLRU = new ConcurrentCache<>(lruCache);
Cache<String, Integer> concurrentLFU = new ConcurrentCache<>(lfuCache);
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. LRUCache manages cache operations, Node represents list node, EvictionListener handles callbacks. Clear separation of concerns. | N/A | - |
| **OCP** | PASS | System is open for extension (new eviction listeners, cache decorators like MeteredLRUCache) without modifying existing code. Template method pattern in Transaction base class enables this. | N/A | - |
| **LSP** | PASS | All cache implementations (LRUCache, ConcurrentLRUCache) properly implement the cache contract. Subclasses are substitutable. | N/A | - |
| **ISP** | PASS | EvictionListener interface is minimal and focused. Clients only implement what they need. No unused methods forced on clients. | N/A | - |
| **DIP** | PASS | Cache operations depend on EvictionListener interface (abstraction), not concrete implementations. DIP is well applied. | N/A | - |

---

## Design Patterns Used

### 1. Composite Pattern (Implicit)

**Where:** Doubly Linked List structure

```java
// Each Node can be part of a chain
public class Node<K, V> {
    Node<K, V> prev;
    Node<K, V> next;
}

// Operations traverse the chain
private void removeNode(Node<K, V> node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
}
```

---

### 2. Decorator Pattern

**Where:** ConcurrentLRUCache wraps LRUCache

```java
public class ConcurrentLRUCache<K, V> {
    private final LRUCache<K, V> cache;  // Wrapped object
    
    public V get(K key) {
        lock.lock();
        try {
            return cache.get(key);  // Delegate with added behavior
        } finally {
            lock.unlock();
        }
    }
}
```

**Benefits:**
- Adds thread safety without modifying LRUCache
- Can stack decorators (logging, metrics, etc.)

---

### 3. Observer Pattern

**Where:** Eviction listeners

```java
public class LRUCache<K, V> {
    private final List<EvictionListener<K, V>> evictionListeners;
    
    public void addEvictionListener(EvictionListener<K, V> listener) {
        evictionListeners.add(listener);
    }
    
    private void notifyEviction(K key, V value) {
        for (EvictionListener<K, V> listener : evictionListeners) {
            listener.onEviction(key, value);
        }
    }
}
```

---

### 4. Sentinel Pattern

**Where:** Dummy head and tail nodes

```java
// Without sentinels - complex edge cases
public void addToHead(Node node) {
    if (head == null) {
        head = tail = node;
    } else {
        node.next = head;
        head.prev = node;
        head = node;
    }
}

// With sentinels - uniform handling
public void addToHead(Node node) {
    node.prev = head;
    node.next = head.next;
    head.next.prev = node;
    head.next = node;
}
```

**Benefits:**
- No null checks
- Same code for all cases
- Cleaner, less error-prone

---

## Why HashMap + Doubly Linked List?

### Requirements Analysis

| Requirement | HashMap Only | LinkedList Only | HashMap + LinkedList |
|-------------|--------------|-----------------|---------------------|
| O(1) get | ✅ | ❌ O(n) | ✅ |
| O(1) put | ✅ | ✅ (at head) | ✅ |
| O(1) remove | ✅ | ❌ O(n) to find | ✅ |
| O(1) reorder | ❌ | ✅ | ✅ |
| Track access order | ❌ | ✅ | ✅ |

### Why Not Just HashMap?

```java
// HashMap alone can't track access order
Map<K, V> map = new HashMap<>();
map.get("A");  // No way to know this was accessed
map.get("B");  // Which was accessed more recently?
```

### Why Not Just LinkedList?

```java
// LinkedList alone has O(n) lookup
LinkedList<Entry<K, V>> list = new LinkedList<>();
// To find key "A", must scan entire list: O(n)
```

### Why Doubly Linked (Not Singly)?

```java
// Singly linked: Can't remove in O(1)
// To remove node, need reference to PREVIOUS node
// Singly linked requires O(n) traversal to find previous

// Doubly linked: O(1) removal
node.prev.next = node.next;
node.next.prev = node.prev;
```

---

## Why Alternatives Were Rejected

### Alternative 1: TreeMap for Ordering

```java
// TreeMap maintains sorted order, not access order
TreeMap<Long, V> cache = new TreeMap<>();  // Sorted by timestamp
```

**Why rejected:**
- O(log n) operations instead of O(1)
- Timestamp collisions
- Complex to update timestamps

### Alternative 2: PriorityQueue

```java
// PriorityQueue for eviction
PriorityQueue<Entry<K, V>> queue = new PriorityQueue<>(
    Comparator.comparingLong(Entry::getAccessTime));
```

**Why rejected:**
- O(log n) for reordering
- O(n) to find specific entry
- Complex to update priorities

### Alternative 3: Single Data Structure

```java
// Just use LinkedHashMap
Map<K, V> cache = new LinkedHashMap<>(capacity, 0.75f, true);
```

**When to use:**
- Simple use cases
- Don't need custom eviction callbacks
- Don't need fine-grained control

**When NOT to use:**
- Need eviction callbacks
- Need custom eviction logic
- Need to extend functionality

---

## Thread Safety Analysis

### Race Conditions Without Locking

```java
// Thread A: get("key")
// Thread B: put("key", newValue)

// Possible interleaving:
Thread A: node = cache.get("key")  // Gets old node
Thread B: cache.put("key", newValue)  // Creates new node
Thread B: moveToHead(newNode)  // Modifies list
Thread A: moveToHead(node)  // Corrupts list!
```

### Why ReadWriteLock?

```java
// Option 1: synchronized (simple but slow)
public synchronized V get(K key) { }
public synchronized void put(K key, V value) { }

// Option 2: ReadWriteLock (better for read-heavy)
// Multiple readers can access simultaneously
// Writers get exclusive access
```

### Gotcha: get() Needs Write Lock

```java
// WRONG: get() seems like a read operation
public V get(K key) {
    readLock.lock();  // Multiple threads can enter
    try {
        Node node = cache.get(key);
        moveToHead(node);  // PROBLEM: Modifies list!
        return node.value;
    } finally {
        readLock.unlock();
    }
}

// CORRECT: get() modifies list, needs write lock
public V get(K key) {
    writeLock.lock();  // Exclusive access
    try {
        Node node = cache.get(key);
        moveToHead(node);  // Safe
        return node.value;
    } finally {
        writeLock.unlock();
    }
}
```

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: What happens if eviction listener throws exception?

**Answer:**

```java
public class LRUCache<K, V> {
    private void evictLRU() {
        Node<K, V> lru = tail.prev;
        removeNode(lru);
        cache.remove(lru.key);
        
        // Safe eviction listener invocation
        for (EvictionListener<K, V> listener : evictionListeners) {
            try {
                listener.onEviction(lru.key, lru.value);
            } catch (Exception e) {
                // Log error but don't fail eviction
                System.err.println("Eviction listener error: " + e.getMessage());
            }
        }
    }
}
```

**Design decision:** Eviction should succeed even if listeners fail. Failures in listeners should not break cache operations.

---

### Q2: How would you implement getOrCompute()?

**Answer:**

```java
public class LRUCache<K, V> {
    
    public V getOrCompute(K key, Function<K, V> computeFunction) {
        V value = get(key);
        if (value != null) {
            return value;
        }
        
        // Compute value (may be expensive)
        V computedValue = computeFunction.apply(key);
        
        // Put in cache
        put(key, computedValue);
        
        return computedValue;
    }
}

// Usage:
String value = cache.getOrCompute("key", k -> expensiveComputation(k));
```

**Extension - with locking for concurrent access:**
```java
public V getOrCompute(K key, Function<K, V> computeFunction) {
    V value = get(key);
    if (value != null) {
        return value;
    }
    
    // Double-check locking pattern
    writeLock.lock();
    try {
        value = get(key);  // Check again
        if (value != null) {
            return value;
        }
        
        V computedValue = computeFunction.apply(key);
        put(key, computedValue);
        return computedValue;
    } finally {
        writeLock.unlock();
    }
}
```

---

### Q3: How would you add statistics/metrics?

**Answer:**

```java
public class MeteredLRUCache<K, V> extends LRUCache<K, V> {
    private final AtomicLong hits = new AtomicLong();
    private final AtomicLong misses = new AtomicLong();
    private final AtomicLong evictions = new AtomicLong();
    
    @Override
    public V get(K key) {
        V value = super.get(key);
        if (value != null) {
            hits.incrementAndGet();
        } else {
            misses.incrementAndGet();
        }
        return value;
    }
    
    @Override
    protected void evictLRU() {
        evictions.incrementAndGet();
        super.evictLRU();
    }
    
    public double getHitRate() {
        long total = hits.get() + misses.get();
        return total == 0 ? 0 : (double) hits.get() / total;
    }
    
    public CacheStats getStats() {
        return new CacheStats(hits.get(), misses.get(), evictions.get());
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add bulk operations** - putAll(), removeAll() for batch updates
2. **Add iteration** - forEach(), stream() for cache traversal
3. **Add soft/weak references** - For memory-sensitive caching scenarios
4. **Add size-based eviction** - Evict by memory size, not just entry count
5. **Add async eviction** - Non-blocking eviction callbacks
6. **Add persistence** - Write-through/write-behind to disk for durability
7. **Add TTL support** - Time-to-live expiration for cache entries
8. **Add cache warming** - Pre-populate cache with frequently accessed items

**Detailed Cache Warming Implementation:**

```java
// Strategy 1: Eager Warming (Startup)
public class LRUCacheWithWarming<K, V> extends LRUCache<K, V> {
    private final CacheWarmer<K, V> warmer;
    
    public LRUCacheWithWarming(int capacity, CacheWarmer<K, V> warmer) {
        super(capacity);
        this.warmer = warmer;
        warmCache();
    }
    
    private void warmCache() {
        Map<K, V> warmData = warmer.getWarmData();
        for (Map.Entry<K, V> entry : warmData.entrySet()) {
            if (size() < getCapacity()) {
                put(entry.getKey(), entry.getValue());
            }
        }
    }
    
    public interface CacheWarmer<K, V> {
        Map<K, V> getWarmData();
    }
}

// Strategy 2: Lazy Warming (Background)
public class LRUCacheWithLazyWarming<K, V> extends LRUCache<K, V> {
    private final ScheduledExecutorService scheduler;
    private final CacheWarmer<K, V> warmer;
    
    public LRUCacheWithLazyWarming(int capacity, CacheWarmer<K, V> warmer) {
        super(capacity);
        this.warmer = warmer;
        this.scheduler = Executors.newScheduledThreadPool(1);
        // Start warming 5 seconds after startup
        scheduler.schedule(this::warmCacheInBackground, 5, TimeUnit.SECONDS);
    }
    
    private void warmCacheInBackground() {
        scheduler.execute(() -> {
            Map<K, V> warmData = warmer.getWarmData();
            for (Map.Entry<K, V> entry : warmData.entrySet()) {
                if (size() < getCapacity()) {
                    put(entry.getKey(), entry.getValue());
                }
            }
        });
    }
}

// Hit Rate Optimization with Metrics
public class LRUCacheWithMetrics<K, V> extends LRUCache<K, V> {
    private final AtomicLong hits = new AtomicLong(0);
    private final AtomicLong misses = new AtomicLong(0);
    
    @Override
    public V get(K key) {
        V value = super.get(key);
        if (value != null) {
            hits.incrementAndGet();
        } else {
            misses.incrementAndGet();
        }
        return value;
    }
    
    public double getHitRate() {
        long total = hits.get() + misses.get();
        return total > 0 ? (double) hits.get() / total : 0.0;
    }
    
    public double getHitRatePercentage() {
        return getHitRate() * 100.0;
    }
}

// Real-world example: E-commerce product cache
CacheWarmer<String, Product> warmer = () -> {
    // Warm top 1000 products by sales + recent views
    List<Product> topProducts = productRepository.findTopProducts(1000);
    return topProducts.stream()
        .collect(Collectors.toMap(Product::getId, p -> p));
};
LRUCache<String, Product> cache = new LRUCacheWithWarming<>(10000, warmer);
```

**Best Practices:**
1. Identify hot data using analytics (top products, active users)
2. Warm gradually in batches (100 items at a time)
3. Monitor hit rate and adjust warming strategy
4. Refresh warm data periodically (every hour)
5. Handle warming failures gracefully (don't block cache operations)

---

### Q5: How would you optimize for very large capacities (millions of entries)?

**Answer:**

```java
// Option 1: Shard the cache
public class ShardedLRUCache<K, V> {
    private final LRUCache<K, V>[] shards;
    
    public ShardedLRUCache(int capacity, int numShards) {
        this.shards = new LRUCache[numShards];
        int shardCapacity = capacity / numShards;
        for (int i = 0; i < numShards; i++) {
            shards[i] = new LRUCache<>(shardCapacity);
        }
    }
    
    private LRUCache<K, V> getShard(K key) {
        int shardIndex = Math.abs(key.hashCode() % shards.length);
        return shards[shardIndex];
    }
    
    public V get(K key) {
        return getShard(key).get(key);
    }
    
    public void put(K key, V value) {
        getShard(key).put(key, value);
    }
}
```

**Benefits:**
- Reduces lock contention (each shard has own lock)
- Better parallel access
- Smaller individual cache structures

---

### Q6: How would you implement cache expiration (TTL)?

**Answer:**

```java
public class ExpiringLRUCache<K, V> extends LRUCache<K, V> {
    private final Map<K, Long> expirationTimes;
    private final long defaultTTL;
    
    public ExpiringLRUCache(int capacity, long defaultTTLMs) {
        super(capacity);
        this.defaultTTL = defaultTTLMs;
        this.expirationTimes = new ConcurrentHashMap<>();
        
        // Background thread to clean expired entries
        startExpirationCleaner();
    }
    
    @Override
    public V get(K key) {
        if (isExpired(key)) {
            remove(key);
            return null;
        }
        return super.get(key);
    }
    
    @Override
    public void put(K key, V value) {
        expirationTimes.put(key, System.currentTimeMillis() + defaultTTL);
        super.put(key, value);
    }
    
    private boolean isExpired(K key) {
        Long expiration = expirationTimes.get(key);
        return expiration != null && System.currentTimeMillis() > expiration;
    }
    
    private void startExpirationCleaner() {
        scheduler.scheduleAtFixedRate(() -> {
            expirationTimes.entrySet().removeIf(e -> 
                System.currentTimeMillis() > e.getValue());
        }, 1, 1, TimeUnit.SECONDS);
    }
}
```

---

### Q7: How would you handle cache persistence (survive restarts)?

**Answer:**

```java
public class PersistentLRUCache<K, V> extends LRUCache<K, V> {
    private final String persistenceFile;
    
    public PersistentLRUCache(int capacity, String persistenceFile) {
        super(capacity);
        this.persistenceFile = persistenceFile;
        loadFromDisk();
        
        // Shutdown hook to save on exit
        Runtime.getRuntime().addShutdownHook(new Thread(this::saveToDisk));
    }
    
    private void loadFromDisk() {
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream(persistenceFile))) {
            Map<K, V> data = (Map<K, V>) ois.readObject();
            data.forEach((k, v) -> super.put(k, v));
        } catch (IOException | ClassNotFoundException e) {
            // File doesn't exist or error - start fresh
        }
    }
    
    private void saveToDisk() {
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream(persistenceFile))) {
            Map<K, V> snapshot = new HashMap<>();
            // Serialize cache contents
            oos.writeObject(snapshot);
        } catch (IOException e) {
            System.err.println("Failed to persist cache: " + e.getMessage());
        }
    }
}
```

---

### Q8: How would you implement a cache with size-based eviction (by memory usage)?

**Answer:**

```java
public class SizeBasedLRUCache<K, V> {
    private final long maxMemoryBytes;
    private final Map<K, Node<K, V>> cache;
    private long currentMemoryBytes;
    
    public void put(K key, V value) {
        long entrySize = estimateSize(key, value);
        
        // Evict until we have space
        while (currentMemoryBytes + entrySize > maxMemoryBytes && !cache.isEmpty()) {
            evictLRU();
        }
        
        // Add new entry
        Node<K, V> node = new Node<>(key, value);
        cache.put(key, node);
        currentMemoryBytes += entrySize;
        moveToHead(node);
    }
    
    private long estimateSize(K key, V value) {
        // Rough estimation
        long keySize = key instanceof String ? ((String) key).length() * 2 : 8;
        long valueSize = value instanceof String ? ((String) value).length() * 2 : 8;
        return keySize + valueSize + 40;  // Node overhead
    }
}
```

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | HashMap | LinkedList | Combined |
|-----------|---------|------------|----------|
| get() | O(1) | O(1) move | O(1) |
| put() new | O(1) | O(1) add | O(1) |
| put() update | O(1) | O(1) move | O(1) |
| remove() | O(1) | O(1) | O(1) |
| evict() | O(1) | O(1) | O(1) |

### Space Complexity

| Component | Space |
|-----------|-------|
| HashMap | O(n) |
| Nodes | O(n) |
| Prev/Next pointers | O(n) |
| **Total** | **O(n)** |

Where n = capacity of cache

### Bottlenecks at Scale

**10x Usage (1K → 10K entries):**
- Problem: HashMap resizing becomes expensive (O(n) rehashing), memory overhead grows noticeably
- Solution: Pre-size HashMap with capacity/load factor, use object pooling for nodes
- Tradeoff: Slightly more memory upfront, faster operations

**100x Usage (1K → 100K entries):**
- Problem: Single read-write lock becomes contention bottleneck, memory usage may exceed single machine capacity
- Solution: Shard cache by key hash, distribute across multiple cache instances with consistent hashing
- Tradeoff: More complex architecture, need coordination between shards

### Memory Overhead

```java
// Per entry overhead:
// - HashMap.Entry: ~32 bytes
// - Node object: ~40 bytes (key, value, prev, next, object header)
// - Total: ~72 bytes + key size + value size

// For 10,000 entries with String keys (avg 20 chars) and Integer values:
// ~72 + 40 + 16 = 128 bytes per entry
// Total: ~1.28 MB
```


### Q1: How would you implement LFU (Least Frequently Used)?

```java
public class LFUCache<K, V> {
    private final Map<K, Node<K, V>> cache;
    private final Map<K, Integer> frequencies;
    private final Map<Integer, LinkedHashSet<K>> frequencyBuckets;
    private int minFrequency;
    
    public V get(K key) {
        if (!cache.containsKey(key)) return null;
        
        // Increment frequency
        int freq = frequencies.get(key);
        frequencies.put(key, freq + 1);
        
        // Move to next frequency bucket
        frequencyBuckets.get(freq).remove(key);
        frequencyBuckets.computeIfAbsent(freq + 1, k -> new LinkedHashSet<>())
                       .add(key);
        
        // Update min frequency if needed
        if (frequencyBuckets.get(minFrequency).isEmpty()) {
            minFrequency++;
        }
        
        return cache.get(key).value;
    }
}
```

### Q2: How would you add TTL (expiration)?

```java
public class TTLLRUCache<K, V> {
    private final LRUCache<K, TTLEntry<V>> cache;
    private final ScheduledExecutorService cleaner;
    
    static class TTLEntry<V> {
        V value;
        long expirationTime;
    }
    
    public V get(K key) {
        TTLEntry<V> entry = cache.get(key);
        if (entry == null) return null;
        
        if (System.currentTimeMillis() > entry.expirationTime) {
            cache.remove(key);
            return null;
        }
        
        return entry.value;
    }
    
    public void put(K key, V value, long ttlMillis) {
        TTLEntry<V> entry = new TTLEntry<>();
        entry.value = value;
        entry.expirationTime = System.currentTimeMillis() + ttlMillis;
        cache.put(key, entry);
    }
}
```

### Q3: How would you make it distributed?

```java
// Use Redis as distributed cache
public class DistributedLRUCache<K, V> {
    private final JedisPool jedisPool;
    private final String cacheName;
    private final int capacity;
    
    public V get(K key) {
        try (Jedis jedis = jedisPool.getResource()) {
            String value = jedis.get(cacheName + ":" + key);
            if (value != null) {
                // Update access time (for LRU)
                jedis.zadd(cacheName + ":access", 
                          System.currentTimeMillis(), key.toString());
            }
            return deserialize(value);
        }
    }
    
    public void put(K key, V value) {
        try (Jedis jedis = jedisPool.getResource()) {
            // Check capacity and evict if needed
            long size = jedis.zcard(cacheName + ":access");
            if (size >= capacity) {
                // Remove LRU (lowest score = oldest access time)
                Set<String> lru = jedis.zrange(cacheName + ":access", 0, 0);
                for (String lruKey : lru) {
                    jedis.del(cacheName + ":" + lruKey);
                    jedis.zrem(cacheName + ":access", lruKey);
                }
            }
            
            // Add new entry
            jedis.set(cacheName + ":" + key, serialize(value));
            jedis.zadd(cacheName + ":access", 
                      System.currentTimeMillis(), key.toString());
        }
    }
}
```

