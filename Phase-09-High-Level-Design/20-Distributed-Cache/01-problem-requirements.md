# Distributed Cache - Problem & Requirements

## What is a Distributed Cache?

A Distributed Cache provides fast, in-memory data storage across multiple servers, enabling high-performance data access with features like consistent hashing, replication, cache invalidation, and eviction policies.

**Example:**
```
User requests: GET /api/user/123
  ↓
Check distributed cache
  ↓
Cache HIT: Return cached data (< 1ms)
  ↓
Cache MISS: Query database, cache result, return data
```

### Why Does This Exist?

1. **Performance**: Sub-millisecond data access
2. **Scalability**: Distribute load across multiple nodes
3. **Availability**: High availability through replication
4. **Cost Reduction**: Reduce database load

### What Breaks Without It?

- Slow response times (database queries)
- Database overload
- Poor user experience
- Higher infrastructure costs

---

## Clarifying Questions

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (QPS)? | Determines cluster size | 100K QPS |
| What's the data size? | Affects memory requirements | 1TB total |
| What's the access pattern? | Affects eviction policy | 80/20 (hot/cold) |
| Do we need persistence? | Affects design | Optional (AOF) |

---

## Functional Requirements

### Core Features

1. **Consistent Hashing**
   - Distribute keys across nodes
   - Handle node failures
   - Rebalance on node addition/removal

2. **Replication**
   - Replicate data for availability
   - Automatic failover
   - Read from replicas

3. **Cache Invalidation**
   - TTL-based expiration
   - Manual invalidation
   - Pattern-based invalidation

4. **Eviction Policies**
   - LRU (Least Recently Used)
   - LFU (Least Frequently Used)
   - Random eviction

5. **Cache Stampede Prevention**
   - Distributed locking
   - Single-flight requests
   - Negative caching

---

## Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Latency | < 1ms p95 |
| Availability | 99.99% |
| Throughput | 100K QPS |
| Hit Rate | > 80% |

---

## Summary

| Aspect | Decision |
|--------|----------|
| Scale | 100K QPS |
| Latency | < 1ms p95 |
| Algorithm | Consistent hashing |
| Replication | 2-3 replicas per shard |

