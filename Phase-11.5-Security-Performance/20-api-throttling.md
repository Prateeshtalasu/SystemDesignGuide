# API Throttling

## 0️⃣ Prerequisites

- Understanding of rate limiting (covered in topic 19)
- Knowledge of API design principles (covered in Phase 2)
- Understanding of distributed systems (covered in Phase 1)
- Basic familiarity with Redis (covered in Phase 4)

**Quick refresher**: Rate limiting sets hard limits and rejects requests when exceeded. API throttling is a form of rate limiting that controls request flow to prevent system overload while maintaining service availability. Rate limiting focuses on preventing abuse, while throttling focuses on protecting system resources.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

API throttling solves problems that pure rate limiting doesn't address:

1. **System overload protection**: Even legitimate users can overwhelm servers if too many requests arrive simultaneously
2. **Fair resource allocation**: Ensures all users get fair share of resources, not just preventing abuse
3. **Graceful degradation**: System continues working under load, just slower, rather than rejecting all requests
4. **Cost control**: Prevents runaway costs from legitimate but excessive usage
5. **Quality of service**: Maintains consistent performance for all users

### What Systems Look Like Without Throttling

Without throttling:

- **Traffic spike during peak hours**: All legitimate users experience slow responses
- **One power user**: Single user with high legitimate usage degrades service for everyone
- **Cascading failures**: High load causes timeouts, which trigger retries, worsening the problem
- **No fair sharing**: Users who send requests faster get more resources
- **All-or-nothing**: System either works perfectly or fails completely

### Real Examples

**Example 1**: E-commerce API during Black Friday. Millions of legitimate users checking prices. Without throttling, all users get slow responses or timeouts. With throttling, requests are queued and processed fairly, maintaining reasonable response times for all.

**Example 2**: Data analytics API. One customer runs large batch jobs that consume significant resources. Without throttling, this blocks other customers. With throttling, large jobs are rate-limited, allowing other customers to access the API.

**Example 3**: Public API with free tier. Free tier users share capacity. Without throttling, aggressive users consume all capacity, blocking others. With throttling, each user gets fair share, ensuring all can use the service.

## 2️⃣ Intuition and Mental Model

**Think of API throttling like a restaurant reservation system:**

The restaurant (API) has limited tables (resources). Without throttling, it's first-come-first-served with no limits - a large group could take all tables. With throttling:
- **Per-user limits**: Each customer can reserve 2 tables max (user-level throttling)
- **Tiered limits**: VIP customers get 5 tables, regular customers get 2 (tiered throttling)
- **Time-based**: Reservations spread throughout the evening (burst vs sustained)
- **Fair sharing**: When full, customers wait in queue rather than being turned away (graceful degradation)

**The mental model**: Throttling is like a traffic cop directing traffic. Instead of closing the road (rejecting requests), the cop controls flow (throttling) so everyone moves forward, just at a controlled pace.

**Another analogy**: Think of throttling like a water park slide. There's a lifeguard (throttler) at the top controlling how often people can go down. Instead of letting everyone rush at once (causing backups and accidents), the lifeguard spaces people out. Everyone gets to go, just in an orderly fashion.

## 3️⃣ How it works internally

### Throttling Mechanisms

#### 1. Request Queuing

**How it works**:
- Requests that exceed limit are queued instead of rejected
- Queue processed at controlled rate
- Requests wait and are processed when capacity available

**Example**: 100 requests/second limit
```
Request arrives → Check current rate → If under limit: Process immediately
                                   → If over limit: Add to queue
                                   → Process queue at 100 req/sec
```

**Implementation**:
```java
Queue<Request> requestQueue = new ConcurrentLinkedQueue<>();
RateLimiter rateLimiter = RateLimiter.create(100.0); // 100 requests/second

public void handleRequest(Request request) {
    if (rateLimiter.tryAcquire()) {
        processRequest(request); // Process immediately
    } else {
        requestQueue.offer(request); // Queue for later
    }
}

// Background thread processes queue
@Scheduled(fixedRate = 10) // Check every 10ms
public void processQueue() {
    while (!requestQueue.isEmpty() && rateLimiter.tryAcquire()) {
        Request request = requestQueue.poll();
        processRequest(request);
    }
}
```

