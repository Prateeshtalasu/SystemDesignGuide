# 🚦 Rate Limiter - Code Walkthrough

## Building From Scratch: Step-by-Step

### Phase 1: Understand the Problem

**What is Rate Limiting?**
- Control how many requests a client can make
- Protect systems from overload
- Ensure fair resource usage

**Key Questions:**
- How many requests per time period?
- What happens when limit is exceeded?
- How to identify clients (IP, API key, user ID)?

---

### Phase 2: Token Bucket Algorithm

```java
// Step 1: Define the bucket state
private static class Bucket {
    private double tokens;       // Current token count
    private long lastRefillTime; // When we last added tokens
}
```

**Why double for tokens?**
- Refill rate might not be integer
- Example: 1.5 tokens per second
- Allows fractional tokens to accumulate

```java
// Step 2: Refill logic
private void refill(double refillRatePerMs, int capacity) {
    long now = System.currentTimeMillis();
    long elapsed = now - lastRefillTime;
    
    if (elapsed > 0) {
        // Calculate tokens to add
        double tokensToAdd = elapsed * refillRatePerMs;
        
        // Don't exceed capacity
        tokens = Math.min(capacity, tokens + tokensToAdd);
        
        lastRefillTime = now;
    }
}
```

**Refill calculation example:**

```
Configuration: refillRate = 10/sec, capacity = 100
Current state: tokens = 50, lastRefillTime = T0

At T0 + 2000ms (2 seconds later):
  elapsed = 2000ms
  refillRatePerMs = 10 / 1000 = 0.01
  tokensToAdd = 2000 * 0.01 = 20
  tokens = min(100, 50 + 20) = 70
```

```java
// Step 3: Consume tokens
synchronized boolean tryConsume(int permits, double refillRatePerMs, int capacity) {
    // First, add any tokens that should have accumulated
    refill(refillRatePerMs, capacity);
    
    // Check if we have enough tokens
    if (tokens >= permits) {
        tokens -= permits;
        return true;
    }
    return false;
}
```

**Complete token bucket trace:**

```
Config: capacity=5, refillRate=2/sec

T0: Initial
  tokens = 5
  lastRefillTime = T0

T0: tryConsume(1)
  refill: elapsed=0, no tokens added
  tokens(5) >= 1? Yes
  tokens = 4
  return true

T0: tryConsume(1) x4
  tokens = 0

T0: tryConsume(1)
  tokens(0) >= 1? No
  return false

T0 + 500ms: tryConsume(1)
  refill: elapsed=500ms
  tokensToAdd = 500 * 0.002 = 1
  tokens = 0 + 1 = 1
  tokens(1) >= 1? Yes
  tokens = 0
  return true
```

---

### Phase 3: Fixed Window Algorithm

```java
// Step 4: Window state
private static class Window {
    final long windowId;        // Which window this is
    final AtomicInteger count;  // Requests in this window
}
```

**Window ID calculation:**

```java
long windowId = currentTimeMs / windowSizeMs;

// Example: windowSize = 1000ms (1 second)
// At T = 1500ms: windowId = 1500 / 1000 = 1
// At T = 2500ms: windowId = 2500 / 1000 = 2
```

```java
// Step 5: Try acquire with window transition
public boolean tryAcquire(String key, int permits) {
    long now = System.currentTimeMillis();
    long currentWindow = now / windowSizeMs;
    
    // Get or create window, reset if new window
    Window window = windows.compute(key, (k, existing) -> {
        if (existing == null || existing.windowId != currentWindow) {
            return new Window(currentWindow);  // New window
        }
        return existing;  // Same window
    });
    
    return window.tryIncrement(permits, maxRequests);
}
```

**Fixed window trace:**

```
Config: maxRequests=5, windowSize=1000ms

T=500ms: Request 1
  windowId = 500 / 1000 = 0
  Window 0: count = 1
  Result: ALLOWED

T=800ms: Request 2-5
  windowId = 0 (same window)
  Window 0: count = 5
  Result: All ALLOWED

T=900ms: Request 6
  windowId = 0
  count(5) + 1 > 5? Yes
  Result: REJECTED

T=1100ms: Request 7
  windowId = 1100 / 1000 = 1  (NEW WINDOW!)
  Create new Window 1: count = 0
  count(0) + 1 <= 5? Yes
  Window 1: count = 1
  Result: ALLOWED
```

---

### Phase 4: Sliding Window Counter

