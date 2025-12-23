# 🗃️ LRU Cache - Code Walkthrough

## Building From Scratch: Step-by-Step

### Phase 1: Understand the Problem

**What is LRU Cache?**
- Cache with limited capacity
- When full, evicts the **Least Recently Used** item
- "Recently used" = most recently accessed (get or put)

**Requirements:**
- O(1) get operation
- O(1) put operation
- O(1) eviction

**Key Insight:**
- HashMap alone: O(1) lookup, but can't track access order
- LinkedList alone: O(1) reorder, but O(n) lookup
- **Combined**: O(1) for both!

---

### Phase 2: Design the Data Structure

```java
// Step 1: Node for doubly linked list
public class Node<K, V> {
    K key;      // Need key for HashMap removal during eviction
    V value;    // The cached value
    Node<K, V> prev;  // Previous node in list
    Node<K, V> next;  // Next node in list
}
```

**Why store key in Node?**

```
When evicting LRU (tail.prev):
1. We have the Node
2. We need to remove from HashMap
3. HashMap.remove() needs the KEY
4. Without key in Node, we can't find it!
```

```java
// Step 2: Cache structure
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> cache;  // O(1) lookup
    private final Node<K, V> head;  // Dummy head (MRU side)
    private final Node<K, V> tail;  // Dummy tail (LRU side)
}
```

**Why dummy head/tail?**

Without sentinels:
```java
void addToHead(Node node) {
    if (head == null) {
        // Special case: empty list
        head = tail = node;
    } else if (head == tail) {
        // Special case: single element
        node.next = head;
        head.prev = node;
        head = node;
    } else {
        // Normal case
        node.next = head;
        head.prev = node;
        head = node;
    }
}
```

With sentinels:
```java
void addToHead(Node node) {
    // Same code for ALL cases!
    node.prev = head;
    node.next = head.next;
    head.next.prev = node;
    head.next = node;
}
```

---

### Phase 3: Implement Core Operations

```java
// Step 3: Constructor
public LRUCache(int capacity) {
    if (capacity <= 0) {
        throw new IllegalArgumentException("Capacity must be positive");
    }
    
    this.capacity = capacity;
    this.cache = new HashMap<>();
    
    // Initialize dummy nodes
    this.head = new Node<>();
    this.tail = new Node<>();
    
    // Connect them
    head.next = tail;
    tail.prev = head;
}
```

**Initial state visualization:**

```
head ←→ tail
(dummy)  (dummy)

HashMap: {}
```

```java
// Step 4: get() operation
public V get(K key) {
    // Step 4a: Look up in HashMap
    Node<K, V> node = cache.get(key);
    
    if (node == null) {
        return null;  // Cache miss
    }
    
    // Step 4b: Move to head (mark as recently used)
    moveToHead(node);
    
    // Step 4c: Return value
    return node.value;
}
```

**get() trace:**

```
Before get("B"):
head ←→ C ←→ B ←→ A ←→ tail
         ↑
         target

After get("B"):
head ←→ B ←→ C ←→ A ←→ tail
         ↑
         moved to front
```

```java
// Step 5: put() operation
public void put(K key, V value) {
    Node<K, V> existingNode = cache.get(key);
    
    if (existingNode != null) {
        // Key exists: update value and move to head
        existingNode.value = value;
        moveToHead(existingNode);
    } else {
        // New key: create node
        Node<K, V> newNode = new Node<>(key, value);
        
        // Check capacity BEFORE adding
        if (cache.size() >= capacity) {
            evictLRU();
        }
        
        // Add to cache and list
        cache.put(key, newNode);
        addToHead(newNode);
    }
}
```

**put() decision tree:**

```
put(key, value)
       │
       ▼
┌──────────────┐
│ key exists?  │
└──────┬───────┘
       │
  ┌────┴────┐
  │         │
 Yes        No
  │         │
  ▼         ▼
Update    ┌──────────┐
value     │ at       │
  │       │ capacity?│
  │       └────┬─────┘
  │            │
  │       ┌────┴────┐
  │      Yes        No
  │       │         │
  │       ▼         │
  │    Evict        │
  │    LRU          │
  │       │         │
  │       └────┬────┘
  │            │
  ▼            ▼
Move to     Add new
head        node to head
```

---

### Phase 4: Implement Helper Methods

```java
// Step 6: addToHead - Add node right after dummy head
private void addToHead(Node<K, V> node) {
    // New node's prev points to head
    node.prev = head;
    
    // New node's next points to current first real node
    node.next = head.next;
    
    // Current first real node's prev points to new node
    head.next.prev = node;
    
    // Head's next points to new node
    head.next = node;
}
```

**addToHead visualization:**

```
Before addToHead(X):
head ←→ A ←→ B ←→ tail

Step 1: node.prev = head
head ← X    A ←→ B ←→ tail

Step 2: node.next = head.next (= A)
head ← X → A ←→ B ←→ tail

Step 3: head.next.prev = node (A.prev = X)
head ← X ←→ A ←→ B ←→ tail

Step 4: head.next = node
head ←→ X ←→ A ←→ B ←→ tail

After:
head ←→ X ←→ A ←→ B ←→ tail
```

