# 💾 PHASE 3: DATA STORAGE & CONSISTENCY (Weeks 3-4)

**Goal**: Master database fundamentals and distributed data

**Learning Objectives**:
- Choose the right database for different use cases
- Understand indexing strategies and query optimization
- Design sharding and replication strategies
- Handle distributed transactions and consistency

**Estimated Time**: 30-40 hours

---

## Topics:

1. **SQL vs NoSQL**
   - ACID properties explained
   - BASE properties
   - When to use each (decision framework)
   - Polyglot persistence

2. **Database Indexing**
   - B-Tree internals (how it actually works)
   - B+ Tree (why databases prefer it)
   - Hash indexes
   - Bitmap indexes
   - Covering indexes
   - Composite indexes (column order matters)
   - Index selection strategy

3. **Normalization vs Denormalization**
   - Normal forms (1NF, 2NF, 3NF, BCNF)
   - When to denormalize
   - Trade-offs (storage vs query speed)
   - Materialized views

4. **Database Replication**
   - Leader-Follower (primary-replica)
   - Multi-Leader (conflict resolution)
   - Leaderless (Dynamo-style, quorum)
   - Sync vs Async replication
   - Replication lag handling

5. **Database Sharding**
   - Hash-based sharding
   - Range-based sharding
   - Directory-based sharding
   - Hot partitions (celebrity problem)
   - Resharding strategies
   - Cross-shard queries

6. **Consistency Models (Database-Specific)**
   - Strong consistency (linearizability)
   - Eventual consistency
   - Causal consistency
   - Read-your-writes consistency
   - Tunable consistency (Cassandra)
   - *Reference*: See Phase 1, Topic 12 for foundational concepts

7. **Transactions**
   - ACID guarantees deep dive
   - Isolation levels (Read Uncommitted → Serializable)
   - 2-Phase Commit (2PC)
   - 3-Phase Commit (3PC)
   - Distributed transactions (Saga pattern)
   - Transaction deadlocks

8. **Advanced DB Concepts**
   - Write amplification
   - LSM trees vs B-trees
   - Query optimization
   - Execution plans (EXPLAIN)
   - Query planner internals

9. **Read Replicas & Scaling Reads**
   - Master-slave replication for read scaling
   - Read-after-write consistency problem
   - Stale read handling
   - Connection pooling (HikariCP, PgBouncer)
   - Read replica lag monitoring

10. **Conflict Resolution**
    - Last-write-wins (LWW)
    - CRDTs (Conflict-free Replicated Data Types)
    - Vector clocks for conflict detection
    - Merge strategies
    - Application-level resolution

11. **Database Partitioning Strategies**
    - Vertical partitioning
    - Horizontal partitioning (sharding)
    - Partition key selection
    - Cross-partition queries
    - Partition pruning

12. **Time-Series Databases**
    - Use cases (metrics, IoT, financial data)
    - Compression techniques
    - Retention policies
    - Downsampling
    - Examples (InfluxDB, TimescaleDB, Prometheus)

13. **NewSQL Databases**
    - What is NewSQL?
    - Google Spanner (TrueTime, global consistency)
    - CockroachDB (distributed SQL)
    - TiDB (MySQL compatible)
    - When to choose NewSQL over traditional

14. **Change Data Capture (CDC)**
    - What is CDC?
    - Debezium for CDC
    - Kafka Connect
    - Use cases (cache invalidation, search indexing, analytics)
    - CDC patterns and pitfalls

15. **Database Connection Management**
    - Connection pooling deep dive
    - HikariCP configuration
    - PgBouncer for PostgreSQL
    - Connection limits and sizing
    - Connection leak detection

---

## Cross-References:
- **Consistency Models**: Foundational concepts in Phase 1, Topic 12
- **Caching**: See Phase 4 for cache-aside pattern with databases
- **CDC**: Used in Phase 6 (Messaging) for event-driven patterns

---

## Practice Problems:
1. Design the database schema for Instagram
2. Choose sharding strategy for a multi-tenant SaaS
3. Implement read-your-writes consistency with replicas
4. Design a migration from MySQL to Cassandra

---

## Common Interview Questions:
- "How would you handle a hot partition?"
- "When would you choose eventual consistency?"
- "How do you ensure consistency across shards?"
- "What's the difference between 2PC and Saga?"

---

## Deliverable
Can design database layer for Instagram
