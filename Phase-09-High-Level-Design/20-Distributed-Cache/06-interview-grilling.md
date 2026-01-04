# Distributed Cache - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a distributed cache design.

---

## Trade-off Questions

### Q1: Consistent Hashing vs Hash Modulo?

**Answer:**

**Hash Modulo:**
```java
int shard = hash(key) % num_nodes;
```

**Problems:**
- All keys remap when nodes added/removed
- Uneven distribution
- Rebalancing is expensive

**Consistent Hashing:**
```java
// Hash ring with virtual nodes
String node = consistentHash.getNode(key);
```

**Benefits:**
- Only 1/N keys remap when node added/removed
- Even distribution (with virtual nodes)
- Smooth rebalancing

**Trade-off Analysis:**

| Factor | Hash Modulo | Consistent Hashing |
|--------|-------------|-------------------|
| Rebalancing | All keys remap | Only 1/N keys remap |
| Distribution | Uneven | Even (with virtual nodes) |
| Complexity | Simple | Complex |
| Performance | Fast | Slightly slower |

**Our Choice:** Consistent hashing (better for dynamic clusters)

---

### Q2: How do you handle hot keys?

**Answer:**

**Problem:** Popular key causes one node to be overloaded

**Solutions:**

1. **Replication**
   ```java
   // Replicate hot key to multiple nodes
   String[] nodes = getNodesForKey(key, replicationFactor=3);
   for (String node : nodes) {
       cache.set(node, key, value);
   }
   ```

2. **Local Caching**
   ```java
   // Cache hot keys in application memory
   private final Map<String, Object> localCache = new LRUCache<>(1000);
   
   public Object get(String key) {
       Object local = localCache.get(key);
       if (local != null) {
           return local;
       }
       // Fetch from distributed cache
   }
   ```

3. **Sharding**
   ```java
   // Split hot key into multiple keys
   String key1 = key + ":shard1";
   String key2 = key + ":shard2";
   // Distribute across nodes
   ```

4. **Pre-warming**
   ```java
   // Pre-load hot keys before expiration
   @Scheduled(fixedRate = 300000)  // Every 5 minutes
   public void prewarmHotKeys() {
       for (String hotKey : getHotKeys()) {
           Object value = fetchFromDatabase(hotKey);
           cache.set(hotKey, value);
       }
   }
   ```

---

### Q3: Cache Coherency - How do you keep cache consistent?

**Answer:**

**Challenge:** Multiple nodes, same key, data changes

**Strategies:**

1. **Write-Through**
   ```java
   public void update(String key, Object value) {
       // Write to database
       database.update(key, value);
       // Invalidate cache
       cache.delete(key);
   }
   ```
   - Pros: Simple, consistent
   - Cons: Slower writes

2. **Write-Behind (Write-Back)**
   ```java
   public void update(String key, Object value) {
       // Write to cache immediately
       cache.set(key, value);
       // Async write to database
       asyncWriteToDatabase(key, value);
   }
   ```
   - Pros: Fast writes
   - Cons: Risk of data loss

3. **Invalidation**
   ```java
   public void update(String key, Object value) {
       database.update(key, value);
       // Invalidate cache (pub/sub to all nodes)
       cache.invalidate(key);
   }
   ```
   - Pros: Simple, fast
   - Cons: Cache miss on next read

4. **Versioning**
   ```java
   public void update(String key, Object value, int version) {
       // Check version before updating
       if (cache.getVersion(key) < version) {
           cache.set(key, value, version);
       }
   }
   ```
   - Pros: Handles concurrent updates
   - Cons: More complex

**Our Choice:** Invalidation (simpler, acceptable cache misses)

---

### Q4: LRU vs LFU vs Random eviction?

**Answer:**

**LRU (Least Recently Used):**
- Evicts least recently accessed
- Good for: Temporal locality (recent = likely to be accessed again)
- Works well for: Most use cases

**LFU (Least Frequently Used):**
- Evicts least frequently accessed
- Good for: Long-term popularity
- Works well for: Stable access patterns

**Random:**
- Evicts random key
- Good for: Uniform access patterns
- Works well for: Simple implementations

**Comparison:**

| Factor | LRU | LFU | Random |
|--------|-----|-----|--------|
| Implementation | Medium | Complex | Simple |
| Performance | Good | Good | Good |
| Accuracy | High | High | Low |
| Memory overhead | Medium | High | Low |

**Our Choice:** LRU (good balance of accuracy and simplicity)

---

## Scaling Questions

### Q5: How to scale from 10K to 100K QPS?