#### 2. Token Bucket Throttling

**How it works**:
- Bucket holds tokens representing processing capacity
- Tokens refill at constant rate
- Request consumes token, processed if token available
- Requests wait if no tokens (throttled)

**Example**: 100 tokens, refill 10/second
```
Tokens available: 50
Request arrives → Consume 1 token → Process (49 remaining)
Request arrives → Consume 1 token → Process (48 remaining)
...
Requests arrive → No tokens → Wait for refill → Process when tokens available
```

#### 3. Leaky Bucket Throttling

**How it works**:
- Bucket holds requests
- Requests leak (processed) at constant rate
- New requests added to bucket if space available
- Requests rejected if bucket full

**Example**: Capacity 1000, leak rate 100/second
```
Bucket: [request1, request2, ..., request1000] (full)
Processing: 100 requests/second
New request arrives → Bucket full → Reject
After 1 second → 100 requests processed → Space available → Accept new requests
```

### Throttling Strategies

#### 1. User-Level Throttling

**Per-user limits**: Each user has individual throttle limit
- User A: 100 requests/minute
- User B: 100 requests/minute
- Independent limits, don't affect each other

#### 2. API Key-Based Throttling

**Per-API-key limits**: Different keys have different limits
- Free tier key: 100 requests/hour
- Paid tier key: 10,000 requests/hour
- Enterprise key: 100,000 requests/hour

#### 3. Tiered Throttling

**Multiple limit levels**: Different limits for different user tiers
- Free: 100/hour
- Basic: 1,000/hour
- Pro: 10,000/hour
- Enterprise: Custom limits

#### 4. Burst vs Sustained Throttling

**Burst limits**: Allow short bursts, throttle sustained high rate
- Burst: 100 requests in 1 minute (allowed)
- Sustained: 10,000 requests in 1 hour (throttled to average)

## 4️⃣ Simulation-first explanation

### Scenario: API with 100 Requests/Minute Per User

**Start**: 1 API server, need to throttle each user to 100 requests/minute

**User sends 150 requests in 1 minute**:

**Without throttling (rate limiting)**:
```
Requests 1-100: Processed (allowed)
Requests 101-150: Rejected (429 Too Many Requests)
Result: 50 requests failed
```

**With throttling**:
```
Requests 1-100: Processed immediately (within limit)
Requests 101-150: Queued (exceed limit)
Minute 1: Process 100 requests
Minute 2: Process remaining 50 requests from queue
Result: All requests processed, just delayed
```

### Scenario: Tiered Throttling

**Free tier user**:
```
Limit: 100 requests/hour
Requests: 150 requests in 1 hour
Result: First 100 processed immediately, next 50 queued/throttled
```

**Paid tier user**:
```
Limit: 10,000 requests/hour
Requests: 150 requests in 1 hour
Result: All 150 processed immediately (well under limit)
```

### Scenario: Burst vs Sustained

**Burst throttling**:
```
User sends 200 requests in 10 seconds (burst)
Burst limit: 100 requests/minute
Sustained limit: 100 requests/minute average

Time 0-10s: 200 requests arrive
- First 100: Processed immediately
- Next 100: Queued
Time 10-60s: Process queued requests at controlled rate
Result: Burst allowed but throttled to sustainable rate
```

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. API Gateway Throttling

**Kong/API Gateway**: Throttling at gateway level
- Global throttling (all traffic)
- Per-consumer throttling (per user)
- Per-route throttling (different endpoints)
- Plugin-based, configurable

#### 2. Cloud Provider Throttling

**AWS API Gateway**: Built-in throttling
- Account-level throttling
- Per-method throttling
- Burst limits and sustained limits
- Automatic scaling

#### 3. Application-Level Throttling

