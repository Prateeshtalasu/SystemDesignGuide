# Key-Value Store (Redis-like) - Problem & Requirements

## What is a Key-Value Store?

A Key-Value Store is a NoSQL database that stores data as key-value pairs. It provides fast read/write operations by using simple data structures (hash tables) and in-memory storage. Think of it as a super-fast dictionary where you can store and retrieve values by their keys.

**Example:**
```
SET user:123 "John Doe"
GET user:123 → "John Doe"
```

### Why Does This Exist?

1. **Performance**: Sub-millisecond latency for reads/writes
2. **Simplicity**: Simple key-value model, easy to use
3. **Caching**: Perfect for caching frequently accessed data
4. **Session Storage**: Store user sessions, shopping carts
5. **Real-Time Data**: Store real-time counters, leaderboards
6. **Pub/Sub**: Publish-subscribe messaging
7. **Data Structures**: Support for strings, lists, sets, sorted sets, hashes

### What Breaks Without It?

- Slow database queries for simple lookups
- No fast caching layer
- Difficult to store session data
- No real-time data structures
- No pub/sub messaging
- High database load for simple operations

---

## Clarifying Questions

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (keys, QPS)? | Determines infrastructure | 100M keys, 1M QPS |
| What data structures needed? | Affects design | Strings, hashes, sorted sets, lists |
| Do we need persistence? | Affects storage design | Yes, RDB + AOF |
| What's the memory limit? | Affects eviction | 100 GB per node |
| Do we need replication? | Affects architecture | Yes, 3x replication |
| Do we need clustering? | Affects sharding | Yes, horizontal scaling |
| What's the latency target? | Affects design | < 1ms p99 |

---

## Functional Requirements

### Core Features (Must Have)

1. **Basic Operations**
   - SET key value
   - GET key
   - DELETE key
   - EXISTS key
   - TTL/EXPIRE key

2. **Data Structures**
   - Strings: Basic key-value
   - Hashes: Field-value pairs
   - Lists: Ordered collections
   - Sets: Unordered unique collections
   - Sorted Sets: Ordered unique collections with scores

3. **Persistence**
   - RDB snapshots (point-in-time backups)
   - AOF (Append-Only File) for durability
   - Configurable persistence modes

4. **Replication**
   - Master-replica replication
   - Automatic failover
   - Async replication

5. **Clustering**
   - Horizontal scaling
   - Data sharding
   - Automatic rebalancing

6. **Memory Management**
   - Eviction policies (LRU, LFU, random)
   - Max memory limits
   - Memory optimization

### Secondary Features

7. **Pub/Sub**
   - Publish messages to channels
   - Subscribe to channels
   - Pattern matching

8. **Transactions**
   - MULTI/EXEC for atomic operations
   - WATCH for optimistic locking

9. **Lua Scripting**
   - Execute Lua scripts server-side
   - Atomic operations

10. **Pipelining**
    - Batch operations
    - Reduce network round-trips

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Read Latency** | < 1ms p99 | Fast lookups |
| **Write Latency** | < 1ms p99 | Fast writes |
| **Throughput** | 1M ops/second | High performance |
| **Availability** | 99.99% | Critical infrastructure |
| **Durability** | 99.999999999% | Data must not be lost |
| **Memory Efficiency** | < 10% overhead | Efficient storage |
| **Scalability** | 10x growth | Horizontal scaling |

---

## User Stories

### Application Developer

1. **As a developer**, I want to cache database queries so I can improve application performance.

2. **As a developer**, I want to store session data so users stay logged in.

3. **As a developer**, I want to use sorted sets for leaderboards so I can rank users.

### System Administrator

1. **As an admin**, I want to persist data so it survives restarts.

2. **As an admin**, I want replication so the system is highly available.

3. **As an admin**, I want clustering so I can scale horizontally.

---

## Constraints & Assumptions

### Constraints

1. **Memory**: 100 GB per node
2. **Scale**: 100M keys, 1M QPS
3. **Latency**: < 1ms p99
4. **Durability**: Must persist to disk

### Assumptions

1. Average key size: 50 bytes
2. Average value size: 1 KB
3. 80% of operations are reads
4. Hot keys: 20% of keys account for 80% of traffic
5. Data structures: 60% strings, 20% hashes, 10% sorted sets, 10% others

---

## Out of Scope

1. **SQL Queries**: No SQL support
2. **Complex Joins**: No relational queries
3. **Full-Text Search**: Not a search engine
4. **Graph Queries**: Not a graph database
5. **ACID Transactions**: Limited transaction support

---

## Key Challenges

1. **Memory Management**: Efficient memory usage, eviction
2. **Persistence**: Fast writes with durability
3. **Replication**: Low-latency replication
4. **Clustering**: Data sharding and rebalancing
5. **Consistency**: Balancing consistency and performance
6. **Network**: Minimizing network overhead

---

## Success Metrics

- **Latency**: < 1ms p99
- **Throughput**: 1M ops/second
- **Availability**: 99.99%
- **Memory Efficiency**: < 10% overhead
- **Cache Hit Rate**: > 90% (for caching use cases)

---

## Next Steps

After requirements, we'll design:
1. Capacity estimation
2. API design
3. Data structures implementation
4. Persistence mechanisms
5. Replication and clustering

