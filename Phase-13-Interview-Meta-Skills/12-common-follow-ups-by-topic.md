# Common Follow-ups by Topic

## 0️⃣ Prerequisites

Before diving into common follow-ups, you should understand:

- **System Design Fundamentals**: Core components like databases, caches, and queues (covered in Phases 1-9)
- **Trade-off Analysis**: How to evaluate and justify decisions (covered in Topic 9)
- **Common Interview Scenarios**: Standard scenarios interviewers use (covered in Topic 11)

Quick refresher: After you present your initial design, interviewers ask follow-up questions to probe deeper. These questions are predictable by topic area. Knowing the common follow-ups and preparing thoughtful answers gives you a significant advantage.

---

## 1️⃣ Why Follow-ups Matter

Interviewers use follow-ups to:

1. **Test depth**: Do you understand the component you chose, or just the name?
2. **Probe trade-offs**: Do you understand the downsides of your choices?
3. **Explore edge cases**: Have you thought about failure modes?
4. **Assess adaptability**: Can you modify your design when requirements change?
5. **Differentiate candidates**: Surface-level answers vs deep understanding

The initial design gets you to "maybe hire." The follow-ups determine "hire" vs "no hire."

---

## 2️⃣ Database Follow-ups

### Follow-up 1: "How do you handle hot partitions?"

**Context**: When using sharding, some shards get more traffic than others.

**Answer**:
"Hot partitions occur when the shard key doesn't distribute evenly. For example, if we shard by user_id and one user has 100x more activity than others.

**Detection**:
- Monitor per-shard metrics (QPS, latency)
- Alert when one shard significantly exceeds others

**Prevention**:
- Choose a shard key with even distribution
- Use hash-based sharding instead of range-based
- Add salt to keys for known hot entities (celebrity user_id + random suffix)

**Mitigation**:
- Split hot shards into sub-shards
- Add caching in front of hot shards
- Rate limit hot entities

**Example**: For a Twitter-like system, celebrity accounts would be hot. We'd either:
1. Cache their data aggressively
2. Use a separate 'celebrity' shard with more capacity
3. Fan-out on read instead of write for celebrities"

### Follow-up 2: "How do you handle schema migrations?"

**Answer**:
"Schema migrations in production require careful planning to avoid downtime.

**Strategy 1: Backward-compatible changes**
- Add new columns as nullable
- Deploy code that handles both old and new schema
- Backfill data
- Remove old column handling from code
- Drop old column

**Strategy 2: Expand-contract pattern**
```
Phase 1 (Expand): Add new column, write to both old and new
Phase 2 (Migrate): Backfill new column from old
Phase 3 (Contract): Read from new, stop writing to old
Phase 4 (Cleanup): Drop old column
```

**Strategy 3: Shadow tables**
- Create new table with new schema
- Dual-write to both tables
- Migrate reads to new table
- Drop old table

**Tools**: Flyway, Liquibase for version control. pt-online-schema-change for MySQL. pg_repack for PostgreSQL."

### Follow-up 3: "SQL vs NoSQL - how did you decide?"

**Answer**:
"The decision depends on several factors:

**Choose SQL when**:
- Complex queries with joins
- ACID transactions required
- Data is relational
- Schema is stable
- Team has SQL expertise

**Choose NoSQL when**:
- Simple key-value or document access
- Horizontal scaling is primary concern
- Schema is flexible/evolving
- High write throughput needed
- Data is naturally denormalized

**For this design**, I chose [SQL/NoSQL] because:
- Our access pattern is [describe]
- We need [transactions/scalability/flexibility]
- The trade-off is [what we give up]

If requirements change to [opposite need], I'd reconsider."

### Follow-up 4: "How do you handle database failover?"

**Answer**:
"Database failover depends on the setup:

**Single primary with replicas**:
1. Health checks detect primary failure
2. Promote replica to primary (1-2 minutes)
3. Update connection strings or use DNS failover
4. Old primary becomes replica when recovered

**Multi-primary (active-active)**:
- Both can accept writes
- Need conflict resolution
- More complex but no failover needed

**Managed services (RDS Multi-AZ)**:
- Automatic failover in 1-2 minutes
- Standby in different AZ
- DNS automatically updated

**Application considerations**:
- Connection pooling handles reconnection
- Retry logic for transient failures
- Circuit breaker prevents cascade
- Queue writes during failover if acceptable"