**Custom throttling middleware**:
- Throttle based on user ID, API key, IP
- Queue requests instead of rejecting
- Graceful degradation under load
- Monitoring and alerting

#### 4. Distributed Throttling

**Redis-based throttling**:
- Shared state across servers
- Consistent throttling
- High availability
- Real-time limit tracking

### Production Workflow

**Step 1: Define throttling requirements**
- Identify resource constraints
- Determine fair usage limits
- Define user tiers
- Set burst vs sustained limits

**Step 2: Implement throttling**
- Choose throttling algorithm
- Integrate with API gateway or application
- Configure limits per tier
- Implement queuing if needed

**Step 3: Monitor and tune**
- Track throttle events
- Monitor queue depths
- Adjust limits based on usage
- Alert on unusual patterns

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: User-Level Throttling

**Throttling service**:
```java
@Service
public class UserThrottlingService {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    private static final String THROTTLE_KEY_PREFIX = "throttle:user:";
    private static final long DEFAULT_LIMIT = 100;
    private static final long WINDOW_SECONDS = 60;
    
    public ThrottleResult checkThrottle(String userId, long limit) {
        String key = THROTTLE_KEY_PREFIX + userId;
        long now = System.currentTimeMillis();
        
        // Get current count
        String countStr = redis.opsForValue().get(key);
        long count = countStr != null ? Long.parseLong(countStr) : 0;
        
        if (count < limit) {
            // Under limit, allow and increment
            redis.opsForValue().increment(key);
            redis.expire(key, Duration.ofSeconds(WINDOW_SECONDS));
            
            return ThrottleResult.allowed(limit, limit - count - 1);
        } else {
            // Over limit, throttle
            Long ttl = redis.getExpire(key);
            return ThrottleResult.throttled(limit, 0, 
                System.currentTimeMillis() + (ttl * 1000));
        }
    }
}
```

**Request queuing**:
```java
@Service
public class ThrottledRequestHandler {
    
    @Autowired
    private UserThrottlingService throttlingService;
    
    private final Map<String, Queue<ThrottledRequest>> userQueues = 
        new ConcurrentHashMap<>();
    
    public CompletableFuture<Response> handleRequest(String userId, Request request) {
        ThrottleResult result = throttlingService.checkThrottle(userId, 100);
        
        if (result.isAllowed()) {
            // Process immediately
            return processRequest(request);
        } else {
            // Queue for throttled processing
            CompletableFuture<Response> future = new CompletableFuture<>();
            ThrottledRequest throttledRequest = new ThrottledRequest(request, future);
            
            userQueues.computeIfAbsent(userId, k -> new ConcurrentLinkedQueue<>())
                .offer(throttledRequest);
            
            return future;
        }
    }
    
    @Scheduled(fixedRate = 100) // Process every 100ms
    public void processThrottledRequests() {
        userQueues.forEach((userId, queue) -> {
            ThrottleResult result = throttlingService.checkThrottle(userId, 100);
            if (result.isAllowed() && !queue.isEmpty()) {
                ThrottledRequest request = queue.poll();
                if (request != null) {
                    processRequest(request.getRequest())
                        .whenComplete((response, error) -> {
                            if (error != null) {
                                request.getFuture().completeExceptionally(error);
                            } else {
                                request.getFuture().complete(response);
                            }
                        });
                }
            }
        });
    }
}
```

#### Example 2: API Key-Based Throttling

