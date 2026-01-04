# 🚦 Rate Limiter - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Token Bucket Algorithm

```
Config: 10 tokens/second, bucket size 10

T=0: Bucket has 10 tokens
  tryAcquire("user1") → true, tokens=9
  tryAcquire("user1") → true, tokens=8

T=0.5: 5 tokens refilled, bucket=13→10 (capped)
  tryAcquire("user1", 5) → true, tokens=5

T=0.5: No refill yet
  tryAcquire("user1", 6) → false (only 5 available)
```

**Final State:**

```
Bucket: 5 tokens
All requests processed correctly
Rate limiting working as expected
```

---

### Scenario 2: Failure/Invalid Input - Request Exceeds Bucket Size

**Initial State:**

```
TokenBucketRateLimiter: 10 tokens/second refill, bucket size 10
Bucket: 10 tokens (full)
```

**Step-by-step:**

1. `tryAcquire("user1", 5)` → true, tokens: 10 - 5 = 5

2. `tryAcquire("user1", 15)` → false

   - Requested permits (15) > bucket size (10)
   - Even if bucket was full, cannot grant 15 tokens
   - Returns false immediately
   - Tokens remain: 5

3. `tryAcquire("user1", -1)` (invalid input)

   - Negative permits → throws IllegalArgumentException
   - No state change

4. `tryAcquire(null, 1)` (invalid input)
   - Null key → throws IllegalArgumentException
   - No state change

**Final State:**

```
Bucket: 5 tokens
Invalid requests rejected
Large requests properly rejected even if bucket has capacity
```

---

### Scenario 3: Concurrency/Race Condition - Concurrent Token Consumption

**Initial State:**

```
TokenBucketRateLimiter: 10 tokens/second, bucket size 10
Bucket: 10 tokens
Thread A: User1 requests
Thread B: User1 requests (concurrent)
Thread C: User2 requests (different user)
```

**Step-by-step (simulating concurrent requests):**

**Thread A:** `tryAcquire("user1", 5)` at time T0
**Thread B:** `tryAcquire("user1", 6)` at time T0 (concurrent, same user)
**Thread C:** `tryAcquire("user2", 3)` at time T0 (concurrent, different user)

1. **Thread A:** Acquires lock on user1's bucket

   - Checks tokens: 10 >= 5 → true
   - Consumes 5 tokens, tokens = 5
   - Releases lock, returns true

2. **Thread B:** Waits for lock (Thread A holds it)

   - After Thread A releases, acquires lock
   - Checks tokens: 5 >= 6 → false
   - Returns false, tokens remain 5

3. **Thread C:** Operates on user2's bucket (different key, no lock contention)
   - User2 has separate bucket with 10 tokens
   - Consumes 3 tokens, tokens = 7
   - Returns true

**Final State:**

```
User1 bucket: 5 tokens (Thread A succeeded, Thread B rejected)
User2 bucket: 7 tokens (Thread C succeeded)
No race conditions, proper synchronization per user
Independent user buckets handled correctly
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions

- **Burst Traffic**: All tokens consumed at once
- **Zero Tokens**: Immediate rejection
- **Large Permit Request**: Exceeds bucket size
- **Time Overflow**: Handle long-running systems

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

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