```java
// Step 7: removeNode - Remove node from its current position
private void removeNode(Node<K, V> node) {
    // Previous node's next skips over this node
    node.prev.next = node.next;
    
    // Next node's prev skips over this node
    node.next.prev = node.prev;
}
```

**removeNode visualization:**

```
Before removeNode(B):
head ←→ A ←→ B ←→ C ←→ tail

Step 1: node.prev.next = node.next (A.next = C)
head ←→ A ──→ C ←→ tail
         ←─ B ─→

Step 2: node.next.prev = node.prev (C.prev = A)
head ←→ A ←→ C ←→ tail
         B (orphaned)

After:
head ←→ A ←→ C ←→ tail
```

```java
// Step 8: moveToHead - Combination of remove + addToHead
private void moveToHead(Node<K, V> node) {
    removeNode(node);
    addToHead(node);
}

// Step 9: evictLRU - Remove the least recently used entry
private void evictLRU() {
    // LRU is the node just before tail
    Node<K, V> lruNode = tail.prev;
    
    // Don't evict if list is empty (lruNode would be head)
    if (lruNode == head) {
        return;
    }
    
    // Remove from list
    removeNode(lruNode);
    
    // Remove from HashMap (this is why we store key in Node!)
    cache.remove(lruNode.key);
    
    // Notify listeners
    notifyEviction(lruNode.key, lruNode.value);
}
```

**evictLRU visualization:**

```
Before evictLRU():
head ←→ D ←→ C ←→ B ←→ A ←→ tail
        MRU              LRU ↑
                             │
                    tail.prev = A

After evictLRU():
head ←→ D ←→ C ←→ B ←→ tail

HashMap: removed key "A"
Listeners: notified about eviction
```

---

### Phase 5: Add Thread Safety

```java
// Step 10: ConcurrentLRUCache
public class ConcurrentLRUCache<K, V> {
    private final LRUCache<K, V> cache;
    private final ReentrantReadWriteLock lock;
    private final ReentrantReadWriteLock.ReadLock readLock;
    private final ReentrantReadWriteLock.WriteLock writeLock;
    
    public ConcurrentLRUCache(int capacity) {
        this.cache = new LRUCache<>(capacity);
        this.lock = new ReentrantReadWriteLock();
        this.readLock = lock.readLock();
        this.writeLock = lock.writeLock();
    }
    
    public V get(K key) {
        // IMPORTANT: get() modifies list order, needs WRITE lock
        writeLock.lock();
        try {
            return cache.get(key);
        } finally {
            writeLock.unlock();
        }
    }
    
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

**Why get() needs write lock:**

```java
// Thread A and B both call get("X") simultaneously

// If using read lock:
Thread A: node = cache.get("X")
Thread B: node = cache.get("X")
Thread A: removeNode(node)    // Modifies node.prev.next
Thread B: removeNode(node)    // CRASH! node.prev is corrupted
```

---

## Testing Approach

### Unit Tests

```java
// LRUCacheTest.java
public class LRUCacheTest {
    
    @Test
    void testBasicPutGet() {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        
        cache.put("A", 1);
        cache.put("B", 2);
        
        assertEquals(1, cache.get("A"));
        assertEquals(2, cache.get("B"));
        assertNull(cache.get("C"));
    }
    
    @Test
    void testEviction() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        
        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("C", 3);  // Should evict A
        
        assertNull(cache.get("A"));  // A was evicted
        assertEquals(2, cache.get("B"));
        assertEquals(3, cache.get("C"));
    }
    
    @Test
    void testAccessUpdatesOrder() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        
        cache.put("A", 1);
        cache.put("B", 2);
        cache.get("A");  // A is now MRU
        cache.put("C", 3);  // Should evict B (not A)
        
        assertEquals(1, cache.get("A"));  // A still exists
        assertNull(cache.get("B"));  // B was evicted
        assertEquals(3, cache.get("C"));
    }
    
    @Test
    void testUpdateExisting() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        
        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("A", 100);  // Update A
        
        assertEquals(100, cache.get("A"));
        assertEquals(2, cache.size());  // Size unchanged
    }
    
    @Test
    void testEvictionListener() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        
        List<String> evictedKeys = new ArrayList<>();
        cache.addEvictionListener((key, value) -> evictedKeys.add(key));
        
        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("C", 3);  // Evicts A
        
        assertEquals(1, evictedKeys.size());
        assertEquals("A", evictedKeys.get(0));
    }
    
    @Test
    void testCapacityOne() {
        LRUCache<String, Integer> cache = new LRUCache<>(1);
        
        cache.put("A", 1);
        assertEquals(1, cache.get("A"));
        
        cache.put("B", 2);  // Evicts A
        assertNull(cache.get("A"));
        assertEquals(2, cache.get("B"));
    }
    
    @Test
    void testRemove() {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        
        cache.put("A", 1);
        cache.put("B", 2);
        
        Integer removed = cache.remove("A");
        
        assertEquals(1, removed);
        assertNull(cache.get("A"));
        assertEquals(1, cache.size());
    }
}
```

### Concurrency Tests

```java
// ConcurrentLRUCacheTest.java
public class ConcurrentLRUCacheTest {
    