**Answer:**

**Current:**
- 10 nodes
- 10K QPS
- 1 TB storage

**Scaling to 100K QPS:**

1. **Add Nodes**
   ```
   Nodes: 10 → 100 (10x)
   QPS per node: 1K → 1K (same)
   ```

2. **Optimize Memory**
   ```
   Use data compression
   Aggressive TTL management
   Better eviction policies
   ```

3. **Connection Pooling**
   ```
   Optimize connection reuse
   Reduce connection overhead
   ```

4. **Local Caching**
   ```
   Cache hot keys in application memory
   Reduce distributed cache load
   ```

**Bottlenecks:**
- Network bandwidth: Add more network capacity
- Memory: Optimize data structures
- CPU: Use faster instances

---

## Failure Scenarios

### Scenario 1: Cache Node Failure

**Problem:** One cache node fails, keys unavailable.

**Solution:**

1. **Automatic Failover**
   - Replica promoted to primary
   - Keys reassigned to other nodes
   - Cache repopulates from database

2. **Impact**
   - Temporary cache misses
   - Slight latency increase
   - System continues operating

3. **Recovery**
   - RTO: < 1 minute
   - RPO: 0 (data in replicas)

---

### Scenario 2: Cache Cluster Failure

**Problem:** Entire cache cluster down.

**Solution:**

1. **Fallback to Database**
   ```java
   public Object get(String key) {
       try {
           return cache.get(key);
       } catch (CacheException e) {
           // Fallback to database
           return database.get(key);
       }
   }
   ```

2. **Impact**
   - Higher latency (database queries)
   - Increased database load
   - System continues operating

3. **Recovery**
   - Automatic cluster restart
   - Cache repopulates gradually
   - Normal operations resume

---

## Level-Specific Expectations

### L4 (Entry-Level)

**Expected:**
- Basic caching concept
- Simple key-value storage
- TTL understanding

**Sample Answer:**
> "A distributed cache stores data in memory across multiple servers for fast access. We use consistent hashing to distribute keys, and LRU eviction when cache is full."

---

### L5 (Mid-Level)

**Expected:**
- Consistent hashing with virtual nodes
- Cache invalidation strategies
- Eviction policies
- Failure handling

**Sample Answer:**
> "I'd use consistent hashing with 150 virtual nodes per physical node for even distribution. When a node fails, only 1/N keys need remapping. For cache coherency, I'd use write-through invalidation. LRU eviction works well for most access patterns."

---

### L6 (Senior)

**Expected:**
- System evolution
- Advanced techniques (cache stampede prevention)
- Cost optimization
- Real-world trade-offs

**Sample Answer:**
> "For MVP, a single Redis instance works. As we scale, consistent hashing with virtual nodes ensures even distribution. The hardest problem is cache stampede - when a popular key expires, thousands of requests hit the database. I solve this with distributed locking: only one request computes, others wait and get the cached result.

> For cost optimization, the biggest lever is hit rate. At 80% hit rate, we serve 4x more requests from cache than database, reducing database load significantly. We optimize by: (1) Longer TTLs for stable data, (2) Pre-warming hot keys, (3) Local caching for hottest keys.

> Trade-offs: Write-through is simpler but slower. Write-behind is faster but risks data loss. We choose write-through for consistency, accepting slightly slower writes."

---

## Common Interviewer Pushbacks

### "What if consistent hashing causes uneven distribution?"

**Response:**
"Virtual nodes solve this. With 150 virtual nodes per physical node, the distribution is very even. Even if one physical node has slightly more keys, the difference is minimal (< 5%). We monitor key distribution and can add more virtual nodes if needed."

### "How do you handle cache stampede at scale?"

**Response:**
"Distributed locking ensures only one request computes the value. Other requests wait briefly (50ms) and then check cache again. This prevents database overload. We also use TTL jitter (±10%) to prevent simultaneous expiration of related keys."

### "What if you had unlimited budget?"

**Response:**
"With unlimited budget: (1) Deploy cache nodes in every region for low latency, (2) Replicate all keys to all nodes (no misses), (3) Use fastest hardware (NVMe SSDs, high-speed network), (4) Advanced ML for eviction policies, (5) Real-time cache analytics. Estimated cost: $500K+/month vs current $12K."

---

## Summary

| Question Type | Key Points to Cover |
|---------------|---------------------|
| Trade-offs | Consistent hashing, eviction policies, coherency |
| Scaling | Add nodes, optimize memory, local caching |
| Failures | Automatic failover, database fallback |
| Hot Keys | Replication, local caching, sharding |
