# 🗑️ Cache Eviction Policies

---

## 0️⃣ Prerequisites

Before diving into cache eviction policies, you need to understand:

- **Cache**: A fast storage layer with limited capacity. Covered in Topics 1-3.
- **Memory (RAM)**: The physical storage where cache data lives. RAM is finite and expensive.
- **TTL (Time To Live)**: Automatic expiration of cache entries based on time. Covered in Topic 2.
- **Cache Hit/Miss**: Whether requested data is found in cache (hit) or not (miss).

If you understand that caches have limited space and we need to decide what to remove when full, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

You have a cache with 1GB of memory. Your application caches user profiles, and each profile is ~1KB. That means you can store about 1 million profiles.

**Day 1**: 100,000 users. All profiles fit in cache. Life is good.

**Day 100**: 5 million users. Only 1 million can fit. What do you do?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE CACHE CAPACITY PROBLEM                            │
│                                                                          │
│   Cache Capacity: 1,000,000 entries                                      │
│                                                                          │
│   Current State:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  user:1, user:2, user:3, ... user:1000000                       │   │
│   │                                                                  │   │
│   │  [████████████████████████████████████████████████████] 100%    │   │
│   │                                                                  │   │
│   │  CACHE IS FULL!                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   New Request: Cache user:1000001                                        │
│                                                                          │
│   Question: Which existing entry should be removed?                      │
│                                                                          │
│   Options:                                                               │
│   - Remove user:1 (oldest)?                                             │
│   - Remove user:500000 (random)?                                        │
│   - Remove the one accessed longest ago?                                │
│   - Remove the one accessed least frequently?                           │
│                                                                          │
│   The answer depends on your ACCESS PATTERNS                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### What Systems Looked Like Before Smart Eviction

**Naive Approach: Just Delete Everything**

```java
// WRONG: When cache is full, clear it all
if (cache.size() >= MAX_SIZE) {
    cache.clear();  // Nuclear option!
}
cache.put(key, value);
```

Problems:
- Cache hit rate drops to 0% after clearing
- All subsequent requests hit the database
- Database gets overwhelmed

**Slightly Better: Random Eviction**

```java
// Better but still not great
if (cache.size() >= MAX_SIZE) {
    String randomKey = cache.keySet().iterator().next();
    cache.remove(randomKey);
}
```

Problems:
- Might remove frequently accessed data
- No consideration of access patterns
- Unpredictable performance

### What Breaks Without Smart Eviction?

1. **Thrashing**: Constantly evicting and re-caching the same items
   - User A's profile evicted, then immediately requested again
   - Cache becomes useless

2. **Low Hit Rate**: Wrong items kept in cache
   - Keeping data that's never accessed again
   - Missing data that's accessed frequently

3. **Inconsistent Performance**: 
   - Sometimes fast (cache hit), sometimes slow (cache miss)
   - Hard to predict and plan for

### Real Examples

**Facebook**: Uses LRU-based eviction for Memcached. Their cache hit rate is >99%. Without smart eviction, they'd need 100x more database capacity.

**Netflix**: Uses a combination of LRU and LFU for different data types. Session data uses LRU (recent matters), recommendation data uses LFU (popular matters).

**CPU Caches**: Your computer's L1/L2/L3 caches use sophisticated eviction policies. A bad eviction policy can slow your program by 100x.

---

## 2️⃣ Intuition and Mental Model

### The Bookshelf Analogy

Imagine you have a small bookshelf that holds 10 books, but you own 100 books. You need to decide which books to keep on the shelf.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE BOOKSHELF PROBLEM                                 │
│                                                                          │
│   Your Bookshelf (Cache): Holds 10 books                                │
│   Your Collection (Database): 100 books                                  │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  📚 📚 📚 📚 📚 📚 📚 📚 📚 📚                                   │   │
│   │  1  2  3  4  5  6  7  8  9  10                                   │   │
│   │                                                                  │   │
│   │  Shelf is FULL. You want to add book #11.                       │   │
│   │  Which book do you remove?                                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   EVICTION STRATEGIES:                                                   │
│                                                                          │
│   📖 FIFO (First In First Out):                                         │
│      "Remove the book that's been on the shelf longest"                 │
│      Like a queue at a store - first to arrive, first to leave          │
│                                                                          │
│   📖 LRU (Least Recently Used):                                         │
│      "Remove the book you haven't read in the longest time"             │
│      If you read book #3 yesterday but haven't touched #7 in months,    │
│      remove #7                                                           │
│                                                                          │
│   📖 LFU (Least Frequently Used):                                       │
│      "Remove the book you've read the fewest times"                     │
│      If you've read book #5 twenty times but #8 only once,              │
│      remove #8                                                           │
│                                                                          │
│   📖 Random:                                                             │
│      "Close your eyes and pick one to remove"                           │
│      Simple but might remove your favorite book                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Different strategies work better for different reading patterns:
- If you tend to re-read recent books: **LRU** is best
- If you have favorite books you read repeatedly: **LFU** is best
- If you read books in order and rarely re-read: **FIFO** is fine

