# Rate Limiting Deep Dive

## 0️⃣ Prerequisites

- Understanding of distributed systems basics (covered in Phase 1)
- Knowledge of caching concepts (covered in Phase 4)
- Understanding of Redis basics (covered in Phase 4)
- Basic familiarity with API design (covered in Phase 2)

**Quick refresher**: Redis is an in-memory data store used for caching and real-time operations. Rate limiting restricts the number of requests a client can make within a time window to prevent abuse and ensure fair resource usage.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

Without rate limiting, systems are vulnerable to:

1. **DDoS attacks**: Attackers overwhelm servers with millions of requests
2. **Resource exhaustion**: A few clients consume all available resources
3. **Cost explosion**: Malicious or buggy clients cause massive cloud bills
4. **Unfair usage**: Some users monopolize resources, degrading service for others
5. **API abuse**: Clients exceed quotas, violate terms of service

### What Systems Look Like Without Rate Limiting

Without rate limiting:

- **Single client sends 100,000 requests/second**: Server crashes, all users affected
- **Buggy mobile app**: Infinite retry loop sends millions of requests, racks up costs
- **Price scraping bots**: Scrape entire product catalog in seconds, impact legitimate users
- **API key sharing**: One key used by thousands, exceeding intended usage
- **Cascading failures**: One service overloaded causes others to fail

### Real Examples

**Example 1**: E-commerce site during sale. Price scraping bots send 10,000 requests/second. Result: Site becomes slow, legitimate users can't complete purchases, sales lost.

**Example 2**: API service with free tier. User shares API key publicly, thousands use it simultaneously. Result: API costs explode, free tier becomes unsustainable.

**Example 3**: Social media app. Bug in mobile app causes retry loop. Result: Single user sends 1 million requests in an hour, service degrades for all users.

## 2️⃣ Intuition and Mental Model

**Think of rate limiting like a nightclub bouncer:**

The bouncer (rate limiter) controls how many people (requests) can enter (be processed) per hour. Rules might be:
- **Fixed window**: "100 people per hour" - counter resets each hour
- **Sliding window**: "100 people in any 1-hour period" - tracks last hour continuously
- **Token bucket**: "People can enter at steady rate" - tokens refill gradually, people consume tokens

**The mental model**: Rate limiting is like a traffic light or valve that controls the flow of requests. It ensures no single client can overwhelm the system and resources are distributed fairly.

**Another analogy**: Think of rate limiting like a water faucet. Without rate limiting, the faucet is fully open - one user can drain all water. With rate limiting, the faucet has a flow restrictor - each user gets a fair share, and the total flow is controlled.

## 3️⃣ How it works internally

### Rate Limiting Algorithms

#### 1. Fixed Window Counter

**How it works**:
- Divide time into fixed windows (e.g., 1 hour)
- Count requests in current window
- Reset counter at window boundary

**Example**: 100 requests/hour
```
Time:  10:00     10:30     11:00     11:30
       |---------|---------|---------|
Window: [10-11)            [11-12)
Count:   85       100       45       80
        ✓        ✗         ✓        ✓
```

**Implementation**:
```java
// Key: user_id:10:00 (window identifier)
// Value: request count
redis.incr("rate_limit:user123:10:00");
if (count > 100) {
    reject();
}
```

**Limitation**: Traffic spike at window boundary
- At 10:59:59, user sends 100 requests (allowed)
- At 11:00:01, window resets, user sends 100 more requests
- Result: 200 requests in 2 seconds (violates intent)

#### 2. Sliding Window Log

**How it works**:
- Store timestamp of each request
- Count requests within last time window
- Remove old timestamps

**Example**: 100 requests/hour
```
Time: 10:00  10:30  11:00  11:30  12:00
      |------|------|------|------|
      
At 11:30, check last hour (10:30-11:30):
Timestamps: [10:35, 10:45, 11:00, 11:15, 11:25] = 5 requests
Result: ✓ Allowed (5 < 100)
```

