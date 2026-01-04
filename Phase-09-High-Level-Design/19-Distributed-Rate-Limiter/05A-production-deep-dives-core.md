# Distributed Rate Limiter - Production Deep Dives (Core)

## Overview

This document covers the core production components: token bucket algorithm, sliding window implementation, and Redis caching strategy.

---

## 1. Token Bucket Algorithm

### A) CONCEPT: What is Token Bucket?

Token bucket is a rate limiting algorithm that allows bursts of traffic while maintaining an average rate. It works like a bucket that:
- Has a maximum capacity (burst size)
- Refills at a constant rate (sustained rate)
- Tokens are consumed per request

**Example:**
```
Capacity: 10 tokens
Refill rate: 2 tokens/second

Initial: 10 tokens
Request 1: 10 → 9 tokens
Request 2: 9 → 8 tokens
...
After 1 second: 8 → 10 tokens (refilled 2)
```

### B) OUR USAGE: Implementation

**Redis-Based Token Bucket:**

```java
@Service
public class TokenBucketRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean isAllowed(String key, int capacity, int refillRate) {
        String bucketKey = "bucket:" + key;
        String lastRefillKey = bucketKey + ":last_refill";
        
        long now = System.currentTimeMillis();
        
        // Lua script for atomic operations
        String luaScript = 
            "local tokens = redis.call('get', KEYS[1]) or ARGV[1]\n" +
            "local lastRefill = redis.call('get', KEYS[2]) or ARGV[2]\n" +
            "local now = tonumber(ARGV[3])\n" +
            "local capacity = tonumber(ARGV[4])\n" +
            "local refillRate = tonumber(ARGV[5])\n" +
            "local elapsed = (now - tonumber(lastRefill)) / 1000\n" +
            "local tokensToAdd = math.floor(elapsed * refillRate)\n" +
            "tokens = math.min(capacity, tonumber(tokens) + tokensToAdd)\n" +
            "if tokens > 0 then\n" +
            "  tokens = tokens - 1\n" +
            "  redis.call('set', KEYS[1], tokens)\n" +
            "  redis.call('set', KEYS[2], now)\n" +
            "  redis.call('expire', KEYS[1], 3600)\n" +
            "  redis.call('expire', KEYS[2], 3600)\n" +
            "  return 1\n" +
            "else\n" +
            "  return 0\n" +
            "end";
        
        Long result = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Arrays.asList(bucketKey, lastRefillKey),
            String.valueOf(capacity),
            String.valueOf(now - 1000),
            String.valueOf(now),
            String.valueOf(capacity),
            String.valueOf(refillRate)
        );
        
        return result != null && result == 1;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Request Allowed**

```
Step 1: Request Arrives
┌─────────────────────────────────────────────────────────────┐
│ Request: User user_123, Endpoint /api/posts                 │
│ Rate limit: 100 requests/minute                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Token Bucket
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET bucket:user_123:/api/posts                       │
│ Result: 5 tokens (current)                                  │
│ Redis: GET bucket:user_123:/api/posts:last_refill           │
│ Result: 1705312800000 (1 second ago)                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Calculate Refill
┌─────────────────────────────────────────────────────────────┐
│ Elapsed: 1000ms (1 second)                                  │
│ Refill rate: 100 tokens/minute = 1.67 tokens/second         │
│ Tokens to add: 1.67 tokens                                  │
│ Current: 5 + 1.67 = 6.67 tokens (capped at capacity 100)   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Consume Token
┌─────────────────────────────────────────────────────────────┐
│ Tokens: 6.67 → 5.67 tokens                                  │
│ Redis: SET bucket:user_123:/api/posts 5                     │
│ Redis: SET bucket:user_123:/api/posts:last_refill <now>     │
│ Decision: ALLOW (tokens > 0)                                │
│ Response: 200 OK                                             │
│ Latency: ~2ms                                                │
└─────────────────────────────────────────────────────────────┘
```

**Rate Limited Flow:**

```
Step 1: Request Arrives
┌─────────────────────────────────────────────────────────────┐
│ Request: User user_123, Endpoint /api/posts                 │
│ Current tokens: 0                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Token Bucket
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET bucket:user_123:/api/posts                       │
│ Result: 0 tokens                                            │
│ Last refill: 500ms ago                                      │
│ Tokens to add: 0.83 tokens (not enough for 1 request)      │
│ Current: 0.83 tokens                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Reject Request
┌─────────────────────────────────────────────────────────────┐
│ Decision: DENY (tokens < 1)                                 │
│ Response: 429 Too Many Requests                             │
│ Headers:                                                     │
│   X-RateLimit-Limit: 100                                    │
│   X-RateLimit-Remaining: 0                                  │
│   X-RateLimit-Reset: 1705312900 (30 seconds from now)       │
│ Latency: ~2ms                                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Sliding Window Algorithm