---

## 3️⃣ How It Works Internally

### Overview of Eviction Policies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVICTION POLICIES COMPARISON                          │
│                                                                          │
│   Policy    │ Evicts           │ Best For              │ Overhead        │
│   ──────────┼──────────────────┼───────────────────────┼───────────────  │
│   FIFO      │ Oldest added     │ Sequential access     │ Low             │
│   LRU       │ Oldest accessed  │ Temporal locality     │ Medium          │
│   LFU       │ Least accessed   │ Frequency patterns    │ High            │
│   Random    │ Random item      │ Uniform access        │ Very Low        │
│   TTL       │ Expired items    │ Time-sensitive data   │ Low             │
│   ARC       │ Adaptive         │ Mixed workloads       │ High            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Policy 1: FIFO (First In First Out)

**The Concept**: Remove the item that was added to the cache earliest, regardless of how often or recently it was accessed.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FIFO EVICTION                                         │
│                                                                          │
│   Implementation: Queue (linked list)                                    │
│                                                                          │
│   Cache State (capacity = 4):                                           │
│   ┌─────┬─────┬─────┬─────┐                                             │
│   │  A  │  B  │  C  │  D  │  ← Newest                                   │
│   └─────┴─────┴─────┴─────┘                                             │
│      ↑                                                                   │
│   Oldest (will be evicted next)                                         │
│                                                                          │
│   Operation: Add E                                                       │
│   ─────────────────────────────────────────────────────────────────────  │
│   1. Cache is full                                                       │
│   2. Remove A (oldest)                                                   │
│   3. Add E at the end                                                    │
│                                                                          │
│   Result:                                                                │
│   ┌─────┬─────┬─────┬─────┐                                             │
│   │  B  │  C  │  D  │  E  │  ← Newest                                   │
│   └─────┴─────┴─────┴─────┘                                             │
│      ↑                                                                   │
│   Oldest                                                                 │
│                                                                          │
│   Time Complexity: O(1) for all operations                              │
│   Space Overhead: O(1) extra (just queue pointers)                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**When FIFO Works Well**:
- Data is accessed in order (like processing a stream)
- Access patterns are uniform
- Simplicity is more important than optimization

**When FIFO Fails**:
- Frequently accessed items might be evicted just because they're old
- No consideration of actual usage patterns

---

### Policy 2: LRU (Least Recently Used)

**The Concept**: Remove the item that hasn't been accessed for the longest time. Assumes recently accessed items are likely to be accessed again.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LRU EVICTION                                          │
│                                                                          │
│   Implementation: HashMap + Doubly Linked List                          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  HashMap: O(1) lookup                                            │   │
│   │  ┌─────────────────────────────────────────────────────────┐    │   │
│   │  │  A → Node1    B → Node2    C → Node3    D → Node4       │    │   │
│   │  └─────────────────────────────────────────────────────────┘    │   │
│   │                                                                  │   │
│   │  Doubly Linked List: O(1) move to front                         │   │
│   │  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐                  │   │
│   │  │ HEAD │◀──▶│  D   │◀──▶│  C   │◀──▶│  B   │◀──▶│  A   │◀──▶ TAIL
│   │  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘       │   │
│   │              Most Recent              Least Recent               │   │
│   │                                       (Evict this)               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Operation: Access C                                                    │
│   ─────────────────────────────────────────────────────────────────────  │
│   1. Find C in HashMap: O(1)                                            │
│   2. Remove C from current position in list: O(1)                       │
│   3. Insert C at head of list: O(1)                                     │
│                                                                          │
│   Result:                                                                │
│   HEAD ◀──▶ C ◀──▶ D ◀──▶ B ◀──▶ A ◀──▶ TAIL                          │
│             ↑                      ↑                                     │
│         Most Recent           Least Recent                               │
│                                                                          │
│   Time Complexity: O(1) for get, put, evict                             │
│   Space Overhead: O(n) for pointers                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why LRU Works**:
- **Temporal Locality**: Recently accessed data is likely to be accessed again soon
- Most real-world workloads exhibit this pattern
- Good balance between effectiveness and complexity

