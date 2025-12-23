# ⚡ PHASE 4: CACHING (Week 4)

**Goal**: Understand caching strategies and Redis mastery

**Learning Objectives**:
- Choose the right caching pattern for different scenarios
- Master Redis data structures and use cases
- Handle cache invalidation correctly
- Design multi-level caching architectures

**Estimated Time**: 12-15 hours

---

## Topics:

1. **Caching Patterns**
   - Cache-Aside (lazy loading)
   - Write-Through
   - Write-Behind (write-back)
   - Refresh-Ahead
   - Read-Through
   - When to use each pattern

2. **Cache Invalidation**
   - TTL strategies (fixed, sliding, adaptive)
   - Event-based invalidation
   - Version-based invalidation
   - "Two hardest things" problem
   - Cache invalidation patterns

3. **Redis Deep Dive**
   - Data structures (String, Hash, List, Set, Sorted Set)
   - Use cases for each structure
   - Persistence (RDB, AOF, hybrid)
   - Redis Cluster (sharding)
   - Redis Sentinel (high availability)
   - Redis Streams
   - Lua scripting

4. **Cache Eviction Policies**
   - LRU (Least Recently Used)
   - LFU (Least Frequently Used)
   - FIFO (First In First Out)
   - Random eviction
   - TTL-based eviction
   - ARC (Adaptive Replacement Cache)
   - Redis eviction policies

5. **Advanced Caching**
   - Cache stampede (thundering herd)
   - Distributed cache coordination
   - Multi-level caching (L1 local + L2 Redis)
   - Cache warming strategies
   - Cache preloading

6. **Local vs Distributed Cache**
   - When to use each
   - Caffeine (local) - configuration and tuning
   - Redis/Memcached (distributed)
   - Hybrid approaches
   - Cache synchronization

7. **Cache Coherency**
   - Cache invalidation strategies
   - Write-through vs write-behind trade-offs
   - Cache versioning
   - Eventual consistency in caching

8. **Cache Optimization**
   - Cache hit ratio optimization
   - Cache partitioning strategies
   - Hot key problem (solutions)
   - Cache warming techniques
   - Cache sizing strategies

9. **Memcached vs Redis**
   - When to use each
   - Performance characteristics
   - Feature comparison
   - Memory management differences
   - Clustering differences

10. **HTTP Caching**
    - Cache-Control headers
    - ETags and conditional requests
    - Vary header
    - Browser caching
    - Proxy caching

11. **CDN Caching**
    - Edge caching strategies
    - Cache invalidation at CDN
    - Stale-while-revalidate
    - Cache key design
    - *Reference*: See Phase 2 for CDN basics

12. **Caching Anti-Patterns**
    - Caching mutable data without invalidation
    - Over-caching (everything in cache)
    - Under-caching (no cache strategy)
    - Cache key collisions
    - Ignoring cache thundering herd

---

## Cross-References:
- **CDN**: See Phase 2 for CDN architecture
- **Redis Cluster**: See Phase 5.5 for consistent hashing
- **Cache Invalidation with CDC**: See Phase 3 and Phase 6

---

## Practice Problems:
1. Design caching strategy for a news feed
2. Implement cache-aside pattern with cache stampede prevention
3. Design multi-level cache for an e-commerce product catalog
4. Handle cache invalidation for a social media "like" count

---

## Common Interview Questions:
- "How do you handle cache invalidation?"
- "What happens if the cache goes down?"
- "How do you prevent cache stampede?"
- "When would you use local cache vs distributed cache?"

---

## Deliverable
Can explain Facebook's cache architecture