**Implementation**:
```java
long now = System.currentTimeMillis();
long windowStart = now - 3600000; // 1 hour ago

// Add current request
redis.zadd("rate_limit:user123", now, UUID.randomUUID().toString());

// Count requests in window
long count = redis.zcount("rate_limit:user123", windowStart, now);

// Remove old requests
redis.zremrangeByScore("rate_limit:user123", 0, windowStart);

if (count > 100) {
    reject();
}
```

**Pros**: Accurate, handles boundary cases
**Cons**: Memory intensive (stores all timestamps)

#### 3. Sliding Window Counter

**How it works**:
- Divide time into smaller sub-windows
- Track count per sub-window
- Calculate weighted count across sliding window

**Example**: 100 requests/hour, sub-windows of 10 minutes
```
Time:  10:00   10:10   10:20   10:30   10:40   10:50   11:00
       |-------|-------|-------|-------|-------|-------|
Window: [10:00-10:10] [10:10-10:20] ... [10:50-11:00]

At 10:55 (55% through current window):
Count = 10:00-10:50 windows (full) + 10:50-11:00 window (55%)
      = 5 windows × 20 req/window + 0.55 × 20
      = 100 + 11 = 111 requests
```

**Implementation**:
```java
long now = System.currentTimeMillis();
long windowSize = 3600000; // 1 hour
long subWindowSize = 600000; // 10 minutes

long currentSubWindow = (now / subWindowSize) * subWindowSize;
long subWindowCount = redis.incr("rate_limit:user123:" + currentSubWindow);
redis.expire("rate_limit:user123:" + currentSubWindow, windowSize / 1000);

// Calculate weighted count across sliding window
long count = calculateWeightedCount(userId, now, windowSize, subWindowSize);

if (count > 100) {
    reject();
}
```

**Pros**: Memory efficient, reasonably accurate
**Cons**: More complex, approximate

#### 4. Token Bucket

**How it works**:
- Bucket holds tokens (capacity)
- Tokens refill at constant rate
- Request consumes one token
- Request allowed if token available

**Example**: 100 tokens, refill 10 tokens/second
```
Time:  0s     10s    20s    30s
Tokens: 100   100    100    100 (refilled continuously)
       |------|------|------|
Request: 50 requests consume 50 tokens
Tokens: 50 → 60 → 70 → ... → 100 (refills over time)
```

**Implementation**:
```java
String key = "rate_limit:user123";
long capacity = 100;
long refillRate = 10; // tokens per second
long refillPeriod = 1000; // 1 second in milliseconds

// Get current state
String data = redis.get(key);
TokenBucket bucket = parseBucket(data);

long now = System.currentTimeMillis();
long elapsed = now - bucket.lastRefill;
long tokensToAdd = (elapsed / refillPeriod) * refillRate;

bucket.tokens = Math.min(capacity, bucket.tokens + tokensToAdd);
bucket.lastRefill = now;

if (bucket.tokens >= 1) {
    bucket.tokens--;
    redis.set(key, serializeBucket(bucket));
    allow();
} else {
    reject();
}
```

**Pros**: Allows bursts, smooth rate limiting
**Cons**: More complex, stateful

#### 5. Leaky Bucket

**How it works**:
- Bucket holds requests (capacity)
- Requests leak (processed) at constant rate
- Request queued if bucket not full
- Request rejected if bucket full

**Example**: Capacity 100, leak rate 10 requests/second
```
Time:  0s     10s    20s
Queue: [100]  [90]   [80] (leaking at 10/sec)
       |------|------|
New request: Added to queue if < 100, rejected if >= 100
```

**Implementation**:
```java
String key = "rate_limit:user123";
long capacity = 100;
long leakRate = 10; // requests per second

long queueSize = redis.llen("rate_limit:user123:queue");
long now = System.currentTimeMillis();

// Leak requests
long lastLeak = redis.get("rate_limit:user123:last_leak");
long elapsed = now - lastLeak;
long leaked = (elapsed / 1000) * leakRate;

if (leaked > 0) {
    for (int i = 0; i < leaked && queueSize > 0; i++) {
        redis.lpop("rate_limit:user123:queue");
        queueSize--;
    }
    redis.set("rate_limit:user123:last_leak", now);
}

if (queueSize < capacity) {
    redis.rpush("rate_limit:user123:queue", now);
    allow();
} else {
    reject();
}
```