**API key throttling service**:
```java
@Service
public class ApiKeyThrottlingService {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @Autowired
    private ApiKeyRepository apiKeyRepository;
    
    private static final String THROTTLE_KEY_PREFIX = "throttle:apikey:";
    
    public ThrottleResult checkThrottle(String apiKey) {
        // Get API key details (with caching)
        ApiKey key = apiKeyRepository.findByKey(apiKey);
        if (key == null) {
            return ThrottleResult.invalidKey();
        }
        
        // Get throttle limits for tier
        ThrottleConfig config = getThrottleConfig(key.getTier());
        
        String throttleKey = THROTTLE_KEY_PREFIX + apiKey;
        long now = System.currentTimeMillis();
        
        // Check both burst and sustained limits
        ThrottleResult burstResult = checkBurstLimit(throttleKey, config);
        if (!burstResult.isAllowed()) {
            return burstResult;
        }
        
        ThrottleResult sustainedResult = checkSustainedLimit(throttleKey, config);
        if (!sustainedResult.isAllowed()) {
            return sustainedResult;
        }
        
        return ThrottleResult.allowed(config.getLimit(), 
            config.getLimit() - getCurrentCount(throttleKey));
    }
    
    private ThrottleResult checkBurstLimit(String key, ThrottleConfig config) {
        String burstKey = key + ":burst";
        long count = redis.opsForValue().increment(burstKey);
        if (count == 1) {
            redis.expire(burstKey, Duration.ofSeconds(60)); // 1 minute
        }
        
        if (count > config.getBurstLimit()) {
            return ThrottleResult.throttled("Burst limit exceeded");
        }
        
        return ThrottleResult.allowed();
    }
    
    private ThrottleResult checkSustainedLimit(String key, ThrottleConfig config) {
        String sustainedKey = key + ":sustained";
        long count = redis.opsForValue().increment(sustainedKey);
        if (count == 1) {
            redis.expire(sustainedKey, Duration.ofSeconds(3600)); // 1 hour
        }
        
        if (count > config.getSustainedLimit()) {
            return ThrottleResult.throttled("Sustained limit exceeded");
        }
        
        return ThrottleResult.allowed();
    }
    
    private ThrottleConfig getThrottleConfig(Tier tier) {
        return switch (tier) {
            case FREE -> new ThrottleConfig(100, 1000); // 100/min burst, 1000/hour sustained
            case BASIC -> new ThrottleConfig(500, 5000);
            case PRO -> new ThrottleConfig(1000, 10000);
            case ENTERPRISE -> new ThrottleConfig(10000, 100000);
        };
    }
}
```

#### Example 3: Spring Boot Throttling with Guava RateLimiter

**Maven dependency**:
```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>
```

**Throttling interceptor**:
```java
@Component
public class ThrottlingInterceptor implements HandlerInterceptor {
    
    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        String userId = extractUserId(request);
        
        // Get or create rate limiter for user
        RateLimiter rateLimiter = rateLimiters.computeIfAbsent(userId, 
            k -> RateLimiter.create(100.0 / 60.0)); // 100 requests per minute
        
        if (rateLimiter.tryAcquire()) {
            // Allowed, set headers
            response.setHeader("X-Throttle-Limit", "100");
            response.setHeader("X-Throttle-Remaining", 
                String.valueOf((int) rateLimiter.getAvailablePermits()));
            return true;
        } else {
            // Throttled
            response.setStatus(429);
            response.setHeader("X-Throttle-Limit", "100");
            response.setHeader("X-Throttle-Remaining", "0");
            response.setHeader("Retry-After", 
                String.valueOf((long) rateLimiter.getRate()));
            return false;
        }
    }
}
```

#### Example 4: Token Bucket Throttling

