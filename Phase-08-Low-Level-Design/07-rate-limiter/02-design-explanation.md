# 🚦 Rate Limiter - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Rate Limiter.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `RateLimiter` | Define rate limiting contract | API changes |
| `TokenBucketRateLimiter` | Token bucket algorithm | Algorithm tuning |
| `SlidingWindowLogRateLimiter` | Sliding window log algorithm | Memory optimization |
| `SlidingWindowCounterRateLimiter` | Sliding window counter algorithm | Accuracy improvements |
| `FixedWindowRateLimiter` | Fixed window algorithm | Simplicity changes |
| `RateLimiterConfig` | Hold configuration | New config options |
| `RateLimiterFactory` | Create rate limiters | New algorithms |

**SRP in Action:**

```java
// Each algorithm is in its own class
public class TokenBucketRateLimiter implements RateLimiter {
    // ONLY handles token bucket logic
}

public class SlidingWindowLogRateLimiter implements RateLimiter {
    // ONLY handles sliding window log logic
}

// Configuration is separate
public class RateLimiterConfig {
    // ONLY holds configuration
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Algorithms:**

```java
// No changes to existing code needed!

// Add new algorithm
public class AdaptiveRateLimiter implements RateLimiter {
    private final RateLimiter delegate;
    
    @Override
    public boolean tryAcquire(String key) {
        // Adaptive logic
        if (systemUnderLoad()) {
            return stricterLimit.tryAcquire(key);
        }
        return delegate.tryAcquire(key);
    }
}

// Add to factory (optional)
public static RateLimiter create(Algorithm algorithm, RateLimiterConfig config) {
    // Existing code unchanged
    case ADAPTIVE:
        return new AdaptiveRateLimiter(config);
}
```

**Adding New Features:**

```java
// Decorator pattern for adding features
public class LoggingRateLimiter implements RateLimiter {
    private final RateLimiter delegate;
    private final Logger logger;
    
    @Override
    public boolean tryAcquire(String key) {
        boolean allowed = delegate.tryAcquire(key);
        logger.info("Rate limit check: key={}, allowed={}", key, allowed);
        return allowed;
    }
}

public class MetricsRateLimiter implements RateLimiter {
    private final RateLimiter delegate;
    private final MetricsRegistry metrics;
    
