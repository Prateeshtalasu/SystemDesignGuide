# ⚡ PHASE 4: CACHING (Week 4)

**Goal**: Understand caching strategies and Redis mastery

## Topics:

1. **Caching Patterns**
   - Cache-Aside (lazy loading)
   - Write-Through
   - Write-Behind (write-back)
   - Refresh-Ahead

2. **Cache Invalidation**
   - TTL strategies
   - Event-based invalidation
   - Version-based invalidation
   - "Two hardest things" problem

3. **Redis Deep Dive**
   - Data structures (String, Hash, List, Set, Sorted Set)
   - Use cases for each
   - Persistence (RDB, AOF)
   - Redis Cluster
   - Redis Sentinel

4. **Advanced Caching**
   - Cache stampede (thundering herd)
   - Distributed cache coordination
   - Multi-level caching (L1 local + L2 Redis)
   - Cache warming

5. **Local vs Distributed Cache**
   - When to use each
   - Caffeine (local)
   - Redis/Memcached (distributed)

6. **Cache Coherency**
   - Cache invalidation strategies
   - Write-through vs write-behind trade-offs
   - Cache versioning

7. **Cache Optimization**
   - Cache hit ratio optimization
   - Cache partitioning strategies
   - Hot key problem
   - Cache warming techniques

8. **Memcached vs Redis**
   - When to use each
   - Performance characteristics
   - Feature comparison

## Deliverable
Can explain Facebook's cache architecture

