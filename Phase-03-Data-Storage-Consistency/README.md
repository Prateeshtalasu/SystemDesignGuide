# 💾 PHASE 3: DATA STORAGE & CONSISTENCY (Weeks 3-4)

**Goal**: Master database fundamentals and distributed data

## Topics:

1. **SQL vs NoSQL**
   - ACID properties explained
   - BASE properties
   - When to use each (not just "depends")

2. **Database Indexing**
   - B-Tree internals (how it actually works)
   - Hash indexes
   - Bitmap indexes
   - Covering indexes
   - Index selection strategy

3. **Normalization vs Denormalization**
   - Normal forms (1NF, 2NF, 3NF, BCNF)
   - When to denormalize
   - Trade-offs

4. **Database Replication**
   - Leader-Follower (primary-replica)
   - Multi-Leader
   - Leaderless (Dynamo-style)
   - Sync vs Async replication

5. **Database Sharding**
   - Hash-based sharding
   - Range-based sharding
   - Directory-based sharding
   - Hot partitions (celebrity problem)
   - Resharding strategies

6. **Consistency Models**
   - Strong consistency
   - Eventual consistency
   - Causal consistency
   - Read-your-writes consistency

7. **Transactions**
   - ACID guarantees
   - Isolation levels (Read Uncommitted → Serializable)
   - 2-Phase Commit (2PC)
   - Distributed transactions

8. **Advanced DB Concepts**
   - Write amplification
   - LSM trees vs B-trees
   - Query optimization
   - Execution plans

9. **Read Replicas & Scaling Reads**
   - Master-slave replication for read scaling
   - Read-after-write consistency problem
   - Stale read handling
   - Connection pooling basics

10. **Conflict Resolution**
    - Last-write-wins (LWW)
    - CRDTs (Conflict-free Replicated Data Types)
    - Vector clocks for conflict detection
    - Merge strategies

11. **Database Partitioning Strategies**
    - Vertical partitioning
    - Horizontal partitioning (sharding)
    - Partition key selection
    - Cross-partition queries

12. **Time-Series Databases**
    - Use cases (metrics, IoT)
    - Compression techniques
    - Retention policies
    - Examples (InfluxDB, TimescaleDB)

## Deliverable
Can design database layer for Instagram