**When LRU Fails**:
- **Scan Resistance**: A one-time scan of many items evicts frequently-used items
- Example: Backup process reads all data once, evicting the hot cache

---

### Policy 3: LFU (Least Frequently Used)

**The Concept**: Remove the item that has been accessed the fewest times. Keeps popular items in cache.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LFU EVICTION                                          │
│                                                                          │
│   Implementation: HashMap + Frequency Buckets                           │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Frequency Buckets:                                              │   │
│   │                                                                  │   │
│   │  Freq 1: [E, F]     ← Evict from here first                     │   │
│   │  Freq 2: [C, D]                                                  │   │
│   │  Freq 5: [B]                                                     │   │
│   │  Freq 10: [A]       ← Most frequently accessed                  │   │
│   │                                                                  │   │
│   │  Access Counts:                                                  │   │
│   │  A: 10 times                                                     │   │
│   │  B: 5 times                                                      │   │
│   │  C: 2 times                                                      │   │
│   │  D: 2 times                                                      │   │
│   │  E: 1 time                                                       │   │
│   │  F: 1 time                                                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Eviction: Remove E or F (frequency = 1)                               │
│             If tie, use LRU within the frequency bucket                 │
│                                                                          │
│   Time Complexity: O(1) with proper implementation                      │
│   Space Overhead: O(n) for frequency tracking                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why LFU Works**:
- Keeps truly popular items in cache
- Resistant to scans (one-time access doesn't increase frequency much)

**When LFU Fails**:
- **Cache Pollution**: Old popular items stay forever even if no longer relevant
- **Cold Start**: New items have low frequency, get evicted quickly
- Example: Yesterday's trending topic stays cached, today's trending topic gets evicted

**Solution: LFU with Aging**
- Decay frequency counts over time
- Or use a time window for frequency counting

---

### Policy 4: Random Eviction

**The Concept**: When cache is full, randomly select an item to evict.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RANDOM EVICTION                                       │
│                                                                          │
│   Implementation: Just pick randomly                                     │
│                                                                          │
│   Cache: [A, B, C, D, E]                                                │
│                                                                          │
│   Eviction:                                                              │
│   1. Generate random index: rand() % size = 2                           │
│   2. Remove item at index 2: C                                          │
│                                                                          │
│   Time Complexity: O(1)                                                  │
│   Space Overhead: O(0) - no extra tracking needed!                      │
│                                                                          │
│   Surprisingly effective!                                                │
│   - For uniform access patterns, performs close to optimal              │
│   - Very low overhead                                                    │
│   - Used in CPU caches (approximated LRU via random sampling)           │
└─────────────────────────────────────────────────────────────────────────┘
```

**When Random Works**:
- Access patterns are uniform
- Overhead of tracking is too expensive
- As an approximation of other policies

---

### Policy 5: TTL-Based Eviction

**The Concept**: Items expire after a set time, regardless of access patterns.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TTL-BASED EVICTION                                    │
│                                                                          │
│   Each item has an expiration timestamp                                  │
│                                                                          │
│   Cache State at 10:00:00:                                              │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │  Key  │  Value  │  Expires At                                   │    │
│   │───────┼─────────┼──────────────────────────────────────────────│    │
│   │  A    │  ...    │  10:05:00  (5 min TTL)                       │    │
│   │  B    │  ...    │  10:02:00  (2 min TTL)  ← Expires soon       │    │
│   │  C    │  ...    │  10:10:00  (10 min TTL)                      │    │
│   │  D    │  ...    │  09:58:00  ← ALREADY EXPIRED!                │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│   Eviction Strategies:                                                   │
│                                                                          │
│   1. Lazy Expiration:                                                    │
│      - Check TTL on access                                               │
│      - If expired, delete and return miss                               │
│      - Pro: No background work                                           │
│      - Con: Expired items consume memory until accessed                  │
│                                                                          │
│   2. Active Expiration:                                                  │
│      - Background thread scans for expired items                        │
│      - Periodically removes them                                         │
│      - Pro: Memory freed promptly                                        │
│      - Con: CPU overhead for scanning                                    │
│                                                                          │
│   3. Hybrid (Redis approach):                                            │
│      - Lazy expiration on access                                         │
│      - Periodic sampling: randomly check 20 keys, delete expired        │
│      - If >25% expired, repeat immediately                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Policy 6: ARC (Adaptive Replacement Cache)

**The Concept**: Automatically balances between LRU and LFU based on workload. Maintains "ghost" entries to track recently evicted items.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARC (ADAPTIVE REPLACEMENT CACHE)                      │
│                                                                          │
│   Four Lists:                                                            │
│                                                                          │
│   T1: Recently accessed ONCE (LRU order)                                │
│   T2: Recently accessed MORE THAN ONCE (LRU order)                      │
│   B1: Ghost entries evicted from T1 (just keys, no values)              │
│   B2: Ghost entries evicted from T2 (just keys, no values)              │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   T1 (Recency)              T2 (Frequency)                      │   │
│   │   ┌─────────────┐           ┌─────────────┐                     │   │
│   │   │ A, B, C     │           │ X, Y, Z     │                     │   │
│   │   └─────────────┘           └─────────────┘                     │   │
│   │                                                                  │   │
│   │   B1 (Ghosts of T1)         B2 (Ghosts of T2)                   │   │
│   │   ┌─────────────┐           ┌─────────────┐                     │   │
│   │   │ D, E        │           │ W           │                     │   │
│   │   └─────────────┘           └─────────────┘                     │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Adaptation:                                                            │
│   - If we hit in B1: "We should have kept this!" → Increase T1 size    │
│   - If we hit in B2: "We should have kept this!" → Increase T2 size    │
│                                                                          │
│   This automatically adapts to workload:                                 │
│   - Scan-heavy workload → T1 grows (LRU behavior)                       │
│   - Frequency-heavy workload → T2 grows (LFU behavior)                  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why ARC is Powerful**:
- Self-tuning: No need to choose between LRU and LFU
- Scan-resistant: One-time accesses don't pollute T2
- Adapts to changing workloads

**Drawback**:
- More complex implementation
- Higher memory overhead (ghost entries)
- Patented by IBM (check licensing)

---

## 4️⃣ Simulation-First Explanation

Let's trace through different policies with the same access pattern.

### Scenario: Cache Size = 3, Access Pattern: A, B, C, A, D, A, E

**FIFO**:
```
Access A: Cache [A]           - Miss, add A
Access B: Cache [A, B]        - Miss, add B
Access C: Cache [A, B, C]     - Miss, add C (full)
Access A: Cache [A, B, C]     - HIT (A in cache)
Access D: Cache [B, C, D]     - Miss, evict A (oldest), add D
Access A: Cache [C, D, A]     - Miss, evict B (oldest), add A
Access E: Cache [D, A, E]     - Miss, evict C (oldest), add E

Hits: 1, Misses: 6, Hit Rate: 14%
```

**LRU**:
```
Access A: Cache [A]           - Miss, add A
Access B: Cache [A, B]        - Miss, add B
Access C: Cache [A, B, C]     - Miss, add C (full)
Access A: Cache [B, C, A]     - HIT, move A to front
Access D: Cache [C, A, D]     - Miss, evict B (LRU), add D
Access A: Cache [C, D, A]     - HIT, move A to front
Access E: Cache [D, A, E]     - Miss, evict C (LRU), add E

Hits: 2, Misses: 5, Hit Rate: 29%
```

**LFU**:
```
Access A: Cache [A(1)]              - Miss, add A with count 1
Access B: Cache [A(1), B(1)]        - Miss, add B with count 1
Access C: Cache [A(1), B(1), C(1)]  - Miss, add C (full)
Access A: Cache [A(2), B(1), C(1)]  - HIT, increment A's count
Access D: Cache [A(2), C(1), D(1)]  - Miss, evict B (LFU, tie-break oldest), add D
Access A: Cache [A(3), C(1), D(1)]  - HIT, increment A's count
Access E: Cache [A(3), D(1), E(1)]  - Miss, evict C (LFU, tie-break oldest), add E

Hits: 2, Misses: 5, Hit Rate: 29%
```

**Observation**: LRU and LFU both keep A in cache because it's accessed frequently. FIFO evicts A just because it was added first, even though it's still being used.

---

## 5️⃣ How Engineers Actually Use This in Production

### Redis Eviction Policies

Redis supports multiple eviction policies, configurable via `maxmemory-policy`:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REDIS EVICTION POLICIES                               │
│                                                                          │
│   Policy              │ Description                                      │
│   ────────────────────┼────────────────────────────────────────────────  │
│   noeviction          │ Return error when memory full                   │
│   allkeys-lru         │ LRU among ALL keys                              │
│   volatile-lru        │ LRU among keys WITH EXPIRE set                  │
│   allkeys-lfu         │ LFU among ALL keys (Redis 4.0+)                 │
│   volatile-lfu        │ LFU among keys WITH EXPIRE set                  │
│   allkeys-random      │ Random eviction among ALL keys                  │
│   volatile-random     │ Random among keys WITH EXPIRE set               │
│   volatile-ttl        │ Evict keys with shortest TTL                    │
│                                                                          │
│   Recommended:                                                           │
│   - General caching: allkeys-lru                                        │
│   - Mixed cache/persistent: volatile-lru                                │
│   - Frequency matters: allkeys-lfu                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Caffeine (Java Local Cache) Policies

```java
// Caffeine supports multiple eviction strategies
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)           // Size-based eviction
    .expireAfterWrite(Duration.ofMinutes(5))  // TTL
    .expireAfterAccess(Duration.ofMinutes(1)) // Idle timeout
    .build();