```java
// Step 6: Track two windows
private static class WindowPair {
    long currentWindowId;
    AtomicInteger currentCount;
    int previousCount;  // Snapshot of previous window
}
```

**Why track previous window?**
- To calculate weighted average
- Smooths out boundary problem

```java
// Step 7: Window transition
synchronized void updateWindow(long newWindowId) {
    if (newWindowId > currentWindowId) {
        if (newWindowId == currentWindowId + 1) {
            // Consecutive window: save current as previous
            previousCount = currentCount.get();
        } else {
            // Skipped windows: previous is 0
            previousCount = 0;
        }
        currentWindowId = newWindowId;
        currentCount.set(0);
    }
}
```

```java
// Step 8: Weighted calculation
synchronized boolean tryAcquire(int permits, int max, long windowPosition, long windowSize) {
    // How far into current window (0.0 to 1.0)
    double positionRatio = (double) windowPosition / windowSize;
    
    // Weight for previous window (decreases as we move into current)
    double previousWeight = 1.0 - positionRatio;
    
    // Weighted count
    double weightedCount = previousCount * previousWeight + currentCount.get();
    
    if (weightedCount + permits <= max) {
        currentCount.addAndGet(permits);
        return true;
    }
    return false;
}
```

**Sliding window counter trace:**

```
Config: maxRequests=10, windowSize=1000ms

Window 0 (0-999ms): 8 requests
Window 1 (1000-1999ms): starts

T=1200ms: Request arrives
  windowId = 1
  windowPosition = 200ms
  positionRatio = 200/1000 = 0.2
  previousWeight = 1 - 0.2 = 0.8
  
  previousCount = 8 (from window 0)
  currentCount = 0
  
  weightedCount = 8 * 0.8 + 0 = 6.4
  
  6.4 + 1 <= 10? Yes
  currentCount = 1
  Result: ALLOWED

T=1800ms: Multiple requests
  positionRatio = 800/1000 = 0.8
  previousWeight = 1 - 0.8 = 0.2
  
  weightedCount = 8 * 0.2 + currentCount
  
  Previous window has less influence now!
```

---

### Phase 5: Sliding Window Log

```java
// Step 9: Store timestamps
private final Map<String, Deque<Long>> requestLogs;

public boolean tryAcquire(String key, int permits) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowSizeMs;
    
    Deque<Long> timestamps = requestLogs.computeIfAbsent(
        key, k -> new LinkedList<>());
    
    synchronized (timestamps) {
        // Remove expired timestamps
        while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStart) {
            timestamps.pollFirst();
        }
        
        // Check limit
        if (timestamps.size() + permits <= maxRequests) {
            for (int i = 0; i < permits; i++) {
                timestamps.addLast(now);
            }
            return true;
        }
    }
    
    return false;
}
```

**Sliding window log trace:**

```
Config: maxRequests=5, windowSize=1000ms

T=100ms: Request 1
  timestamps = [100]
  count(1) <= 5? Yes → ALLOWED

T=200ms: Request 2
  timestamps = [100, 200]
  count(2) <= 5? Yes → ALLOWED

T=500ms: Requests 3-5
  timestamps = [100, 200, 500, 500, 500]
  count(5) <= 5? Yes → ALLOWED

T=600ms: Request 6
  windowStart = 600 - 1000 = -400 (no cleanup needed)
  timestamps = [100, 200, 500, 500, 500]
  count(5) + 1 > 5? Yes → REJECTED

T=1200ms: Request 7
  windowStart = 1200 - 1000 = 200
  Cleanup: remove 100 (< 200)
  timestamps = [200, 500, 500, 500]
  count(4) + 1 <= 5? Yes → ALLOWED
  timestamps = [200, 500, 500, 500, 1200]
```

---

### Phase 6: Thread Safety

```java
// Step 10: Per-key locking
// Option A: Synchronized on bucket
synchronized boolean tryConsume(...) {
    // Only one thread per bucket
}

// Option B: ConcurrentHashMap + AtomicInteger
private final Map<String, Window> windows = new ConcurrentHashMap<>();

boolean tryIncrement(int permits, int max) {
    while (true) {
        int current = count.get();
        if (current + permits > max) {
            return false;
        }
        if (count.compareAndSet(current, current + permits)) {
            return true;
        }
        // Retry if CAS failed
    }
}
```

**CAS (Compare-And-Swap) explained:**

