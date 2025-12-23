# 🚦 Rate Limiter - Design Explanation

## SOLID Principles Analysis

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

## Complexity Analysis

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

---

## Interview Follow-ups

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

