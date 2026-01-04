# Key-Value Store - Interview Grilling

## STEP 14: TRADEOFFS & INTERVIEW QUESTIONS

### Why In-Memory over Disk-Based?

**In-Memory:**
- Pros: Fast (< 1ms), simple
- Cons: Limited by memory, data lost on crash
- **We chose in-memory** for performance

**Disk-Based:**
- Pros: Larger capacity, persistent
- Cons: Slower (milliseconds), more complex
- **Why not:** Need sub-millisecond latency

### Why Eventual Consistency?

**Answer:** Key-value store prioritizes availability. Eventual consistency is acceptable for most use cases (caching, sessions). Replication provides eventual consistency with low latency.

### What Breaks First Under Load?

**Answer:** Memory. When memory is full, eviction starts, causing cache misses and increased database load.

**Mitigation:** Scale horizontally, optimize memory usage, increase eviction threshold

### System Evolution

**MVP:**
- Single node
- Basic operations (GET/SET)
- No persistence

**Production:**
- Replication
- Persistence (RDB/AOF)
- Clustering
- Advanced data structures

**Scale:**
- Horizontal scaling
- Multi-region
- Advanced features (transactions, Lua)

### Level-Specific Answers

**L4:** Understand basic operations, data structures, eviction
**L5:** Understand replication, persistence, clustering, failure scenarios
**L6:** Understand distributed systems, consistency, performance optimization

## STEP 15: FAILURE SCENARIOS

**Master Node Failure:**
- Sentinel promotes replica
- Clients reconnect
- No data loss
- RTO: < 30 seconds

**Replica Failure:**
- Master continues serving
- Replica can be replaced
- No impact on availability

**Network Partition:**
- Split-brain possible
- Use quorum for writes
- Eventual consistency on merge

**Memory Full:**
- Eviction starts
- Cache misses increase
- Database load increases
- Mitigation: Scale horizontally

**Disk Full (Persistence):**
- AOF writes fail
- RDB snapshots fail
- Data still in memory
- Mitigation: Add disk space, cleanup old files

**Data Corruption:**
- RDB/AOF validation
- Restore from backup
- Replication provides redundancy