**Pros**: Smooth output rate, no bursts
**Cons**: Queues requests, more complex

## 4️⃣ Simulation-first explanation

### Scenario: API with 100 Requests/Hour Limit

**Start**: 1 API server, 1 Redis instance, need to limit users to 100 requests/hour

**Fixed Window Counter**:

```
User makes request at 10:30
1. Check current window: "10:00-11:00"
2. Key: "rate_limit:user123:10:00"
3. Increment: redis.incr(key) → 1
4. Check: 1 <= 100? Yes → Allow

User makes 99 more requests
5. Increment: redis.incr(key) → 100
6. Check: 100 <= 100? Yes → Allow

User makes 1 more request
7. Increment: redis.incr(key) → 101
8. Check: 101 <= 100? No → Reject (429 Too Many Requests)

At 11:00, window resets
9. Key changes to "rate_limit:user123:11:00"
10. Count resets to 0
```

**Token Bucket**:

```
Initial state: 100 tokens, last refill = 10:00:00

User makes request at 10:30:00
1. Calculate tokens to add: (30 min = 1800s) × (10 tokens/sec) = 18,000 tokens
2. But capacity is 100, so tokens = min(100, 0 + 18,000) = 100
3. Consume 1 token: tokens = 99
4. Allow request

User makes 98 more requests quickly
5. Tokens: 99 → 98 → ... → 1
6. All allowed (tokens available)

User makes 1 more request
7. Tokens: 1 → 0
8. Allow request

User makes 1 more request immediately
9. Tokens: 0 (no time to refill)
10. Reject request (429)

After 0.1 seconds
11. Tokens refilled: 0 + (0.1s × 10 tokens/sec) = 1 token
12. Next request allowed
```

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. API Gateway Rate Limiting

**Kong/API Gateway**: Rate limiting at gateway level
- Global limits (all users)
- Per-user limits (API key based)
- Per-IP limits (DDoS protection)
- Tiered limits (free vs paid tiers)

#### 2. Distributed Rate Limiting with Redis

**Twitter's approach**: Distributed rate limiting
- All API servers share Redis
- Consistent limits across servers
- High availability (Redis Cluster)
- Sliding window algorithm

#### 3. Tiered Rate Limiting

**Stripe's API**: Different limits for different tiers
- Free tier: 100 requests/hour
- Paid tier: 10,000 requests/hour
- Enterprise: Custom limits

#### 4. Adaptive Rate Limiting

**Netflix's adaptive limiting**: Limits adjust based on system load
- Normal load: Standard limits
- High load: Stricter limits
- Low load: Relaxed limits

### Production Implementation

**Rate limiting middleware**:
```java
@Component
public class RateLimitingInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        String userId = extractUserId(request);
        RateLimitResult result = rateLimiter.checkLimit(userId);
        
        if (!result.isAllowed()) {
            response.setStatus(429);
            response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setHeader("X-RateLimit-Reset", 
                String.valueOf(result.getResetTime()));
            return false;
        }
        
        response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        response.setHeader("X-RateLimit-Remaining", 
            String.valueOf(result.getRemaining()));
        response.setHeader("X-RateLimit-Reset", 
            String.valueOf(result.getResetTime()));
        return true;
    }
}
```

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: Token Bucket Rate Limiter

