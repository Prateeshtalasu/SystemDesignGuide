# 🗃️ LRU Cache - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Cache Operations with Eviction

```
Initial State: LRUCache(capacity=3), empty

put(1, "A"):
  Cache: {1→A}, Order: [1]

put(2, "B"):
  Cache: {1→A, 2→B}, Order: [1, 2]

put(3, "C"):
  Cache: {1→A, 2→B, 3→C}, Order: [1, 2, 3]

get(1):
  Returns "A", moves 1 to end
  Order: [2, 3, 1]

put(4, "D"):
  Capacity reached, evict LRU (key=2)
  Cache: {1→A, 3→C, 4→D}, Order: [3, 1, 4]
```

**Final State:**

```
Cache: {1→A, 3→C, 4→D}
Order: [3, 1, 4] (3 is LRU, 4 is MRU)
All operations completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Empty Cache Get

**Initial State:**

```
LRUCache(capacity=3), empty
```

**Step-by-step:**

1. `get(1)` on empty cache

   - Key 1 not in cache → returns null
   - No exception thrown, graceful handling

2. `get(null)` (invalid input)

   - Null key validation → throws IllegalArgumentException
   - Cache state unchanged

3. `put(null, "value")` (invalid input)
   - Null key validation → throws IllegalArgumentException
   - Cache state unchanged

**Final State:**

```
Cache: {} (still empty)
Invalid operations rejected without side effects
```

---

### Scenario 3: Concurrency/Race Condition - Concurrent Access

**Initial State:**

```
ConcurrentLRUCache(capacity=3), empty
Thread A: Writing operations
Thread B: Reading operations
Thread C: Writing operations
```

**Step-by-step (simulating concurrent access):**

**Thread A:** `cache.put(1, "A")` at time T0
**Thread B:** `cache.get(1)` at time T0 (concurrent)
**Thread C:** `cache.put(2, "B")` at time T0 (concurrent)

1. **Thread A:** Acquires write lock, puts (1, "A")

   - Cache: {1→A}
   - Releases write lock

2. **Thread B:** Acquires read lock, gets(1)

   - Read lock allows concurrent reads
   - Returns "A" successfully
   - Releases read lock

3. **Thread C:** Acquires write lock (waits if Thread A holds it)

   - Puts (2, "B")
   - Cache: {1→A, 2→B}
   - Releases write lock

4. **Thread A:** `cache.put(3, "C")` (concurrent with Thread C)

   - Write lock ensures atomicity
   - Cache: {1→A, 2→B, 3→C}

5. **Thread B:** `cache.get(2)` (concurrent with Thread A)
   - Read lock allows concurrent access
   - Returns "B" successfully

**Final State:**

```
Cache: {1→A, 2→B, 3→C}
All concurrent operations completed correctly
No data corruption, proper read-write lock handling
Thread-safe behavior maintained
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions

- **Empty Cache Get**: Returns null
- **Capacity 1**: Every put evicts previous
- **Update Existing Key**: Updates value, moves to front
- **Concurrent Access**: Thread safety verification

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

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