// Caffeine uses Window TinyLFU by default
// - Combines recency and frequency
// - Admission filter to prevent cache pollution
// - Near-optimal hit rates
```

### Real-World Configuration Examples

**Netflix's EVCache**:
```
# Uses LRU with TTL
maxmemory-policy: allkeys-lru
maxmemory: 4gb

# Different TTLs for different data types
session_ttl: 24h
recommendation_ttl: 1h
user_profile_ttl: 30m
```

**Facebook's TAO Cache**:
```
# Uses LRU with multiple cache tiers
L1 (local): 1GB, LRU, 1 minute TTL
L2 (regional): 100GB, LRU, 1 hour TTL
Database: Source of truth
```

---

## 6️⃣ How to Implement Eviction Policies in Java

### LRU Cache Implementation

```java
// LRUCache.java
package com.example.cache;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * LRU Cache using LinkedHashMap
 * 
 * LinkedHashMap with accessOrder=true maintains access order.
 * Override removeEldestEntry to evict when full.
 */
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    
    private final int capacity;

    /**
     * @param capacity Maximum number of entries
     */
    public LRUCache(int capacity) {
        // accessOrder=true: iteration order is access order (LRU)
        // accessOrder=false: iteration order is insertion order (FIFO)
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    /**
     * Called after every put/putAll.
     * Return true to remove the eldest entry.
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }

    // Usage example
    public static void main(String[] args) {
        LRUCache<String, String> cache = new LRUCache<>(3);
        
        cache.put("A", "1");
        cache.put("B", "2");
        cache.put("C", "3");
        System.out.println(cache.keySet());  // [A, B, C]
        
        cache.get("A");  // Access A, moves to end
        System.out.println(cache.keySet());  // [B, C, A]
        
        cache.put("D", "4");  // Evicts B (least recently used)
        System.out.println(cache.keySet());  // [C, A, D]
    }
}
```

### LFU Cache Implementation

```java
// LFUCache.java
package com.example.cache;

import java.util.*;

/**
 * LFU Cache with O(1) operations
 * 
 * Uses:
 * - HashMap for key -> value
 * - HashMap for key -> frequency
 * - HashMap for frequency -> LinkedHashSet of keys (maintains insertion order for tie-breaking)
 */
public class LFUCache<K, V> {
    
    private final int capacity;
    private int minFrequency;
    
    // Key -> Value
    private final Map<K, V> values;
    
    // Key -> Frequency
    private final Map<K, Integer> frequencies;
    
    // Frequency -> Keys with that frequency (LinkedHashSet for LRU tie-breaking)
    private final Map<Integer, LinkedHashSet<K>> frequencyBuckets;

    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.minFrequency = 0;
        this.values = new HashMap<>();
        this.frequencies = new HashMap<>();
        this.frequencyBuckets = new HashMap<>();
    }

    /**
     * Get value and update frequency
     */
    public V get(K key) {
        if (!values.containsKey(key)) {
            return null;
        }
        
        // Update frequency
        updateFrequency(key);
        
        return values.get(key);
    }

    /**
     * Put value, evicting if necessary
     */
    public void put(K key, V value) {
        if (capacity <= 0) return;
        
        // If key exists, update value and frequency
        if (values.containsKey(key)) {
            values.put(key, value);
            updateFrequency(key);
            return;
        }
        
        // If at capacity, evict
        if (values.size() >= capacity) {
            evict();
        }
        
        // Add new entry with frequency 1
        values.put(key, value);
        frequencies.put(key, 1);
        frequencyBuckets.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFrequency = 1;
    }

    /**
     * Update frequency of a key
     */
    private void updateFrequency(K key) {
        int freq = frequencies.get(key);
        
        // Remove from current frequency bucket
        frequencyBuckets.get(freq).remove(key);
        
        // If this was the min frequency bucket and it's now empty, increment min
        if (freq == minFrequency && frequencyBuckets.get(freq).isEmpty()) {
            minFrequency++;
        }
        
        // Add to new frequency bucket
        int newFreq = freq + 1;
        frequencies.put(key, newFreq);
        frequencyBuckets.computeIfAbsent(newFreq, k -> new LinkedHashSet<>()).add(key);
    }

    /**
     * Evict the least frequently used item
     * If tie, evict least recently used among them
     */
    private void evict() {
        LinkedHashSet<K> minFreqBucket = frequencyBuckets.get(minFrequency);
        
        // Get first item (oldest in this frequency - LRU tie-breaker)
        K keyToEvict = minFreqBucket.iterator().next();
        
        // Remove from all data structures
        minFreqBucket.remove(keyToEvict);
        values.remove(keyToEvict);
        frequencies.remove(keyToEvict);
    }

    public int size() {
        return values.size();
    }

    // Usage example
    public static void main(String[] args) {
        LFUCache<String, String> cache = new LFUCache<>(3);
        
        cache.put("A", "1");
        cache.put("B", "2");
        cache.put("C", "3");
        
        cache.get("A");  // A: freq=2
        cache.get("A");  // A: freq=3
        cache.get("B");  // B: freq=2
        
        // Frequencies: A=3, B=2, C=1
        cache.put("D", "4");  // Evicts C (lowest frequency)
        
        System.out.println(cache.get("C"));  // null (evicted)
        System.out.println(cache.get("A"));  // "1" (still there)
    }
}
```

### Using Caffeine (Production-Ready)

```java
// CaffeineCacheExample.java
package com.example.cache;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.RemovalCause;
import com.github.benmanes.caffeine.cache.Weigher;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