**Token bucket implementation**:
```java
@Service
public class TokenBucketRateLimiter {
    
    @Autowired
    private StringRedisTemplate redis;
    
    private static final String RATE_LIMIT_KEY_PREFIX = "rate_limit:";
    private static final long DEFAULT_CAPACITY = 100;
    private static final long DEFAULT_REFILL_RATE = 10; // tokens per second
    
    public RateLimitResult checkLimit(String userId, long capacity, long refillRate) {
        String key = RATE_LIMIT_KEY_PREFIX + userId;
        long now = System.currentTimeMillis();
        
        // Get or create bucket
        TokenBucket bucket = getOrCreateBucket(key, capacity);
        
        // Refill tokens
        long elapsed = now - bucket.getLastRefill();
        long tokensToAdd = (elapsed / 1000) * refillRate;
        long newTokens = Math.min(capacity, bucket.getTokens() + tokensToAdd);
        
        // Check if request allowed
        if (newTokens >= 1) {
            newTokens--;
            bucket.setTokens(newTokens);
            bucket.setLastRefill(now);
            saveBucket(key, bucket);
            
            return RateLimitResult.allowed(capacity, newTokens, 
                calculateResetTime(now, refillRate, capacity, newTokens));
        } else {
            return RateLimitResult.denied(capacity, 0, 
                calculateResetTime(now, refillRate, capacity, 0));
        }
    }
    
    private TokenBucket getOrCreateBucket(String key, long capacity) {
        String data = redis.opsForValue().get(key);
        if (data == null) {
            return new TokenBucket(capacity, System.currentTimeMillis());
        }
        return deserializeBucket(data);
    }
    
    private void saveBucket(String key, TokenBucket bucket) {
        redis.opsForValue().set(key, serializeBucket(bucket), 
            Duration.ofHours(24)); // Expire after 24 hours
    }
    
    private long calculateResetTime(long now, long refillRate, 
                                   long capacity, long currentTokens) {
        if (currentTokens >= capacity) {
            return now + 1000; // Already full
        }
        long tokensNeeded = capacity - currentTokens;
        long secondsToRefill = (tokensNeeded + refillRate - 1) / refillRate;
        return now + (secondsToRefill * 1000);
    }
}

@Data
class TokenBucket {
    private long tokens;
    private long lastRefill;
    
    TokenBucket(long tokens, long lastRefill) {
        this.tokens = tokens;
        this.lastRefill = lastRefill;
    }
    
    // Serialization/deserialization methods
    String serialize() {
        return tokens + ":" + lastRefill;
    }
    
    static TokenBucket deserialize(String data) {
        String[] parts = data.split(":");
        return new TokenBucket(Long.parseLong(parts[0]), Long.parseLong(parts[1]));
    }
}
```

#### Example 2: Sliding Window Counter

**Sliding window implementation**:
```java
@Service
public class SlidingWindowRateLimiter {
    
    @Autowired
    private StringRedisTemplate redis;
    
    private static final String RATE_LIMIT_KEY_PREFIX = "rate_limit:";
    private static final long WINDOW_SIZE = 3600000; // 1 hour
    private static final long SUB_WINDOW_SIZE = 600000; // 10 minutes
    
    public RateLimitResult checkLimit(String userId, long limit) {
        long now = System.currentTimeMillis();
        long currentSubWindow = (now / SUB_WINDOW_SIZE) * SUB_WINDOW_SIZE;
        
        // Increment current sub-window
        String subWindowKey = RATE_LIMIT_KEY_PREFIX + userId + ":" + currentSubWindow;
        Long count = redis.opsForValue().increment(subWindowKey);
        redis.expire(subWindowKey, Duration.ofMillis(WINDOW_SIZE));
        
        // Calculate weighted count across sliding window
        long windowStart = now - WINDOW_SIZE;
        long weightedCount = calculateWeightedCount(userId, now, windowStart);
        
        if (weightedCount <= limit) {
            return RateLimitResult.allowed(limit, limit - weightedCount, 
                currentSubWindow + SUB_WINDOW_SIZE);
        } else {
            return RateLimitResult.denied(limit, 0, 
                findNextAvailableTime(userId, now, limit));
        }
    }
    
    private long calculateWeightedCount(String userId, long now, long windowStart) {
        long totalCount = 0;
        long subWindowStart = (windowStart / SUB_WINDOW_SIZE) * SUB_WINDOW_SIZE;
        
        for (long subWindow = subWindowStart; subWindow <= now; 
             subWindow += SUB_WINDOW_SIZE) {
            String key = RATE_LIMIT_KEY_PREFIX + userId + ":" + subWindow;
            Long count = redis.opsForValue().get(key);
            if (count != null) {
                // Weight by how much of sub-window is in sliding window
                long subWindowEnd = subWindow + SUB_WINDOW_SIZE;
                long overlapStart = Math.max(subWindow, windowStart);
                long overlapEnd = Math.min(subWindowEnd, now);
                double weight = (overlapEnd - overlapStart) / (double) SUB_WINDOW_SIZE;
                
                totalCount += (long) (count * weight);
            }
        }
        
        return totalCount;
    }
}
```