    @Override
    public boolean tryAcquire(String key) {
        boolean allowed = delegate.tryAcquire(key);
        if (allowed) {
            metrics.increment("ratelimit.allowed");
        } else {
            metrics.increment("ratelimit.rejected");
        }
        return allowed;
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All implementations are interchangeable:**

```java
// Any RateLimiter works here
public class ApiGateway {
    private final RateLimiter rateLimiter;
    
    public ApiGateway(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }
    
    public Response handleRequest(Request request) {
        String key = request.getClientId();
        
        if (!rateLimiter.tryAcquire(key)) {
            return Response.tooManyRequests();
        }
        
        return processRequest(request);
    }
}

// All these work correctly:
new ApiGateway(new TokenBucketRateLimiter(100, 10));
new ApiGateway(new SlidingWindowCounterRateLimiter(100, 1000));
new ApiGateway(new FixedWindowRateLimiter(100, 1000));
```

**LSP Contract:**
- `tryAcquire()` returns `true` if request allowed
- `tryAcquire()` returns `false` if rate limited
- `getRemainingTokens()` returns non-negative count

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
// Minimal interface
public interface RateLimiter {
    boolean tryAcquire(String key);
    boolean tryAcquire(String key, int permits);
    int getRemainingTokens(String key);
}
```

**If we needed more features:**

```java
// Separate interfaces for different needs
public interface RateLimiter {
    boolean tryAcquire(String key);
}

public interface BatchRateLimiter extends RateLimiter {
    boolean tryAcquire(String key, int permits);
}

public interface InspectableRateLimiter extends RateLimiter {
    int getRemainingTokens(String key);
    long getResetTime(String key);
}

public interface BlockingRateLimiter extends RateLimiter {
    void acquire(String key) throws InterruptedException;
    boolean tryAcquire(String key, long timeout, TimeUnit unit);
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// Factory depends on concrete implementations
public class RateLimiterFactory {
    public static RateLimiter create(Algorithm algorithm, ...) {
        switch (algorithm) {
            case TOKEN_BUCKET:
                return new TokenBucketRateLimiter(...);
        }
    }
}
```

**Better with DIP:**

```java
// Define abstraction for algorithm providers
public interface RateLimiterProvider {
    RateLimiter create(RateLimiterConfig config);
    String getAlgorithmName();
}

// Implementations
public class TokenBucketProvider implements RateLimiterProvider {
    @Override
    public RateLimiter create(RateLimiterConfig config) {
        return new TokenBucketRateLimiter(config);
    }
    
    @Override
    public String getAlgorithmName() {
        return "TOKEN_BUCKET";
    }
}

// Factory depends on abstraction
public class RateLimiterFactory {
    private final Map<String, RateLimiterProvider> providers;
    
    public RateLimiterFactory(List<RateLimiterProvider> providers) {
        this.providers = providers.stream()
            .collect(Collectors.toMap(
                RateLimiterProvider::getAlgorithmName,
                p -> p
            ));
    }
    
    public RateLimiter create(String algorithm, RateLimiterConfig config) {
        RateLimiterProvider provider = providers.get(algorithm);
        return provider.create(config);
    }
}
```

**Why We Use Concrete Factory Implementations in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete factory implementations instead of a RateLimiterProvider interface for the following reasons:

1. **Well-Known Types**: The factory creates well-known rate limiter types (TokenBucket, SlidingWindow, LeakyBucket). There's no requirement for multiple factory implementations in the interview context.

2. **Core Algorithm Focus**: LLD interviews focus on rate limiting algorithms and token management. Adding factory abstraction shifts focus away from these core concepts.

3. **Interview Time Constraints**: Implementing full provider interface hierarchies takes time away from demonstrating more critical LLD concepts like algorithm implementation and concurrency control.

4. **Production vs Interview**: In production systems, we would absolutely extract `RateLimiterProvider` interface and use dependency injection for:
   - Testability (mock providers in unit tests)
   - Algorithm extensibility (add new algorithms without modifying factory)
   - Plugin architecture (dynamically load rate limiter implementations)

**The Trade-off:**
- **Interview Scope**: Concrete factory focuses on algorithm implementation and token management
- **Production Scope**: Provider interface provides testability and algorithm extensibility

Note: We do use the RateLimiter interface well (good DIP!), showing we understand when interfaces add value.

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. RateLimiter interface defines contract, TokenBucketRateLimiter manages tokens, SlidingWindowRateLimiter manages windows, RateLimiterFactory creates instances. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new rate limiting algorithms) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All RateLimiter implementations properly implement the RateLimiter interface contract. They are substitutable. | N/A | - |
| **ISP** | PASS | RateLimiter interface is minimal and focused. Clients only depend on what they need (tryAcquire method). No unused methods. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | Factory depends on concrete implementations. For LLD interview scope, this is acceptable as it focuses on core rate limiting algorithms. In production, we would use RateLimiterProvider interface abstraction. | See "Why We Use Concrete Factory Implementations" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and algorithm extensibility |

---

## Design Patterns Used

### 1. Strategy Pattern

**Where:** Different rate limiting algorithms

```java
// Strategy interface
public interface RateLimiter {
    boolean tryAcquire(String key);
}

// Concrete strategies
public class TokenBucketRateLimiter implements RateLimiter { }
public class SlidingWindowLogRateLimiter implements RateLimiter { }
public class FixedWindowRateLimiter implements RateLimiter { }

// Context
public class ApiGateway {
    private RateLimiter strategy;
    
    public void setRateLimiter(RateLimiter strategy) {
        this.strategy = strategy;
    }
    
    public boolean checkLimit(String key) {
        return strategy.tryAcquire(key);
    }
}
```

**Benefits:**
- Swap algorithms at runtime
- Add new algorithms without changing clients
- Test with mock rate limiters

---

### 2. Factory Pattern

**Where:** RateLimiterFactory

```java
public class RateLimiterFactory {
    public static RateLimiter create(Algorithm algorithm, RateLimiterConfig config) {
        switch (algorithm) {
            case TOKEN_BUCKET:
                return new TokenBucketRateLimiter(config);
            case SLIDING_WINDOW_LOG:
                return new SlidingWindowLogRateLimiter(config);
            // ...
        }
    }
}

// Usage
RateLimiter limiter = RateLimiterFactory.create(
    Algorithm.TOKEN_BUCKET,
    RateLimiterConfig.perSecond(100)
);
```

---

### 3. Builder Pattern

**Where:** RateLimiterConfig

```java
RateLimiterConfig config = RateLimiterConfig.builder()
    .maxRequests(100)
    .windowSizeMs(1000)
    .bucketCapacity(200)
    .refillRatePerSecond(50)
    .build();
```

**Benefits:**
- Clear, readable configuration
- Optional parameters
- Immutable result

---

### 4. Decorator Pattern

**Where:** Adding cross-cutting concerns

```java
// Base rate limiter
RateLimiter base = new TokenBucketRateLimiter(100, 10);

// Add logging
RateLimiter withLogging = new LoggingRateLimiter(base);

// Add metrics
RateLimiter withMetrics = new MetricsRateLimiter(withLogging);

// Add circuit breaker
RateLimiter withCircuitBreaker = new CircuitBreakerRateLimiter(withMetrics);
```

---

## Algorithm Comparison

### When to Use Each Algorithm

| Algorithm | Best For | Avoid When |
|-----------|----------|------------|
| Token Bucket | APIs allowing bursts | Need strict rate |
| Leaky Bucket | Smooth output rate | Bursts are acceptable |
| Fixed Window | Simple, low overhead | Boundary abuse possible |
| Sliding Window Log | Highest accuracy | High request volume |
| Sliding Window Counter | Balance of accuracy/memory | Need exact counts |

### Detailed Comparison

```
                    Token     Leaky     Fixed     Sliding   Sliding
                    Bucket    Bucket    Window    Log       Counter
─────────────────────────────────────────────────────────────────────
Allows bursts       ✅        ❌        ✅        ❌        Partial
Memory usage        Low       Low       Low       High      Medium
Accuracy            Good      Exact     Poor      Exact     Good
Boundary problem    No        No        Yes       No        No
Implementation      Medium    Medium    Simple    Simple    Medium
Distributed         Easy      Medium    Easy      Hard      Medium
```

---

## Why Alternatives Were Rejected

### Alternative 1: Simple Counter with Reset Thread

```java
// Rejected
public class SimpleRateLimiter {
    private Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();
    
    public SimpleRateLimiter(int limit, long windowMs) {
        // Background thread to reset counters
        scheduler.scheduleAtFixedRate(() -> {
            counters.clear();
        }, windowMs, windowMs, TimeUnit.MILLISECONDS);
    }
}
```

**Why rejected:**
- All users reset at same time (thundering herd)
- Memory leak if many unique keys
- Background thread overhead

### Alternative 2: Database-Based Counting

```java
// Rejected
public class DatabaseRateLimiter {
    public boolean tryAcquire(String key) {
        int count = db.query("SELECT count FROM rate_limits WHERE key = ?", key);
        if (count < limit) {
            db.update("UPDATE rate_limits SET count = count + 1 WHERE key = ?", key);
            return true;
        }
        return false;
    }
}
```

**Why rejected:**
- Database becomes bottleneck
- High latency for every request
- Race conditions without proper locking

### Alternative 3: Single Global Lock

```java
// Rejected
public class GlobalLockRateLimiter {
    private final Object lock = new Object();
    
    public boolean tryAcquire(String key) {
        synchronized (lock) {
            // All rate limiting logic
        }
    }
}
```

**Why rejected:**
- Serializes all requests
- Terrible performance under load
- Single point of contention

---

## Thread Safety Analysis

### Token Bucket Thread Safety

```java
private static class Bucket {
    private double tokens;
    private long lastRefillTime;
    
    // Synchronized to prevent race conditions
    synchronized boolean tryConsume(int permits, double refillRatePerMs, int capacity) {
        refill(refillRatePerMs, capacity);
        
        if (tokens >= permits) {
            tokens -= permits;
            return true;
        }
        return false;
    }
}
```

**Why synchronized?**

```
Without synchronization:
Thread A: reads tokens = 5
Thread B: reads tokens = 5
Thread A: tokens >= 1? Yes, tokens = 4
Thread B: tokens >= 1? Yes, tokens = 4  // Should be 3!

Both threads consume tokens, but count is wrong.
```

### Fixed Window Lock-Free Approach

```java
private static class Window {
    final AtomicInteger count;
    
    boolean tryIncrement(int permits, int max) {
        while (true) {
            int current = count.get();
            if (current + permits > max) {
                return false;
            }
            if (count.compareAndSet(current, current + permits)) {
                return true;
            }
            // CAS failed, another thread modified, retry
        }
    }
}
```

**Why CAS instead of synchronized?**
- No blocking
- Better performance under contention
- Atomic operation at CPU level

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle clock skew in distributed systems?

**Answer:**

```java
// Use logical clocks or centralized time
public class ClockSyncRateLimiter implements RateLimiter {
    private final TimeService timeService;  // Centralized time
    
    @Override
    public boolean tryAcquire(String key) {
        long now = timeService.getCurrentTime();  // From central server
        // Use 'now' for all calculations
        // ... rest of logic
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

---

### Q2: How would you implement priority-based rate limiting?

**Answer:**

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

---

### Q3: How would you implement rate limiting with quotas?

**Answer:**

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
        if (quota.getRemaining() <= 0) {
            return false;
        }
        
        quota.consume(1);
        return true;
    }
}

class Quota {
    private final long total;
    private long used;
    private final long resetPeriodMs;
    private long resetTime;
    
    public boolean consume(long amount) {
        if (used + amount > total) {
            return false;
        }
        used += amount;
        return true;
    }
    
    public long getRemaining() {
        long now = System.currentTimeMillis();
        if (now >= resetTime) {
            used = 0;
            resetTime = now + resetPeriodMs;
        }
        return total - used;
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add distributed rate limiting** - Use Redis or distributed cache for multi-server scenarios
2. **Add adaptive rate limiting** - Dynamically adjust limits based on system load
3. **Add rate limit headers** - Return X-RateLimit-* headers to clients
4. **Add priority queues** - Prioritize requests based on user tiers
5. **Add rate limit analytics** - Track and report rate limit violations
6. **Add circuit breaker integration** - Combine with circuit breakers for resilience
7. **Add rate limit overrides** - Allow admin override for specific users/keys
8. **Add smooth rate limiting** - Distribute requests evenly over time window

---

## STEP 7: Complexity Analysis

### Time Complexity

| Algorithm | tryAcquire | getRemainingTokens |
|-----------|------------|-------------------|
| Token Bucket | O(1) | O(1) |
| Leaky Bucket | O(1) | O(1) |
| Fixed Window | O(1) | O(1) |
| Sliding Window Log | O(n)* | O(n) |
| Sliding Window Counter | O(1) | O(1) |

*n = number of requests in window (cleanup)

### Space Complexity

| Algorithm | Per Key | Total |
|-----------|---------|-------|
| Token Bucket | O(1) | O(k) |
| Leaky Bucket | O(1) | O(k) |
| Fixed Window | O(1) | O(k) |
| Sliding Window Log | O(n) | O(k * n) |
| Sliding Window Counter | O(1) | O(k) |

k = number of unique keys, n = requests per window

### Bottlenecks at Scale

**10x Usage (1K → 10K keys):**
- Problem: Memory usage grows linearly (O(k)), in-memory storage becomes significant, synchronized methods cause contention
- Solution: Use distributed rate limiter (Redis) for shared state, implement key-based sharding
- Tradeoff: Network latency added, requires external dependency (Redis)

**100x Usage (1K → 100K keys):**
- Problem: Single instance can't handle all keys, memory pressure, lock contention on shared structures
- Solution: Shard rate limiters by key hash across multiple instances, use distributed coordination (Redis cluster)
- Tradeoff: Higher infrastructure complexity, need consistent hashing and distributed coordination


### Q1: How would you handle rate limiting across multiple servers?

```java
// Use Redis for distributed state
public class DistributedTokenBucket implements RateLimiter {
    private final JedisPool jedisPool;
    
    private static final String LUA_SCRIPT = 
        "local tokens = tonumber(redis.call('get', KEYS[1]) or ARGV[1])\n" +
        "local lastRefill = tonumber(redis.call('get', KEYS[2]) or ARGV[2])\n" +
        "local now = tonumber(ARGV[2])\n" +
        "local refillRate = tonumber(ARGV[3])\n" +
        "local capacity = tonumber(ARGV[1])\n" +
        "\n" +
        "-- Refill tokens\n" +
        "local elapsed = now - lastRefill\n" +
        "tokens = math.min(capacity, tokens + elapsed * refillRate / 1000)\n" +
        "\n" +
        "if tokens >= 1 then\n" +
        "    tokens = tokens - 1\n" +
        "    redis.call('set', KEYS[1], tokens)\n" +
        "    redis.call('set', KEYS[2], now)\n" +
        "    return 1\n" +
        "else\n" +
        "    return 0\n" +
        "end";
    
    @Override
    public boolean tryAcquire(String key) {
        try (Jedis jedis = jedisPool.getResource()) {
            Object result = jedis.eval(
                LUA_SCRIPT,
                Arrays.asList(key + ":tokens", key + ":lastRefill"),
                Arrays.asList(
                    String.valueOf(capacity),
                    String.valueOf(System.currentTimeMillis()),
                    String.valueOf(refillRate)
                )
            );
            return "1".equals(result.toString());
        }
    }
}
```

### Q2: How would you implement different limits for different tiers?

```java
public class TieredRateLimiter implements RateLimiter {
    private final Map<Tier, RateLimiter> tierLimiters;
    private final Function<String, Tier> tierResolver;
    
    public TieredRateLimiter(Function<String, Tier> tierResolver) {
        this.tierResolver = tierResolver;
        this.tierLimiters = new EnumMap<>(Tier.class);
        
        tierLimiters.put(Tier.FREE, new TokenBucketRateLimiter(10, 1));
        tierLimiters.put(Tier.BASIC, new TokenBucketRateLimiter(100, 10));
        tierLimiters.put(Tier.PREMIUM, new TokenBucketRateLimiter(1000, 100));
        tierLimiters.put(Tier.ENTERPRISE, new TokenBucketRateLimiter(10000, 1000));
    }
    
    @Override
    public boolean tryAcquire(String key) {
        Tier tier = tierResolver.apply(key);
        return tierLimiters.get(tier).tryAcquire(key);
    }
}

public enum Tier {
    FREE, BASIC, PREMIUM, ENTERPRISE
}
```

### Q3: How would you add rate limit headers to responses?

```java
public class RateLimitInfo {
    private final int limit;
    private final int remaining;
    private final long resetTime;
    
    public Map<String, String> toHeaders() {
        return Map.of(
            "X-RateLimit-Limit", String.valueOf(limit),
            "X-RateLimit-Remaining", String.valueOf(remaining),
            "X-RateLimit-Reset", String.valueOf(resetTime)
        );
    }
}

public interface InspectableRateLimiter extends RateLimiter {
    RateLimitInfo getRateLimitInfo(String key);
}
```

### Q4: How would you handle graceful degradation?

```java
public class GracefulRateLimiter implements RateLimiter {
    private final RateLimiter primary;
    private final RateLimiter fallback;
    private final CircuitBreaker circuitBreaker;
    
    @Override
    public boolean tryAcquire(String key) {
        if (circuitBreaker.isOpen()) {
            // Primary is down, use fallback (more permissive)
            return fallback.tryAcquire(key);
        }
        
        try {
            boolean result = primary.tryAcquire(key);
            circuitBreaker.recordSuccess();
            return result;
        } catch (Exception e) {
            circuitBreaker.recordFailure();
            // Fail open - allow request if rate limiter is down
            return true;
        }
    }
}
```