/**
 * Production-ready caching with Caffeine
 * 
 * Caffeine uses Window TinyLFU algorithm:
 * - Combines recency (LRU) and frequency (LFU)
 * - Admission filter prevents cache pollution
 * - Near-optimal hit rates
 */
public class CaffeineCacheExample {

    /**
     * Basic LRU-style cache with size limit
     */
    public Cache<String, User> createSizeBasedCache() {
        return Caffeine.newBuilder()
                .maximumSize(10_000)  // Evict when exceeds 10K entries
                .recordStats()         // Enable statistics
                .build();
    }

    /**
     * Cache with TTL (time-based expiration)
     */
    public Cache<String, User> createTtlCache() {
        return Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))  // Expire 5 min after write
                .build();
    }

    /**
     * Cache with idle timeout (sliding TTL)
     */
    public Cache<String, Session> createSessionCache() {
        return Caffeine.newBuilder()
                .maximumSize(100_000)
                .expireAfterAccess(Duration.ofMinutes(30))  // Expire 30 min after last access
                .build();
    }

    /**
     * Cache with weight-based eviction
     * Useful when entries have different sizes
     */
    public Cache<String, byte[]> createWeightBasedCache() {
        return Caffeine.newBuilder()
                .maximumWeight(100_000_000)  // 100MB total
                .weigher((String key, byte[] value) -> value.length)  // Weight = byte array size
                .build();
    }

    /**
     * Cache with eviction listener
     */
    public Cache<String, User> createCacheWithListener() {
        return Caffeine.newBuilder()
                .maximumSize(10_000)
                .evictionListener((String key, User value, RemovalCause cause) -> {
                    System.out.printf("Evicted %s due to %s%n", key, cause);
                    // Could write to secondary storage here
                })
                .build();
    }

    /**
     * Cache with automatic loading
     */
    public com.github.benmanes.caffeine.cache.LoadingCache<String, User> createLoadingCache(
            UserRepository repository) {
        return Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .build(key -> repository.findById(key));  // Auto-load on miss
    }

    /**
     * Get cache statistics
     */
    public void printStats(Cache<?, ?> cache) {
        var stats = cache.stats();
        System.out.printf("""
            Cache Statistics:
            - Hit Rate: %.2f%%
            - Miss Rate: %.2f%%
            - Load Count: %d
            - Eviction Count: %d
            - Average Load Time: %.2f ms
            """,
            stats.hitRate() * 100,
            stats.missRate() * 100,
            stats.loadCount(),
            stats.evictionCount(),
            stats.averageLoadPenalty() / 1_000_000.0
        );
    }

    // Domain classes for examples
    record User(String id, String name) {}
    record Session(String id, String userId, long createdAt) {}
    interface UserRepository {
        User findById(String id);
    }
}
```

### Configuring Redis Eviction

```java
// RedisEvictionConfig.java
package com.example.cache.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import io.lettuce.core.ClientOptions;
import io.lettuce.core.resource.ClientResources;