---

## 3️⃣ Cache Follow-ups

### Follow-up 1: "What if the cache goes down?"

**Answer**:
"Cache failure has two concerns: availability and thundering herd.

**Immediate impact**:
- All requests go to database
- Database might be overwhelmed

**Mitigation strategies**:

1. **Cache redundancy**: Redis Cluster or Sentinel for automatic failover

2. **Graceful degradation**: 
   - Return stale data if available
   - Reduce functionality (disable recommendations)
   - Show 'temporarily unavailable' for non-critical features

3. **Thundering herd prevention**:
   - Request coalescing (only one request fetches, others wait)
   - Probabilistic early expiration
   - Gradual cache warming

4. **Circuit breaker**: If cache is slow, bypass it rather than wait

**Recovery**:
- Cache warms up gradually from DB queries
- Pre-warm critical data on startup
- Monitor cache hit rate to track recovery"

### Follow-up 2: "How do you handle cache invalidation?"

**Answer**:
"Cache invalidation is famously hard. Strategies depend on consistency requirements:

**Time-based (TTL)**:
- Set expiration time
- Simple but data can be stale
- Good for: analytics, recommendations

**Event-based**:
- Invalidate when data changes
- More complex but more consistent
- Good for: user profiles, inventory

**Write-through**:
- Update cache on every write
- Always consistent but slower writes
- Good for: session data

**Patterns**:

```
CACHE-ASIDE (most common):
Read:  Check cache → if miss, read DB → update cache
Write: Update DB → invalidate cache

WRITE-THROUGH:
Write: Update cache → cache updates DB

WRITE-BEHIND:
Write: Update cache → async update DB (risky)
```

**Common pitfalls**:
- Race conditions (read old data, write to cache after invalidation)
- Solution: Use versioning or cache-aside with short TTL"

### Follow-up 3: "How do you size your cache?"

**Answer**:
"Cache sizing involves balancing hit rate against cost.

**Estimation approach**:
1. Identify hot data (often 20% of data gets 80% of traffic)
2. Calculate size of hot data
3. Add headroom for growth

**Example calculation**:
```
Total users: 10 million
Hot users (daily active): 1 million (10%)
Data per user: 1 KB
Cache size: 1M × 1 KB = 1 GB

With 3x headroom: 3 GB
```

**Monitoring approach**:
- Start with estimate
- Monitor hit rate and eviction rate
- If hit rate < 95%, consider increasing size
- If eviction rate is high, increase size

**Cost consideration**:
- Redis: ~$0.10/GB/hour on AWS
- Balance hit rate improvement vs cost
- Diminishing returns after ~95% hit rate"

---

## 4️⃣ Messaging Follow-ups

### Follow-up 1: "How do you ensure message ordering?"

**Answer**:
"Message ordering depends on the scope needed:

**Global ordering** (all messages in order):
- Single partition/queue
- Limits throughput to one consumer
- Rarely needed

**Per-key ordering** (messages for same entity in order):
- Partition by key (user_id, order_id)
- All messages for same key go to same partition
- Kafka does this naturally

**Implementation in Kafka**:
```
// Producer: same key → same partition
producer.send(new ProducerRecord<>(topic, orderId, message));

// Consumer: one consumer per partition
// Messages within partition are ordered
```

**Causal ordering** (related messages in order):
- Include causality information in message
- Consumer reorders if needed
- More complex but allows parallelism