**Token bucket implementation**:
```java
@Service
public class TokenBucketThrottler {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    public ThrottleResult tryAcquire(String userId, long capacity, double refillRate) {
        String key = "throttle:token:" + userId;
        
        // Get current bucket state
        TokenBucket bucket = getBucket(key);
        long now = System.currentTimeMillis();
        
        // Refill tokens
        long elapsed = now - bucket.getLastRefill();
        double tokensToAdd = (elapsed / 1000.0) * refillRate;
        double newTokens = Math.min(capacity, bucket.getTokens() + tokensToAdd);
        
        if (newTokens >= 1.0) {
            // Token available, consume and allow
            newTokens -= 1.0;
            bucket.setTokens(newTokens);
            bucket.setLastRefill(now);
            saveBucket(key, bucket);
            
            return ThrottleResult.allowed();
        } else {
            // No tokens, throttle
            double waitTime = (1.0 - newTokens) / refillRate;
            return ThrottleResult.throttled(waitTime);
        }
    }
    
    private TokenBucket getBucket(String key) {
        String data = redis.opsForValue().get(key);
        if (data == null) {
            return new TokenBucket(100.0, System.currentTimeMillis());
        }
        return TokenBucket.deserialize(data);
    }
    
    private void saveBucket(String key, TokenBucket bucket) {
        redis.opsForValue().set(key, bucket.serialize(), 
            Duration.ofHours(24));
    }
}

@Data
class TokenBucket {
    private double tokens;
    private long lastRefill;
    
    TokenBucket(double tokens, long lastRefill) {
        this.tokens = tokens;
        this.lastRefill = lastRefill;
    }
    
    String serialize() {
        return tokens + ":" + lastRefill;
    }
    
    static TokenBucket deserialize(String data) {
        String[] parts = data.split(":");
        return new TokenBucket(Double.parseDouble(parts[0]), 
            Long.parseLong(parts[1]));
    }
}
```

#### Example 5: Graceful Degradation with Throttling

**Degradation strategy**:
```java
@Service
public class GracefulThrottlingService {
    
    @Autowired
    private SystemHealthMonitor healthMonitor;
    
    public ThrottleResult checkThrottle(String userId, ThrottleConfig baseConfig) {
        // Adjust limits based on system health
        SystemHealth health = healthMonitor.getHealth();
        
        ThrottleConfig adjustedConfig = adjustConfigForHealth(baseConfig, health);
        
        // Check throttle with adjusted limits
        return doCheckThrottle(userId, adjustedConfig);
    }
    
    private ThrottleConfig adjustConfigForHealth(ThrottleConfig base, SystemHealth health) {
        if (health.isCritical()) {
            // Reduce limits by 80% during critical load
            return base.withLimit((long) (base.getLimit() * 0.2));
        } else if (health.isHigh()) {
            // Reduce limits by 50% during high load
            return base.withLimit((long) (base.getLimit() * 0.5));
        } else if (health.isModerate()) {
            // Reduce limits by 20% during moderate load
            return base.withLimit((long) (base.getLimit() * 0.8));
        }
        
        return base; // Normal limits
    }
}
```

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Queuing vs Rejection
- **Queuing**: Better UX, but requires memory and can cause delays
- **Rejection**: Simpler, immediate feedback, but worse UX

#### 2. Strictness vs Flexibility
- **Strict throttling**: Better resource protection, but users hit limits more
- **Flexible throttling**: Better UX, but less protection

#### 3. Global vs Per-User
- **Global throttling**: Simpler, but unfair (one user can affect all)
- **Per-user throttling**: Fair, but more complex

### Common Pitfalls

#### Pitfall 1: Queue Memory Exhaustion
**Problem**: Unlimited queues consume all memory
```java
// Bad: Unlimited queue
Queue<Request> queue = new LinkedBlockingQueue<>(); // Can grow unbounded
```

**Solution**: Use bounded queues
```java
// Good: Bounded queue
Queue<Request> queue = new LinkedBlockingQueue<>(1000); // Max 1000 items
// Reject when full
```

#### Pitfall 2: Stale Queued Requests
**Problem**: Requests sit in queue too long, clients timeout
**Solution**: Expire old requests, reject with appropriate status

#### Pitfall 3: Not Handling Redis Failures
**Problem**: Throttling fails when Redis is down
**Solution**: Fail open with monitoring, or use local fallback

#### Pitfall 4: Inconsistent Throttling Across Servers
**Problem**: Different servers throttle differently
**Solution**: Use shared state (Redis) or consistent hashing

#### Pitfall 5: No Monitoring
**Problem**: Can't tune throttling without data
**Solution**: Track throttle events, queue depths, wait times

### Performance Gotchas

#### Gotcha 1: Throttling Overhead
**Problem**: Throttling check adds latency to every request
**Solution**: Cache throttle state, use async checks when possible

#### Gotcha 2: Queue Processing Overhead
**Problem**: Background queue processing consumes resources
**Solution**: Batch queue processing, limit processing frequency