/**
 * Redis configuration with eviction policy settings
 * 
 * Note: Eviction policy is set in redis.conf, not in Java client.
 * This shows how to configure the connection.
 */
@Configuration
public class RedisEvictionConfig {

    /**
     * Redis configuration
     * 
     * In redis.conf or via CONFIG SET:
     * 
     * maxmemory 4gb
     * maxmemory-policy allkeys-lru
     * 
     * For LFU (Redis 4.0+):
     * maxmemory-policy allkeys-lfu
     * lfu-log-factor 10
     * lfu-decay-time 1
     */
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        
        return new LettuceConnectionFactory(config);
    }

    /**
     * Check and log current eviction policy
     */
    public void logEvictionPolicy(RedisConnectionFactory connectionFactory) {
        // Using Redis CLI or RedisTemplate:
        // CONFIG GET maxmemory-policy
        // Expected output: "allkeys-lru" or similar
    }
}
```

```conf
# redis.conf - Eviction settings

# Maximum memory Redis can use
maxmemory 4gb

# Eviction policy
# Options: noeviction, allkeys-lru, volatile-lru, allkeys-lfu, 
#          volatile-lfu, allkeys-random, volatile-random, volatile-ttl
maxmemory-policy allkeys-lru

# LFU settings (if using LFU policy)
# lfu-log-factor: Higher = slower frequency counter growth
# lfu-decay-time: Minutes before frequency counter is halved
lfu-log-factor 10
lfu-decay-time 1