#### Example 3: Spring Boot Rate Limiting with Bucket4j

**Maven dependency**:
```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.7.0</version>
</dependency>
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-redis</artifactId>
    <version>8.7.0</version>
</dependency>
```

**Rate limiter service**:
```java
@Service
public class Bucket4jRateLimiter {
    
    private final ProxyManager<String> buckets;
    
    public Bucket4jRateLimiter(ProxyManager<String> buckets) {
        this.buckets = buckets;
    }
    
    public RateLimitResult checkLimit(String userId, long capacity, long refillRate) {
        BucketConfiguration config = BucketConfiguration.builder()
            .addLimit(Bandwidth.classic(capacity, 
                Refill.intervally(refillRate, Duration.ofSeconds(1))))
            .build();
        
        Bucket bucket = buckets.getProxy(userId, () -> config);
        
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        
        if (probe.isConsumed()) {
            return RateLimitResult.allowed(
                capacity, 
                probe.getRemainingTokens(),
                System.currentTimeMillis() + probe.getNanosToWaitForRefill() / 1_000_000
            );
        } else {
            return RateLimitResult.denied(
                capacity,
                0,
                System.currentTimeMillis() + probe.getNanosToWaitForRefill() / 1_000_000
            );
        }
    }
}
```

**Redis proxy manager configuration**:
```java
@Configuration
public class RateLimitingConfig {
    
    @Bean
    public ProxyManager<String> proxyManager(RedisConnectionFactory connectionFactory) {
        return Bucket4j.extension(Redis.class)
            .proxyManagerForRedis(connectionFactory);
    }
}
```

#### Example 4: Rate Limiting Annotation

**Custom annotation**:
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    long capacity();
    long refillRate(); // tokens per second
    String key() default ""; // SpEL expression for rate limit key
}
```

**Aspect for rate limiting**:
```java
@Aspect
@Component
public class RateLimitAspect {
    
    @Autowired
    private TokenBucketRateLimiter rateLimiter;
    
    @Around("@annotation(rateLimit)")
    public Object rateLimit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) 
            throws Throwable {
        String key = evaluateKey(joinPoint, rateLimit.key());
        RateLimitResult result = rateLimiter.checkLimit(
            key, rateLimit.capacity(), rateLimit.refillRate());
        
        if (!result.isAllowed()) {
            throw new RateLimitExceededException(
                "Rate limit exceeded. Try again after " + 
                (result.getResetTime() - System.currentTimeMillis()) / 1000 + " seconds");
        }
        
        return joinPoint.proceed();
    }
    
    private String evaluateKey(ProceedingJoinPoint joinPoint, String keyExpression) {
        if (keyExpression.isEmpty()) {
            return "default";
        }
        // Evaluate SpEL expression
        // Implementation depends on SpEL evaluation
        return keyExpression;
    }
}
```

**Usage**:
```java
@RestController
public class ApiController {
    
    @GetMapping("/api/data")
    @RateLimit(capacity = 100, refillRate = 10, key = "#userId")
    public ResponseEntity<Data> getData(@RequestParam String userId) {
        return ResponseEntity.ok(getData(userId));
    }
}
```

#### Example 5: Distributed Rate Limiting

**Redis-based distributed rate limiter**:
```java
@Service
public class DistributedRateLimiter {
    