## 8️⃣ When NOT to use this

### When Throttling Isn't Needed

1. **Low-traffic APIs**: APIs with predictable, low traffic don't need throttling

2. **Internal services**: Services behind firewall, trusted clients

3. **Read-only cached data**: Public data that's heavily cached

### Anti-Patterns

1. **Over-throttling**: Too strict limits hurt legitimate users

2. **Under-throttling**: Limits too high, doesn't protect resources

3. **No feedback**: Users don't know why requests are slow

## 9️⃣ Comparison with Alternatives

### Throttling vs Rate Limiting

**Throttling**: Queues requests, processes at controlled rate
- Pros: Better UX, graceful degradation
- Cons: More complex, requires queuing

**Rate Limiting**: Rejects requests when limit exceeded
- Pros: Simpler, immediate feedback
- Cons: Worse UX, all-or-nothing

**Best practice**: Use throttling for user-facing APIs, rate limiting for abuse prevention

### Throttling vs Load Balancing

**Throttling**: Controls request rate per user/service
- Pros: Fair resource allocation, prevents overload
- Cons: Adds latency, complexity

**Load Balancing**: Distributes load across servers
- Pros: Horizontal scaling, high availability
- Cons: Doesn't prevent overload, just distributes it

**Best practice**: Use both - load balancing for capacity, throttling for control

## 🔟 Interview follow-up questions WITH answers

### Question 1: What's the difference between throttling and rate limiting?

**Answer**:
**Rate limiting**: Hard limits, rejects requests when exceeded
- Binary: Allow or reject
- Immediate feedback (429 status)
- Simpler implementation
- Use for: Abuse prevention, quota enforcement

**Throttling**: Soft limits, queues requests when exceeded
- Gradual: Process at controlled rate
- Requests delayed but not rejected
- More complex (requires queuing)
- Use for: Resource protection, fair sharing, graceful degradation

**Example**:
- **Rate limiting**: "100 requests/hour" → Reject 101st request
- **Throttling**: "100 requests/hour" → Queue 101st request, process when capacity available

### Question 2: How do you implement tiered throttling?

**Answer**:
**Implementation**:

1. **User tier lookup**: Get user's tier (free, paid, enterprise)
2. **Tier-specific limits**: Different limits per tier
3. **Per-user tracking**: Track usage per user within tier limits
4. **Dynamic limits**: Limits can change based on tier upgrades

**Example**:
```java
public ThrottleResult checkThrottle(String userId) {
    User user = getUser(userId);
    Tier tier = user.getTier();
    
    ThrottleConfig config = getConfigForTier(tier);
    // free: 100/hour, paid: 10000/hour, enterprise: unlimited
    
    return throttler.checkThrottle(userId, config);
}
```

**Considerations**:
- Store tier information (database, cache)
- Update limits when tier changes
- Monitor usage per tier
- Alert on tier limit violations

### Question 3: How do you handle burst vs sustained throttling?

**Answer**:
**Dual limits**:

1. **Burst limit**: Short-term limit (e.g., 100 requests/minute)
   - Allows traffic spikes
   - Prevents sudden overload
   - Uses sliding window or token bucket

2. **Sustained limit**: Long-term limit (e.g., 10,000 requests/hour)
   - Prevents sustained abuse
   - Controls total usage
   - Uses fixed or sliding window

**Implementation**:
```java
public ThrottleResult checkThrottle(String userId) {
    // Check burst limit
    ThrottleResult burst = checkBurstLimit(userId, 100, 60); // 100/min
    if (!burst.isAllowed()) {
        return ThrottleResult.throttled("Burst limit exceeded");
    }
    
    // Check sustained limit
    ThrottleResult sustained = checkSustainedLimit(userId, 10000, 3600); // 10k/hour
    if (!sustained.isAllowed()) {
        return ThrottleResult.throttled("Sustained limit exceeded");
    }
    
    return ThrottleResult.allowed();
}
```