# Sample size for eviction (higher = more accurate but slower)
maxmemory-samples 5
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Policy Comparison

| Policy | Hit Rate | Overhead | Scan Resistant | Adapts to Workload |
|--------|----------|----------|----------------|-------------------|
| FIFO | Low | Very Low | No | No |
| LRU | Good | Low | No | Somewhat |
| LFU | Good | Medium | Yes | No |
| Random | Medium | Very Low | Somewhat | No |
| ARC | Excellent | High | Yes | Yes |
| W-TinyLFU | Excellent | Medium | Yes | Yes |

### Common Mistakes

**1. Using FIFO When LRU is Needed**

```java
// WRONG: Using insertion-order LinkedHashMap
Map<String, User> cache = new LinkedHashMap<>(1000);

// RIGHT: Using access-order LinkedHashMap
Map<String, User> cache = new LinkedHashMap<>(1000, 0.75f, true);
```

**2. Not Considering Memory Overhead**

```java
// WRONG: Storing large objects directly
cache.put("user:123", entireUserObjectWith10MBOfHistory);

// RIGHT: Store references or summaries
cache.put("user:123:summary", userSummary);  // Small
// Fetch full data from DB when needed
```

**3. Wrong Policy for Workload**

```
Workload: Backup job scans all data once per night
Policy: LRU

Problem: Backup evicts all hot data!

Solution: 
- Use LFU (one-time access doesn't increase frequency)
- Or use ARC (adapts automatically)
- Or exclude backup from cache
```

**4. Not Monitoring Hit Rate**

```java
// WRONG: Set and forget
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .build();

// RIGHT: Monitor and tune
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .recordStats()  // Enable statistics
    .build();

// Periodically check
if (cache.stats().hitRate() < 0.8) {
    // Consider increasing cache size or changing policy
    log.warn("Cache hit rate below 80%: {}", cache.stats().hitRate());
}
```

---

## 8️⃣ When NOT to Use Each Policy

### FIFO: Don't Use When
- Items are accessed multiple times
- Access patterns have temporal locality
- Cache hit rate matters

### LRU: Don't Use When
- Workload has sequential scans
- Frequency matters more than recency
- Memory overhead is critical

### LFU: Don't Use When
- Access patterns change over time (old popular items stay forever)
- New items need fair chance
- Memory overhead is critical

### Random: Don't Use When
- Access patterns are skewed (some items much more popular)
- Predictable performance is needed
- Hit rate is critical

---

## 9️⃣ Comparison: Choosing the Right Policy

### Decision Guide