```
Thread A and B both try to increment count from 3 to 4:

Thread A: current = count.get() = 3
Thread B: current = count.get() = 3

Thread A: compareAndSet(3, 4) → SUCCESS, count = 4
Thread B: compareAndSet(3, 4) → FAIL (count is now 4, not 3)
Thread B: retry loop
Thread B: current = count.get() = 4
Thread B: compareAndSet(4, 5) → SUCCESS, count = 5

Result: Both increments happened correctly!
```

---

## Testing Approach

### Unit Tests

```java
// TokenBucketRateLimiterTest.java
public class TokenBucketRateLimiterTest {
    
    @Test
    void testBurstAllowed() {
        RateLimiter limiter = new TokenBucketRateLimiter(10, 5);
        
        // Should allow burst up to capacity
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.tryAcquire("user1"));
        }
        
        // 11th request should be rejected
        assertFalse(limiter.tryAcquire("user1"));
    }
    
    @Test
    void testRefill() throws InterruptedException {
        RateLimiter limiter = new TokenBucketRateLimiter(5, 10); // 10/sec refill
        
        // Consume all tokens
        for (int i = 0; i < 5; i++) {
            limiter.tryAcquire("user1");
        }
        assertFalse(limiter.tryAcquire("user1"));
        
        // Wait for refill
        Thread.sleep(200); // Should add ~2 tokens
        
        assertTrue(limiter.tryAcquire("user1"));
        assertTrue(limiter.tryAcquire("user1"));
    }
    
    @Test
    void testIndependentKeys() {
        RateLimiter limiter = new TokenBucketRateLimiter(2, 1);
        
        // User1 uses their limit
        assertTrue(limiter.tryAcquire("user1"));
        assertTrue(limiter.tryAcquire("user1"));
        assertFalse(limiter.tryAcquire("user1"));
        
        // User2 has their own limit
        assertTrue(limiter.tryAcquire("user2"));
        assertTrue(limiter.tryAcquire("user2"));
        assertFalse(limiter.tryAcquire("user2"));
    }
}
```

```java
// FixedWindowRateLimiterTest.java
public class FixedWindowRateLimiterTest {
    
    @Test
    void testWindowReset() throws InterruptedException {
        RateLimiter limiter = new FixedWindowRateLimiter(5, 100); // 100ms window
        
        // Use all in first window
        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire("user1"));
        }
        assertFalse(limiter.tryAcquire("user1"));
        
        // Wait for new window
        Thread.sleep(150);
        
        // New window, new limit
        assertTrue(limiter.tryAcquire("user1"));
    }
    
    @Test
    void testBoundaryProblem() throws InterruptedException {
        // This test demonstrates the boundary problem
        RateLimiter limiter = new FixedWindowRateLimiter(10, 1000);
        
        // Wait until near end of window
        long windowEnd = ((System.currentTimeMillis() / 1000) + 1) * 1000;
        Thread.sleep(windowEnd - System.currentTimeMillis() - 50);
        
        // 10 requests at end of window
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.tryAcquire("user1"));
        }
        
        // Wait for new window
        Thread.sleep(100);
        
        // 10 more at start of new window
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.tryAcquire("user1"));
        }
        
        // 20 requests in ~150ms - boundary problem!
    }
}
```

### Concurrency Tests

```java
// ConcurrencyTest.java
public class ConcurrencyTest {
    
    @Test
    void testConcurrentAccess() throws InterruptedException {
        RateLimiter limiter = new TokenBucketRateLimiter(100, 50);
        AtomicInteger allowed = new AtomicInteger(0);
        AtomicInteger rejected = new AtomicInteger(0);
        
        int numThreads = 10;
        int requestsPerThread = 50;
        CountDownLatch latch = new CountDownLatch(numThreads);
        
        for (int t = 0; t < numThreads; t++) {
            new Thread(() -> {
                for (int i = 0; i < requestsPerThread; i++) {
                    if (limiter.tryAcquire("shared-key")) {
                        allowed.incrementAndGet();
                    } else {
                        rejected.incrementAndGet();
                    }
                }
                latch.countDown();
            }).start();
        }
        
        latch.await(10, TimeUnit.SECONDS);
        
        // Total should equal all requests
        assertEquals(numThreads * requestsPerThread, allowed.get() + rejected.get());
        
        // Allowed should not exceed capacity
        assertTrue(allowed.get() <= 100);
    }
}
```

---

## Complexity Analysis

### Token Bucket