**Benefits**: Allows legitimate bursts while preventing sustained abuse

### Question 4: How do you implement graceful degradation with throttling?

**Answer**:
**Strategies**:

1. **Dynamic limit adjustment**: Reduce limits when system under load
```java
if (systemLoad.isHigh()) {
    limit = baseLimit * 0.5; // Reduce to 50%
}
```

2. **Priority queuing**: Process high-priority requests first
```java
if (request.isHighPriority()) {
    priorityQueue.offer(request);
} else {
    normalQueue.offer(request);
}
```

3. **Quality reduction**: Return simplified responses under load
```java
if (systemLoad.isHigh()) {
    return simplifiedResponse(); // Less data, faster
}
```

4. **User tier prioritization**: Premium users get better service
```java
if (user.isPremium() && systemLoad.isHigh()) {
    // Premium users get full service
    return fullResponse();
} else {
    // Free users get throttled/degraded
    return throttledResponse();
}
```

### Question 5: How would you throttle in a distributed system?

**Answer**:
**Challenges**:
- Multiple servers need consistent throttling
- Shared state required
- Network latency for throttle checks

**Solutions**:

1. **Redis-based throttling**: Shared state in Redis
```java
// All servers check same Redis key
String key = "throttle:" + userId;
long count = redis.incr(key);
if (count == 1) redis.expire(key, 3600);
if (count > limit) throttle();
```

2. **Local cache with Redis sync**: Cache throttle state locally, sync with Redis
```java
// Check local cache first
if (localCache.has(userId)) {
    return localCache.get(userId);
}
// Miss: Check Redis and update cache
ThrottleResult result = checkRedis(userId);
localCache.put(userId, result);
```

3. **Consistent hashing**: Route same user to same server, use local throttling
```java
int server = consistentHash(userId);
// User always goes to same server, can use local throttling
```

**Best practice**: Use Redis for accuracy, local cache for performance

### Question 6 (L5/L6): How would you design a throttling system that handles millions of requests per second?

**Answer**:
**Design considerations**:

1. **Sharding**: Shard throttle data by user ID
   - Consistent hashing for distribution
   - Reduces Redis key space per shard
   - Parallel processing

2. **Caching layer**: Multi-level caching
   - L1: Local cache (Caffeine) - hot users
   - L2: Redis cluster - distributed state
   - Cache hit rate target: > 95%

3. **Async processing**: Non-blocking throttle checks
   - Check throttle asynchronously
   - Process requests while checking
   - Pre-warm cache for known users

4. **Approximate algorithms**: Probabilistic data structures
   - Count-Min Sketch for approximate counts
   - Trade accuracy for performance
   - Good enough for throttling

5. **Hardware optimization**:
   - Redis on fast SSDs (NVMe)
   - Keep hot data in memory
   - Network optimization (low latency)

**Architecture**:
```
Request → Load Balancer → API Server
                            ↓
                    Local Cache (Caffeine)
                            ↓ (miss)
                    Redis Cluster (sharded)
                            ↓
                    Throttle Decision → Queue/Process
```

**Monitoring**:
- P99 latency < 1ms for throttle check
- Cache hit rate > 95%
- Redis CPU < 70%
- Queue depth monitoring
- Throttle event tracking

## 1️⃣1️⃣ One clean mental summary

API throttling controls request flow to protect system resources while maintaining service availability, unlike rate limiting which simply rejects excess requests. Key mechanisms include request queuing (requests wait instead of being rejected), token bucket throttling (tokens refill at constant rate, requests consume tokens), and leaky bucket throttling (requests leak at constant rate). Throttling strategies include user-level (per-user limits), API key-based (different limits per key), tiered (different limits per tier), and burst vs sustained (allow bursts but throttle sustained high rates). Critical considerations include queuing vs rejection tradeoffs, handling distributed systems (shared state in Redis), graceful degradation (adjust limits based on system load), and monitoring throttle effectiveness. Always use bounded queues to prevent memory exhaustion, handle Redis failures gracefully, and provide feedback to users about throttling status.