```
                              What's your workload?
                                      │
                    ┌─────────────────┴─────────────────┐
                    │                                   │
                    ▼                                   ▼
            Temporal Locality?                  Frequency Patterns?
        (Recent items accessed again)       (Some items always popular)
                    │                                   │
                    ▼                                   ▼
            ┌───────────────┐                   ┌───────────────┐
            │     LRU       │                   │     LFU       │
            └───────────────┘                   └───────────────┘
                                                        │
                                                        ▼
                                              Patterns change over time?
                                                        │
                                          ┌─────────────┴─────────────┐
                                          │                           │
                                          ▼                           ▼
                                         YES                          NO
                                          │                           │
                                          ▼                           ▼
                                  ┌───────────────┐           ┌───────────────┐
                                  │  W-TinyLFU    │           │   Pure LFU    │
                                  │  (Caffeine)   │           │               │
                                  └───────────────┘           └───────────────┘
```

### Real-World Choices

| Use Case | Recommended Policy | Why |
|----------|-------------------|-----|
| Web session cache | LRU | Recent sessions likely active |
| Product catalog | LFU or W-TinyLFU | Popular products accessed repeatedly |
| API response cache | LRU + TTL | Fresh data matters |
| DNS cache | LRU | Recent lookups likely repeated |
| Database query cache | W-TinyLFU | Mixed patterns |
| CDN edge cache | LRU | Temporal locality strong |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is cache eviction and why do we need it?**

A: Cache eviction is the process of removing items from a cache when it reaches capacity to make room for new items. We need it because caches have limited memory. Without eviction, we'd either run out of memory or have to reject new cache entries. The goal is to evict items that are least likely to be needed again, maximizing cache hit rate.

**Q: What's the difference between LRU and LFU?**

A: LRU (Least Recently Used) evicts the item that hasn't been accessed for the longest time. It assumes recently accessed items will be accessed again soon. LFU (Least Frequently Used) evicts the item with the fewest total accesses. It keeps popular items in cache. LRU works better when recent access predicts future access. LFU works better when some items are consistently more popular than others.

### L5 (Mid-Level) Questions

**Q: How would you implement an LRU cache with O(1) operations?**

A: I'd use a HashMap combined with a doubly linked list. The HashMap provides O(1) key lookup. The doubly linked list maintains access order. On access: (1) Find node in HashMap O(1). (2) Remove node from current position O(1) using prev/next pointers. (3) Insert at head O(1). On eviction: Remove from tail O(1). The HashMap stores key → node reference, enabling O(1) removal from the middle of the list.

**Q: What is cache pollution and how do you prevent it?**

A: Cache pollution occurs when items unlikely to be accessed again occupy cache space, evicting useful items. Common cause: sequential scans that touch many items once. Solutions: (1) Use LFU instead of LRU, as one-time accesses don't increase frequency much. (2) Use ARC or W-TinyLFU, which are scan-resistant. (3) Use an admission policy that requires items to be accessed multiple times before caching. (4) Segment the cache, keeping a small "probation" area for new items.

### L6 (Senior) Questions

**Q: Design an eviction policy for a CDN edge cache serving mixed content (static assets, API responses, personalized content).**

A: I'd design a multi-tier approach:

**Tier 1 - Static Assets (images, JS, CSS)**:
- Policy: LRU with long TTL (1 week)
- Reasoning: These rarely change and have high temporal locality
- Size: 70% of cache

**Tier 2 - API Responses (product data, search results)**:
- Policy: W-TinyLFU with medium TTL (5 minutes)
- Reasoning: Mix of popular and long-tail queries
- Size: 25% of cache

**Tier 3 - Personalized Content (user dashboards)**:
- Policy: LRU with short TTL (1 minute)
- Reasoning: Each user's data is unique, recent access matters
- Size: 5% of cache

**Admission Control**:
- Use Bloom filter to track "seen" items
- Only admit to main cache on second access
- Prevents scan pollution

**Monitoring**:
- Track hit rate per tier
- Adjust tier sizes based on observed patterns
- Alert if hit rate drops below threshold

---

## 1️⃣1️⃣ One Clean Mental Summary

Cache eviction policies determine which items to remove when the cache is full. **FIFO** removes the oldest item, simple but ignores access patterns. **LRU** removes the least recently accessed item, good for temporal locality. **LFU** removes the least frequently accessed item, keeps popular items but can be polluted by old popular items. **Random** is surprisingly effective with low overhead. **ARC** and **W-TinyLFU** adapt automatically between recency and frequency, providing near-optimal hit rates. Choose based on your access patterns: LRU for temporal locality, LFU for frequency patterns, W-TinyLFU (Caffeine) for mixed workloads. Always monitor hit rates and adjust.