    @Test
    void testConcurrentAccess() throws InterruptedException {
        ConcurrentLRUCache<Integer, Integer> cache = new ConcurrentLRUCache<>(100);
        int numThreads = 10;
        int operationsPerThread = 1000;
        
        ExecutorService executor = Executors.newFixedThreadPool(numThreads);
        CountDownLatch latch = new CountDownLatch(numThreads);
        
        for (int t = 0; t < numThreads; t++) {
            final int threadId = t;
            executor.submit(() -> {
                try {
                    for (int i = 0; i < operationsPerThread; i++) {
                        int key = threadId * operationsPerThread + i;
                        cache.put(key, key);
                        cache.get(key);
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await(30, TimeUnit.SECONDS);
        executor.shutdown();
        
        // Cache should not exceed capacity
        assertTrue(cache.size() <= 100);
    }
    
    @Test
    void testNoDeadlock() throws InterruptedException {
        ConcurrentLRUCache<String, String> cache = new ConcurrentLRUCache<>(10);
        
        Thread writer = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                cache.put("key" + (i % 20), "value" + i);
            }
        });
        
        Thread reader = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                cache.get("key" + (i % 20));
            }
        });
        
        writer.start();
        reader.start();
        
        writer.join(5000);
        reader.join(5000);
        
        assertFalse(writer.isAlive());
        assertFalse(reader.isAlive());
    }
}
```

---

## Complexity Analysis

### Operation Breakdown

**get(key):**
```
1. HashMap.get(key)     → O(1)
2. removeNode(node)     → O(1) - just pointer updates
3. addToHead(node)      → O(1) - just pointer updates
Total: O(1)
```

**put(key, value):**
```
Case 1: Key exists
1. HashMap.get(key)     → O(1)
2. Update value         → O(1)
3. moveToHead(node)     → O(1)
Total: O(1)

Case 2: New key, not at capacity
1. HashMap.get(key)     → O(1)
2. Create Node          → O(1)
3. HashMap.put(key)     → O(1)
4. addToHead(node)      → O(1)
Total: O(1)

Case 3: New key, at capacity
1. HashMap.get(key)     → O(1)
2. evictLRU()           → O(1)
3. Create Node          → O(1)
4. HashMap.put(key)     → O(1)
5. addToHead(node)      → O(1)
Total: O(1)
```

### Space Analysis

```
Per entry:
- HashMap.Entry: key reference + value reference + hash + next pointer
  ≈ 32 bytes (on 64-bit JVM with compressed OOPs)
- Node: key + value + prev + next + object header
  ≈ 40 bytes

Total per entry: ~72 bytes + actual key/value sizes

For capacity N:
- HashMap: O(N)
- Nodes: O(N)
- Head/Tail dummies: O(1)
Total: O(N)
```

---

## Interview Follow-ups with Answers

### Q1: What happens if eviction listener throws exception?

```java
private void notifyEviction(K key, V value) {
    for (EvictionListener<K, V> listener : evictionListeners) {
        try {
            listener.onEviction(key, value);
        } catch (Exception e) {
            // Log but don't propagate
            // Cache operation should succeed even if listener fails
            System.err.println("Eviction listener error: " + e.getMessage());
        }
    }
}
```

### Q2: How would you implement getOrCompute()?

```java
public V getOrCompute(K key, Function<K, V> computeFunction) {
    V value = get(key);
    if (value != null) {
        return value;
    }
    
    // Compute and cache
    value = computeFunction.apply(key);
    put(key, value);
    return value;
}

// Thread-safe version needs double-checked locking
public V getOrComputeThreadSafe(K key, Function<K, V> computeFunction) {
    writeLock.lock();
    try {
        V value = cache.get(key);
        if (value != null) {
            return value;
        }
        
        value = computeFunction.apply(key);
        cache.put(key, value);
        return value;
    } finally {
        writeLock.unlock();
    }
}
```

### Q3: How would you add statistics/metrics?

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
    
    public double getHitRate() {
        long total = hits.get() + misses.get();
        return total == 0 ? 0 : (double) hits.get() / total;
    }
    
    public CacheStats getStats() {
        return new CacheStats(hits.get(), misses.get(), evictions.get());
    }
}
```

### Q4: What would you do differently with more time?

1. **Add bulk operations** - putAll(), removeAll()
2. **Add iteration** - forEach(), stream()
3. **Add soft/weak references** - For memory-sensitive caching
4. **Add size-based eviction** - Evict by memory size, not count
5. **Add async eviction** - Non-blocking eviction callbacks
6. **Add persistence** - Write-through/write-behind to disk