```
tryAcquire(key):
  1. computeIfAbsent on ConcurrentHashMap: O(1) amortized
  2. synchronized block per bucket
     - refill(): O(1) - just arithmetic
     - token check: O(1)
  Total: O(1)

Space: O(k) where k = unique keys
  - Each bucket: 2 longs + overhead ≈ 32 bytes
```

### Sliding Window Log

```
tryAcquire(key):
  1. Get/create deque: O(1)
  2. Remove expired (synchronized):
     - Worst case: O(n) where n = requests in window
  3. Add new timestamps: O(permits)
  Total: O(n) worst case, O(1) amortized

Space: O(k * n)
  - Each key: deque of n timestamps
  - Each timestamp: 8 bytes
  
For 100 requests/sec, 1 minute window:
  - 6000 timestamps per key
  - ~48KB per key
```

### Fixed Window

```
tryAcquire(key):
  1. Compute window ID: O(1)
  2. Get/create window: O(1)
  3. CAS increment: O(1) expected (few retries)
  Total: O(1)

Space: O(k)
  - Each window: 1 long + 1 AtomicInteger ≈ 24 bytes
```

---

## Interview Follow-ups with Answers

### Q1: How would you handle clock skew in distributed systems?

```java
// Use logical clocks or centralized time
public class ClockSyncRateLimiter implements RateLimiter {
    private final TimeService timeService;  // Centralized time
    
    @Override
    public boolean tryAcquire(String key) {
        long now = timeService.getCurrentTime();  // From central server
        // Use 'now' for all calculations
    }
}

// Or use NTP-synchronized time with tolerance
public class TolerantRateLimiter implements RateLimiter {
    private static final long CLOCK_TOLERANCE_MS = 100;
    
    private void cleanup(Deque<Long> timestamps, long windowStart) {
        // Add tolerance to account for clock skew
        long adjustedWindowStart = windowStart - CLOCK_TOLERANCE_MS;
        while (!timestamps.isEmpty() && timestamps.peekFirst() < adjustedWindowStart) {
            timestamps.pollFirst();
        }
    }
}
```

### Q2: How would you implement priority-based rate limiting?

```java
public class PriorityRateLimiter implements RateLimiter {
    private final Map<Priority, RateLimiter> limiters;
    
    public PriorityRateLimiter(int totalCapacity) {
        // High priority gets 50%, medium 30%, low 20%
        limiters = new EnumMap<>(Priority.class);
        limiters.put(Priority.HIGH, new TokenBucketRateLimiter((int)(totalCapacity * 0.5), 10));
        limiters.put(Priority.MEDIUM, new TokenBucketRateLimiter((int)(totalCapacity * 0.3), 10));
        limiters.put(Priority.LOW, new TokenBucketRateLimiter((int)(totalCapacity * 0.2), 10));
    }
    
    public boolean tryAcquire(String key, Priority priority) {
        // Try own bucket first
        if (limiters.get(priority).tryAcquire(key)) {
            return true;
        }
        
        // High priority can borrow from lower priorities
        if (priority == Priority.HIGH) {
            if (limiters.get(Priority.MEDIUM).tryAcquire(key)) return true;
            if (limiters.get(Priority.LOW).tryAcquire(key)) return true;
        }
        
        return false;
    }
}
```

### Q3: How would you implement rate limiting with quotas?

```java
public class QuotaRateLimiter implements RateLimiter {
    private final Map<String, Quota> quotas;
    private final RateLimiter burstLimiter;  // Per-second burst limit
    
    @Override
    public boolean tryAcquire(String key) {
        Quota quota = quotas.get(key);
        if (quota == null) {
            return false;  // No quota assigned
        }
        
        // Check burst limit (short-term)
        if (!burstLimiter.tryAcquire(key)) {
            return false;
        }
        
        // Check quota (long-term)
        return quota.tryConsume(1);
    }
    
    static class Quota {
        private final AtomicLong remaining;
        private final LocalDate resetDate;
        
        boolean tryConsume(int amount) {
            return remaining.addAndGet(-amount) >= 0;
        }
    }
}
```

### Q4: What would you do differently with more time?

1. **Add backpressure** - Queue requests instead of rejecting
2. **Add warm-up period** - Gradually increase limit after cold start
3. **Add adaptive limits** - Adjust based on system health
4. **Add cost-based limiting** - Different operations cost different tokens
5. **Add detailed metrics** - Latency percentiles, rejection rates
6. **Add configuration hot-reload** - Change limits without restart

