# Distributed Rate Limiter - Interview Grilling

## Trade-off Questions

### Q1: Token Bucket vs Sliding Window?

**Answer:**

**Token Bucket:**
- Allows bursts (good for user experience)
- Simpler implementation
- Less accurate (can exceed limit during bursts)

**Sliding Window:**
- More accurate (no boundary bursts)
- More complex (requires sorted set)
- No bursts allowed

**When to Use:**
- Token bucket: User-facing APIs (better UX)
- Sliding window: Strict rate limits (API quotas)

**Our Choice:** Both (token bucket for user limits, sliding window for strict quotas)

---

### Q2: Why fail open instead of fail closed?

**Answer:**

**Fail Closed:**
- Blocks all requests if Redis down
- Prevents abuse but causes outages
- Cascading failures

**Fail Open:**
- Allows requests if Redis down
- System continues operating
- Slight over-counting acceptable

**Why Fail Open:**
- Better to allow some extra requests than block all
- Prevents cascading failures
- System availability > perfect rate limiting

---

### Q3: How do you handle distributed rate limiting?

**Answer:**

**Challenge:** Multiple servers, shared state

**Solution:**
1. **Redis Cluster**: Shared state across servers
2. **Atomic Operations**: Lua scripts for consistency
3. **Consistent Hashing**: Distribute keys evenly
4. **Replication**: High availability

**Trade-offs:**
- Slight latency (Redis network call)
- Eventual consistency during partitions
- Acceptable for rate limiting use case

---

## Scaling Questions

### Q4: How to scale from 10K to 100K QPS?

**Answer:**

**Current:**
- 3 Redis nodes
- 10K QPS

**Scaling:**
1. **Add Redis Nodes**: 3 → 10 nodes
2. **Sharding**: Distribute keys across nodes
3. **Connection Pooling**: Optimize connections
4. **Local Caching**: Cache hot keys locally

**Bottlenecks:**
- Redis network latency
- Key distribution (hot keys)

---

## Level-Specific Expectations

### L4 (Entry-Level)

**Expected:**
- Basic rate limiting concept
- Simple counter implementation
- Fixed window algorithm

---

### L5 (Mid-Level)

**Expected:**
- Token bucket or sliding window
- Redis implementation
- Distributed considerations

---

### L6 (Senior)

**Expected:**
- Multiple algorithms
- Trade-off analysis
- Failure handling
- Cost optimization

---

## Failure Scenarios

### Scenario 1: Redis Cluster Failure

**Problem:** Redis cluster down, can't check rate limits.

**Solution:**

1. **Fail Open Strategy**
   ```java
   public boolean isAllowed(String key) {
       try {
           return redisRateLimiter.check(key);
       } catch (RedisException e) {
           // Fail open: allow request
           log.warn("Redis down, allowing request");
           return true;
       }
   }
   ```

2. **Impact**
   - Rate limiting disabled temporarily
   - Some extra requests allowed
   - System continues operating

3. **Recovery**
   - Automatic failover to replica
   - Rate limiting resumes
   - RTO: < 3 minutes

---

### Scenario 2: Hot Key Problem

**Problem:** Popular user makes many requests, all hit same Redis node.

**Solution:**

1. **Local Caching**
   ```java
   // Cache rate limit result locally (1 second)
   private final Cache<String, Boolean> localCache = 
       Caffeine.newBuilder().expireAfterWrite(1, TimeUnit.SECONDS).build();
   
   public boolean isAllowed(String key) {
       Boolean cached = localCache.getIfPresent(key);
       if (cached != null) {
           return cached;
       }
       
       boolean allowed = redisRateLimiter.check(key);
       localCache.put(key, allowed);
       return allowed;
   }
   ```

2. **Key Sharding**
   ```java
   // Shard hot key across multiple Redis keys
   String shardKey = key + ":" + (requestId % 10);
   return redisRateLimiter.check(shardKey);
   ```

---

## Common Interviewer Pushbacks

### "Why not use a database for rate limiting?"

**Response:**
"Databases are too slow for rate limiting. At 100K QPS, database queries would be a bottleneck (5-10ms each). Redis provides sub-millisecond lookups, which is essential for rate limiting. The trade-off is eventual consistency, which is acceptable for rate limiting (slight over-counting is better than blocking all requests)."

### "What if Redis is slow?"

**Response:**
"We monitor Redis latency and have circuit breakers. If Redis latency exceeds 20ms p95, we: (1) Scale Redis cluster, (2) Optimize Lua scripts, (3) Add read replicas, (4) Use local caching for hot keys. We also fail open if Redis is completely down to prevent cascading failures."

### "How do you prevent rate limit bypass?"

**Response:**
"Multiple strategies: (1) Check multiple identifiers (user_id, IP, API key), (2) Device fingerprinting, (3) Behavioral analysis (detect unusual patterns), (4) Rate limit aggregation (per-user, per-IP, per-endpoint). An attacker would need to bypass all layers, which is difficult."

---

## Summary

| Question Type | Key Points to Cover |
|---------------|---------------------|
| Algorithms | Token bucket vs sliding window, trade-offs |
| Consistency | Eventual (fail open preferred) |
| Scaling | Redis cluster, sharding, local caching |
| Failures | Fail open, automatic failover |
| Hot Keys | Local caching, key sharding |