### A) CONCEPT: What is Sliding Window?

Sliding window tracks requests in a time window that "slides" forward. It's more accurate than fixed window because it doesn't allow bursts at window boundaries.

**Example:**
```
Window: 1 minute
Limit: 100 requests

Fixed window (problem):
  [0:00-1:00]: 100 requests (at 0:59)
  [1:00-2:00]: 100 requests (at 1:00)
  Total: 200 requests in 1 second (burst!)

Sliding window (solution):
  Always looks at last 60 seconds
  No boundary bursts
```

### B) OUR USAGE: Redis Sorted Set Implementation

```java
@Service
public class SlidingWindowRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String windowKey = "window:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000);
        
        // Lua script for atomic operations
        String luaScript = 
            "redis.call('zremrangebyscore', KEYS[1], 0, ARGV[1])\n" +
            "local count = redis.call('zcard', KEYS[1])\n" +
            "if count < tonumber(ARGV[2]) then\n" +
            "  redis.call('zadd', KEYS[1], ARGV[3], ARGV[4])\n" +
            "  redis.call('expire', KEYS[1], ARGV[5])\n" +
            "  return 1\n" +
            "else\n" +
            "  return 0\n" +
            "end";
        
        String requestId = UUID.randomUUID().toString();
        
        Long result = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(windowKey),
            String.valueOf(windowStart),
            String.valueOf(limit),
            String.valueOf(now),
            requestId,
            String.valueOf(windowSeconds + 60)  // TTL buffer
        );
        
        return result != null && result == 1;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Request Allowed**

```
Step 1: Request Arrives
┌─────────────────────────────────────────────────────────────┐
│ Request: User user_123, Endpoint /api/posts                 │
│ Limit: 100 requests/minute                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Clean Old Requests
┌─────────────────────────────────────────────────────────────┐
│ Redis: ZREMRANGEBYSCORE window:user_123:/api/posts          │
│         0 (now - 60000)                                     │
│ Removes: Requests older than 1 minute                       │
│ Result: 45 requests removed                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Count Current Requests
┌─────────────────────────────────────────────────────────────┐
│ Redis: ZCARD window:user_123:/api/posts                     │
│ Result: 55 requests in last minute                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Add Request
┌─────────────────────────────────────────────────────────────┐
│ Count: 55 < 100 (limit)                                      │
│ Redis: ZADD window:user_123:/api/posts <now> <request_id>  │
│ Decision: ALLOW                                              │
│ Response: 200 OK                                             │
│ Latency: ~3ms                                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Caching Strategy

### Redis Configuration

**Connection Pooling:**
- Max connections: 100 per server
- Connection timeout: 5 seconds
- Read timeout: 2 seconds

**Memory Management:**
- Max memory: 32 GB per node
- Eviction policy: allkeys-lru
- Persistence: AOF (append-only file)

**Replication:**
- Primary: 1 per shard
- Replicas: 1-2 per shard
- Automatic failover enabled

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Token Bucket | Redis + Lua | Atomic operations |
| Sliding Window | Redis ZSET | Sorted set with TTL |
| Latency | < 10ms p95 | Redis local/regional |
| Accuracy | 99.9% | Atomic operations prevent race conditions |