    @Autowired
    private StringRedisTemplate redis;
    
    private static final String RATE_LIMIT_SCRIPT = 
        "local key = KEYS[1]\n" +
        "local limit = tonumber(ARGV[1])\n" +
        "local window = tonumber(ARGV[2])\n" +
        "local current = redis.call('INCR', key)\n" +
        "if current == 1 then\n" +
        "    redis.call('EXPIRE', key, window)\n" +
        "end\n" +
        "if current > limit then\n" +
        "    return {0, current, limit}\n" +
        "else\n" +
        "    return {1, current, limit}\n" +
        "end";
    
    public RateLimitResult checkLimit(String userId, long limit, long windowSeconds) {
        String key = "rate_limit:" + userId;
        
        List<Long> result = redis.execute(
            new DefaultRedisScript<>(RATE_LIMIT_SCRIPT, List.class),
            Collections.singletonList(key),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        
        boolean allowed = result.get(0) == 1;
        long current = result.get(1);
        long limitValue = result.get(2);
        
        if (allowed) {
            return RateLimitResult.allowed(limitValue, limitValue - current, 
                System.currentTimeMillis() + (windowSeconds * 1000));
        } else {
            Long ttl = redis.getExpire(key);
            return RateLimitResult.denied(limitValue, 0, 
                System.currentTimeMillis() + (ttl * 1000));
        }
    }
}
```

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Algorithm Choice: Accuracy vs Memory vs Complexity
- **Fixed window**: Simple, memory efficient, but inaccurate at boundaries
- **Sliding window log**: Accurate, but memory intensive
- **Sliding window counter**: Good balance, but more complex
- **Token bucket**: Allows bursts, smooth limiting, but more complex
- **Leaky bucket**: Smooth output, no bursts, but queues requests

#### 2. Strictness vs User Experience
- **Strict limits**: Better resource protection, but users hit limits more often
- **Soft limits**: Better UX, but less protection

#### 3. Global vs Per-User Limits
- **Global limits**: Simple, but unfair (one user can consume all)
- **Per-user limits**: Fair, but more complex

### Common Pitfalls

#### Pitfall 1: Race Conditions in Distributed Systems
**Problem**: Multiple servers increment counter simultaneously
```java
// Bad: Race condition
long count = redis.get(key);
count++;
redis.set(key, count);
if (count > limit) reject();
```

**Solution**: Use atomic operations (INCR) or Lua scripts
```java
// Good: Atomic operation
long count = redis.incr(key);
if (count > limit) reject();
```

#### Pitfall 2: Fixed Window Boundary Issue
**Problem**: Users can send 2× limit at window boundary
**Solution**: Use sliding window or token bucket

#### Pitfall 3: Not Handling Redis Failures
**Problem**: If Redis is down, rate limiting fails
**Solution**: Fail open (allow all) or fail closed (reject all) with circuit breaker

#### Pitfall 4: Memory Leaks
**Problem**: Rate limit keys never expire, Redis memory fills
**Solution**: Always set TTL on keys

#### Pitfall 5: Inconsistent Limits Across Servers
**Problem**: Different servers use different algorithms or configurations
**Solution**: Centralized configuration, shared Redis

### Performance Gotchas

#### Gotcha 1: Redis Latency
**Problem**: Every request hits Redis, adds latency
**Solution**: Use local cache with Redis as source of truth, or async rate limiting

#### Gotcha 2: Key Cardinality
**Problem**: Too many unique keys (per IP, per user, etc.) exhausts Redis memory
**Solution**: Use hierarchical keys, aggregate limits, or sampling

## 8️⃣ When NOT to use this

### When Rate Limiting Isn't Needed

1. **Internal services**: Services behind firewall, trusted clients

2. **Low-traffic APIs**: APIs with predictable, low traffic

3. **Read-only public data**: Public data that's cached, no abuse risk

### Anti-Patterns

1. **Over-limiting**: Too strict limits hurt legitimate users

2. **Under-limiting**: Limits too high, doesn't prevent abuse

3. **No monitoring**: Not tracking rate limit hits, can't tune limits

## 9️⃣ Comparison with Alternatives

### Rate Limiting vs Throttling

**Rate Limiting**: Hard limit, requests rejected when exceeded
- Pros: Predictable, prevents abuse
- Cons: User experience impact

**Throttling**: Soft limit, requests queued/delayed when exceeded
- Pros: Better UX, no rejections
- Cons: More complex, requires queuing

**Best practice**: Use rate limiting for API protection, throttling for fair resource sharing

### Rate Limiting vs Quotas

**Rate Limiting**: Per-time-window limits (e.g., 100/hour)
- Pros: Prevents bursts, smooth traffic
- Cons: Doesn't limit total usage

**Quotas**: Total usage limits (e.g., 10,000/month)
- Pros: Controls total cost/usage
- Cons: Doesn't prevent bursts

**Best practice**: Use both - rate limiting for traffic shaping, quotas for cost control

### In-Memory vs Distributed Rate Limiting

**In-memory**: Each server has its own limits
- Pros: No network latency, simple
- Cons: Inconsistent limits, not accurate across servers

**Distributed (Redis)**: Shared limits across servers
- Pros: Consistent, accurate
- Cons: Network latency, Redis dependency

**Best practice**: Use distributed for accurate limits, in-memory as fallback

## 🔟 Interview follow-up questions WITH answers

### Question 1: How do you handle rate limiting in a distributed system?

**Answer**:
**Challenges**:
- Multiple servers need shared state
- Consistency across servers
- Redis availability

**Solutions**:

1. **Redis-based distributed rate limiting**:
   - All servers share Redis
   - Atomic operations (INCR, Lua scripts)
   - Consistent limits across all servers

2. **Hierarchical rate limiting**:
   - Global limits in Redis
   - Per-server limits in memory
   - Hybrid approach

3. **Consistent hashing**:
   - Route same user to same server
   - In-memory rate limiting per server
   - Less accurate but faster

**Example**:
```java
// Distributed rate limiter using Redis
long count = redis.incr("rate_limit:" + userId);
if (count == 1) {
    redis.expire("rate_limit:" + userId, 3600); // 1 hour
}
if (count > limit) {
    reject();
}
```

### Question 2: What happens when Redis is down?

**Answer**:
**Strategies**:

1. **Fail open**: Allow all requests
   - Pros: Service continues working
   - Cons: No protection during Redis outage

2. **Fail closed**: Reject all requests
   - Pros: Maximum protection
   - Cons: Service unavailable

3. **Local fallback**: Use in-memory rate limiting
   - Pros: Some protection, service continues
   - Cons: Limits reset on server restart, inconsistent across servers

4. **Circuit breaker**: Fail open after threshold
   - Pros: Balanced approach
   - Cons: More complex

**Best practice**: Fail open with monitoring/alerting, or use local fallback with higher limits

### Question 3: How do you choose between different rate limiting algorithms?

**Answer**:
**Decision factors**:

1. **Accuracy requirements**:
   - Need exact limits? → Sliding window log
   - Approximate OK? → Fixed window or sliding window counter

2. **Memory constraints**:
   - Limited memory? → Fixed window or token bucket
   - Memory available? → Sliding window log

3. **Burst tolerance**:
   - Allow bursts? → Token bucket
   - No bursts? → Leaky bucket or sliding window

4. **Complexity tolerance**:
   - Simple needed? → Fixed window
   - Can handle complexity? → Token bucket or sliding window

**Typical choices**:
- **API rate limiting**: Token bucket (allows bursts, smooth)
- **DDoS protection**: Fixed window (simple, fast)
- **Fair resource sharing**: Leaky bucket (smooth output)
- **Exact limits needed**: Sliding window log

### Question 4: How do you implement tiered rate limiting (different limits for different users)?

**Answer**:
**Implementation**:

1. **User metadata lookup**:
```java
UserTier tier = userService.getTier(userId);
RateLimitConfig config = getConfigForTier(tier);
RateLimitResult result = rateLimiter.checkLimit(userId, config);
```

2. **Key-based routing**:
```java
String key = "rate_limit:" + tier + ":" + userId;
// Different keys for different tiers
```

3. **Configuration-driven**:
```java
@Configuration
public class RateLimitConfig {
    public Map<Tier, RateLimitConfig> getConfigs() {
        return Map.of(
            Tier.FREE, new RateLimitConfig(100, 3600), // 100/hour
            Tier.PAID, new RateLimitConfig(10000, 3600), // 10k/hour
            Tier.ENTERPRISE, new RateLimitConfig(Long.MAX_VALUE, 3600) // Unlimited
        );
    }
}
```

4. **Dynamic limits**: Limits can change based on user behavior, system load, etc.

### Question 5: How would you implement rate limiting that allows bursts but prevents sustained abuse?

**Answer**:
**Solution: Token bucket with multiple limits**:

1. **Short-term bucket**: Allows bursts (e.g., 100 tokens, refill 10/sec)
2. **Long-term bucket**: Prevents sustained abuse (e.g., 1000 tokens, refill 1/sec)

**Implementation**:
```java
public RateLimitResult checkLimit(String userId) {
    // Check short-term limit (allows bursts)
    RateLimitResult shortTerm = shortTermLimiter.checkLimit(userId);
    if (!shortTerm.isAllowed()) {
        return RateLimitResult.denied("Burst limit exceeded");
    }
    
    // Check long-term limit (prevents sustained abuse)
    RateLimitResult longTerm = longTermLimiter.checkLimit(userId);
    if (!longTerm.isAllowed()) {
        return RateLimitResult.denied("Sustained rate limit exceeded");
    }
    
    return RateLimitResult.allowed();
}
```

**Alternative: Sliding window with multiple windows**:
- 100 requests in 1 minute (burst limit)
- 1000 requests in 1 hour (sustained limit)

### Question 6 (L5/L6): How would you design a rate limiting system that handles millions of users with sub-millisecond latency?

**Answer**:
**Design considerations**:

1. **Caching layer**:
   - Local cache (Caffeine) for hot users
   - Redis for distributed state
   - Cache hit rate should be > 90%

2. **Sharding**:
   - Shard rate limit data by user ID
   - Consistent hashing for distribution
   - Reduces Redis key space per shard

3. **Async processing**:
   - Check limits asynchronously when possible
   - Pre-warm cache for known users
   - Background refresh of limits

4. **Approximate algorithms**:
   - Use probabilistic data structures (Count-Min Sketch)
   - Trade accuracy for performance
   - Good enough for rate limiting

5. **Hardware optimization**:
   - Redis on fast SSDs (NVMe)
   - Keep hot data in memory
   - Network optimization (low latency)

**Architecture**:
```
Request → Local Cache → Redis Cluster (sharded) → Response
           ↓ (miss)
        Redis Cluster → Update Cache → Response
```

**Monitoring**:
- P99 latency < 1ms
- Cache hit rate > 90%
- Redis CPU < 70%
- Rate limit accuracy monitoring

## 1️⃣1️⃣ One clean mental summary

Rate limiting controls request flow to prevent abuse and ensure fair resource usage. Key algorithms include fixed window (simple but inaccurate at boundaries), sliding window (accurate but memory intensive), token bucket (allows bursts, smooth limiting), and leaky bucket (smooth output, no bursts). Distributed rate limiting requires shared state (typically Redis) with atomic operations to prevent race conditions. Critical considerations include algorithm choice (accuracy vs memory vs complexity), handling Redis failures (fail open/closed/local fallback), and balancing strictness with user experience. Always set TTLs on rate limit keys to prevent memory leaks, use atomic Redis operations to avoid race conditions, and monitor rate limit effectiveness to tune limits appropriately.