**When ordering matters**:
- Order status updates (can't ship before payment)
- User actions (can't unlike before like)
- Financial transactions

**When ordering doesn't matter**:
- Independent events (page views)
- Idempotent operations"

### Follow-up 2: "How do you handle duplicate messages?"

**Answer**:
"Duplicates occur due to retries, network issues, or producer failures.

**At-least-once delivery** (most common):
- Messages may be delivered multiple times
- Consumer must handle duplicates

**Idempotency strategies**:

1. **Idempotency key**:
   - Each message has unique ID
   - Consumer tracks processed IDs
   - Skip if already processed

```java
public void processMessage(Message msg) {
    if (processedIds.contains(msg.getId())) {
        return; // Already processed
    }
    // Process message
    processedIds.add(msg.getId());
}
```

2. **Idempotent operations**:
   - Design operations to be naturally idempotent
   - SET user.balance = 100 (idempotent)
   - vs INCREMENT user.balance by 10 (not idempotent)

3. **Database constraints**:
   - Unique constraint on message ID
   - Insert fails if duplicate

**Exactly-once semantics**:
- Kafka supports with transactions
- More complex and slower
- Use only when truly needed"

### Follow-up 3: "What happens if a consumer is slow?"

**Answer**:
"Slow consumers cause backpressure. Handling depends on the system:

**Detection**:
- Monitor consumer lag (messages waiting)
- Alert when lag exceeds threshold

**Strategies**:

1. **Scale consumers**:
   - Add more consumers (up to partition count)
   - Each consumer handles subset of partitions

2. **Increase consumer throughput**:
   - Batch processing
   - Async processing within consumer
   - Optimize processing logic

3. **Backpressure to producer**:
   - Queue fills up
   - Producer blocks or drops messages
   - Protects system from overload

4. **Dead letter queue**:
   - Move problematic messages aside
   - Process rest of queue
   - Handle DLQ separately

**Kafka specifics**:
- Consumer lag metric shows how far behind
- Can pause consumption to catch up
- Retention period determines how long messages wait"

---

## 5️⃣ Consistency Follow-ups

### Follow-up 1: "What if there's a network partition?"

**Answer**:
"Network partitions force a choice between consistency and availability (CAP theorem).

**Scenario**: Database primary in US-East, replica in US-West, network between them fails.

**Option 1: Choose Consistency (CP)**:
- Reject writes to partitioned replica
- Users in US-West see errors or read-only mode
- No risk of conflicting writes

**Option 2: Choose Availability (AP)**:
- Both sides accept writes
- Risk of conflicts when partition heals
- Need conflict resolution strategy

**Conflict resolution**:
- Last-write-wins (simple but may lose data)
- Merge (for compatible changes)
- Manual resolution (for critical data)

**Our approach for this system**:
- Critical data (payments): CP, reject writes during partition
- Non-critical data (likes, views): AP, accept writes, merge later
- User sessions: AP with last-write-wins

**Detection and recovery**:
- Monitor replication lag
- Alert on partition detection
- Automatic reconciliation when partition heals"

### Follow-up 2: "How do you handle distributed transactions?"

**Answer**:
"Distributed transactions across services are challenging. Options:

**Option 1: Two-Phase Commit (2PC)**
- Coordinator asks all participants to prepare
- If all say yes, coordinator says commit
- Pros: Strong consistency
- Cons: Blocking, single point of failure, doesn't scale

**Option 2: Saga Pattern**
- Break transaction into local transactions
- Each step has compensating action
- If step fails, run compensations

```
Order Saga:
1. Reserve inventory → Compensate: Release inventory
2. Charge payment → Compensate: Refund payment
3. Create shipment → Compensate: Cancel shipment
```

**Option 3: Eventual consistency**
- Accept temporary inconsistency
- Use events to propagate changes
- Reconcile asynchronously

**For this system**, I'd use saga pattern because:
- We have multiple services (inventory, payment, shipping)
- 2PC doesn't scale to our throughput
- We can define compensating actions
- Brief inconsistency during saga is acceptable"

---

## 6️⃣ Scale Follow-ups

### Follow-up 1: "How do you handle 10x growth?"

**Answer**:
"10x growth requires systematic scaling:

**Step 1: Identify current bottlenecks**
- Load test to find breaking point
- Usually database, then application, then network

**Step 2: Database scaling**
- Add read replicas (if read-heavy)
- Implement sharding (if write-heavy)
- Add caching layer

**Step 3: Application scaling**
- Ensure stateless (no local state)
- Auto-scaling based on CPU/memory
- Optimize hot paths

**Step 4: Async processing**
- Move non-critical work to queues
- Reduces response time
- Handles spikes better

**Step 5: Caching**
- Add/expand caching layers
- CDN for static content
- Application-level caching

**Step 6: Geographic distribution**
- Multi-region if users are global
- Reduces latency
- Increases availability

**Timeline**:
- Quick wins (caching, optimization): 1-2 weeks
- Read replicas: 1 week
- Sharding: 1-3 months
- Multi-region: 3-6 months"

### Follow-up 2: "What's your capacity planning process?"

**Answer**:
"Capacity planning involves forecasting and preparing:

**Forecasting**:
1. Analyze historical growth trends
2. Account for planned launches/promotions
3. Add safety margin (typically 2x)

**Metrics to track**:
- Current utilization (CPU, memory, DB connections)
- Growth rate (users, requests, data)
- Headroom remaining

**Planning process**:
```
Current: 10,000 QPS, 60% DB capacity
Growth: 20% month-over-month
Projection: 6 months → 30,000 QPS

Action needed: DB upgrade before month 4
(when we hit 80% capacity)
```

**Automation**:
- Auto-scaling for application tier
- Alerts when approaching capacity limits
- Regular capacity review meetings

**Cost optimization**:
- Right-size instances
- Use reserved instances for baseline
- Spot/preemptible for burst capacity"

---

## 7️⃣ Quick Reference: 50+ Follow-ups

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FOLLOW-UP QUESTIONS BY TOPIC                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DATABASE                                                           │
│  □ How do you handle hot partitions?                                │
│  □ How do you handle schema migrations?                             │
│  □ SQL vs NoSQL - how did you decide?                               │
│  □ How do you handle database failover?                             │
│  □ How do you handle backup and recovery?                           │
│  □ What indexes would you create?                                   │
│  □ How do you handle connection pooling?                            │
│  □ What's your replication strategy?                                │
│                                                                      │
│  CACHE                                                              │
│  □ What if the cache goes down?                                     │
│  □ How do you handle cache invalidation?                            │
│  □ How do you size your cache?                                      │
│  □ How do you handle cache stampede?                                │
│  □ What's your eviction policy?                                     │
│  □ How do you warm up the cache?                                    │
│  □ Cache-aside vs write-through?                                    │
│                                                                      │
│  MESSAGING                                                          │
│  □ How do you ensure message ordering?                              │
│  □ How do you handle duplicate messages?                            │
│  □ What happens if a consumer is slow?                              │
│  □ How do you handle poison messages?                               │
│  □ What's your retry strategy?                                      │
│  □ How do you handle dead letter queues?                            │
│  □ Kafka vs RabbitMQ - why?                                         │
│                                                                      │
│  CONSISTENCY                                                        │
│  □ What if there's a network partition?                             │
│  □ How do you handle distributed transactions?                      │
│  □ Strong vs eventual consistency - why?                            │
│  □ How do you handle conflicts?                                     │
│  □ What's your isolation level?                                     │
│  □ How do you ensure exactly-once processing?                       │
│                                                                      │
│  SCALE                                                              │
│  □ How do you handle 10x growth?                                    │
│  □ What's your capacity planning process?                           │
│  □ What's the first bottleneck?                                     │
│  □ How do you handle traffic spikes?                                │
│  □ Horizontal vs vertical scaling?                                  │
│  □ How do you handle global users?                                  │
│                                                                      │
│  RELIABILITY                                                        │
│  □ What if [component] fails?                                       │
│  □ How do you handle cascading failures?                            │
│  □ What's your disaster recovery plan?                              │
│  □ How do you handle data loss?                                     │
│  □ What's your SLA/SLO?                                             │
│  □ How do you handle partial failures?                              │
│                                                                      │
│  SECURITY                                                           │
│  □ How do you handle authentication?                                │
│  □ How do you handle authorization?                                 │
│  □ How do you protect sensitive data?                               │
│  □ How do you handle rate limiting?                                 │
│  □ How do you prevent DDoS?                                         │
│                                                                      │
│  OPERATIONS                                                         │
│  □ How do you monitor this?                                         │
│  □ How do you handle deployments?                                   │
│  □ What alerts would you set up?                                    │
│  □ How do you debug production issues?                              │
│  □ What's your on-call process?                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ One Clean Mental Summary

Follow-up questions are predictable by topic area. For databases: hot partitions, migrations, failover. For caches: invalidation, sizing, failures. For messaging: ordering, duplicates, slow consumers. For consistency: partitions, distributed transactions. For scale: 10x growth, capacity planning.

Prepare structured answers for each topic area. Know the trade-offs of your choices. Be ready to explain why you chose one approach over alternatives. The follow-ups are where you demonstrate depth beyond surface-level knowledge.

Practice until you can answer these questions confidently and concisely. The interviewer is looking for evidence that you've thought deeply about these problems, not just memorized definitions.

