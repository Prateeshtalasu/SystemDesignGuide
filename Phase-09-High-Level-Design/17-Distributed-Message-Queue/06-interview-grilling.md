# Distributed Message Queue - Interview Grilling

## 1. Core Design Trade-offs

### Q1: Why use a pull-based model instead of push-based for consumers?

**Answer:**

```
Pull-based (Kafka approach):
├── Consumer controls rate
├── Natural backpressure
├── Batching efficiency (fetch many at once)
├── Consumer can rewind/replay
└── Simpler broker (no consumer state tracking)

Push-based (RabbitMQ approach):
├── Lower latency (immediate delivery)
├── Broker manages distribution
├── Risk of overwhelming slow consumers
└── Complex flow control needed

Why Kafka chose pull:
1. Throughput: Batching is more efficient
2. Backpressure: Consumer naturally throttles
3. Replayability: Consumer controls position
4. Simplicity: Broker doesn't track consumer state

Trade-off: Slightly higher latency (polling interval)
Mitigation: Long polling (fetch.max.wait.ms)
```

### Q2: Why does Kafka use a log-structured storage model?

**Answer:**

```
Log-structured storage:
├── Sequential writes only (append-only)
├── No random I/O for writes
├── Leverages disk sequential throughput (600+ MB/s)
├── Simple crash recovery (replay from checkpoint)
└── Natural ordering guarantee

Alternative: B-tree or LSM-tree
├── Supports updates/deletes natively
├── Better for random access patterns
├── More complex implementation
└── Higher write amplification

Why log works for messaging:
1. Messages are immutable (no updates)
2. Access is mostly sequential (consume in order)
3. Disk is cheap, RAM is expensive
4. OS page cache handles hot data
5. Retention-based deletion (no compaction overhead for delete policy)

Performance impact:
- Write: 100K+ messages/sec per partition
- Read: Limited by network, not disk
- Storage: Linear growth, predictable costs
```

### Q3: How do you choose between at-least-once, at-most-once, and exactly-once?

**Answer:**

```
At-most-once (acks=0):
├── Fire and forget
├── May lose messages
├── Highest throughput
├── Use for: Metrics, logs, non-critical data
└── Example: Application telemetry

At-least-once (acks=all, no idempotence):
├── Messages delivered at least once
├── May have duplicates on retry
├── Good throughput
├── Use for: Most use cases with idempotent consumers
└── Example: Event sourcing with deduplication

Exactly-once (acks=all + idempotence + transactions):
├── Each message processed exactly once
├── Highest latency
├── Most complex
├── Use for: Financial transactions, inventory
└── Example: Payment processing

Decision framework:
1. Can you tolerate message loss? → No → Not at-most-once
2. Can consumers handle duplicates? → Yes → At-least-once
3. Need atomic multi-topic writes? → Yes → Exactly-once
4. Is latency critical? → Yes → Avoid exactly-once

Real-world: 90% of use cases work with at-least-once + idempotent consumers
```

### Q4: Why partition data instead of using a single queue?

**Answer:**

```
Single queue limitations:
├── Single consumer at a time (for ordering)
├── Vertical scaling only
├── Single point of failure
├── Limited throughput (~10K msg/s)
└── No parallelism

Partitioning benefits:
├── Horizontal scaling (add partitions)
├── Parallel consumption (consumer per partition)
├── Higher throughput (partitions × partition_throughput)
├── Fault isolation (partition failure is partial)
└── Data locality (related data together)

Partitioning trade-offs:
├── Ordering only within partition
├── Rebalancing complexity
├── Cannot reduce partition count
├── Uneven distribution possible (hot partitions)
└── More metadata overhead

Partition key strategy:
- User ID: All user events together
- Order ID: All order events together
- Random: Even distribution, no ordering
- Timestamp: Time-based partitioning

Rule of thumb:
- Partitions = max(throughput_needed / 10MB/s, consumer_count)
- Start with more partitions (can't decrease)
- Monitor for hot partitions
```

---

## 2. Scaling Questions

### Q5: How would you handle a 10x traffic spike?

**Answer:**

```
Immediate actions (no infra changes):
├── Increase producer batch size (batch.size)
├── Increase consumer parallelism (if partitions available)
├── Enable compression (compression.type=lz4)
├── Reduce acks (if acceptable: acks=1)
└── Increase fetch size (fetch.min.bytes)

Short-term (minutes to hours):
├── Add consumer instances (up to partition count)
├── Scale broker instances (if bottleneck)
├── Increase partition count (for new data)
└── Add broker capacity (vertical scaling)

Medium-term (hours to days):
├── Add more brokers
├── Rebalance partitions across brokers
├── Increase replication factor if needed
└── Add consumer groups for parallel processing

Long-term (architectural):
├── Topic sharding (multiple topics by tenant/region)
├── Tiered storage for cost efficiency
├── Multi-cluster with MirrorMaker
└── Dedicated clusters for high-volume topics

Key metrics to watch:
- Broker CPU and network utilization
- Consumer lag
- Produce latency (p99)
- Under-replicated partitions
```

### Q6: How do you handle hot partitions?

**Answer:**

```
Hot partition causes:
├── Skewed partition key (popular user/item)
├── Poor key distribution
├── Time-based keys (current hour)
└── Single key for ordering requirement

Detection:
├── Monitor per-partition metrics
├── BytesInPerSec per partition
├── Consumer lag per partition
└── Broker disk I/O imbalance

Solutions:

1. Salted keys (add random suffix):
   Original: "user-123"
   Salted: "user-123-{0-9}" (10 sub-partitions)
   
   Trade-off: Lose strict ordering, need aggregation

2. Dedicated partitions:
   Route hot keys to specific partitions
   Scale those partitions independently
   
3. Separate topics:
   High-volume entities get own topic
   Different retention/scaling policies

4. Custom partitioner:
   class CustomPartitioner implements Partitioner {
       public int partition(...) {
           if (isHotKey(key)) {
               return hash(key + timestamp) % hotPartitions;
           }
           return hash(key) % normalPartitions;
       }
   }

5. Pre-aggregation:
   Aggregate events before producing
   Reduces message count for hot keys
```

### Q7: How would you design for multi-region deployment?

**Answer:**

```
Option 1: Stretched Cluster
┌─────────────────────────────────────────────────────────────┐
│  Region A          Region B          Region C              │
│  ┌────────┐       ┌────────┐       ┌────────┐             │
│  │Broker 1│◄─────►│Broker 2│◄─────►│Broker 3│             │
│  └────────┘       └────────┘       └────────┘             │
│                                                            │
│  Pros: Automatic failover, single cluster                 │
│  Cons: High latency (cross-region sync), complex networking│
│  Use: When strong consistency required                     │
└─────────────────────────────────────────────────────────────┘

Option 2: Active-Passive with MirrorMaker
┌─────────────────────────────────────────────────────────────┐
│  Region A (Active)              Region B (Passive)         │
│  ┌────────────┐                 ┌────────────┐             │
│  │ Cluster A  │ ──MirrorMaker──►│ Cluster B  │             │
│  └────────────┘                 └────────────┘             │
│                                                            │
│  Pros: Low latency, simple                                │
│  Cons: Manual failover, data lag                          │
│  Use: DR scenarios                                        │
└─────────────────────────────────────────────────────────────┘

Option 3: Active-Active with Conflict Resolution
┌─────────────────────────────────────────────────────────────┐
│  Region A                       Region B                   │
│  ┌────────────┐                 ┌────────────┐             │
│  │ Cluster A  │◄──MirrorMaker──►│ Cluster B  │             │
│  └────────────┘                 └────────────┘             │
│       ▲                               ▲                    │
│       │                               │                    │
│  Local Producers              Local Producers              │
│                                                            │
│  Pros: Low latency, high availability                     │
│  Cons: Conflict resolution needed, complex                │
│  Use: Global applications                                 │
└─────────────────────────────────────────────────────────────┘

Conflict resolution strategies:
- Last-write-wins (timestamp-based)
- Topic prefixing (region-specific topics)
- Application-level merge
```

---

## 3. Failure Scenarios

### Q8: What happens when a broker fails mid-produce with acks=all?

**Answer:**

```
Scenario timeline:

T0: Producer sends produce request
T1: Leader receives, writes to local log
T2: Follower 1 replicates successfully
T3: Leader crashes before responding

Without idempotence:
├── Producer times out (request.timeout.ms)
├── Producer retries to new leader
├── New leader accepts (different sequence)
├── Result: Duplicate message
└── Consumer sees message twice

With idempotence (enable.idempotence=true):
├── Producer assigned PID and sequence number
├── Producer times out
├── Producer retries with same sequence
├── New leader checks: already have seq N?
│   ├── Yes → Return success (deduplicated)
│   └── No → Accept and store
├── Result: Exactly one message
└── Consumer sees message once

Key configurations:
- enable.idempotence=true
- acks=all
- retries=MAX_INT
- max.in.flight.requests.per.connection=5

Why max.in.flight=5 with idempotence?
- Broker tracks last 5 sequence numbers per partition
- Allows batching without ordering issues
- Higher values risk out-of-order on retry
```

### Q9: How do you handle consumer rebalancing during processing?

**Answer:**

```
Problem: Consumer processing message when rebalance triggered
- Partition revoked mid-processing
- Another consumer gets partition
- Original consumer commits → offset already moved
- Risk: Duplicate processing or lost messages

Solution 1: Cooperative Rebalancing
├── partition.assignment.strategy=CooperativeStickyAssignor
├── Incremental rebalance (only affected partitions)
├── No stop-the-world
├── Reduces duplicate processing window
└── Default in newer Kafka versions

Solution 2: Rebalance Listener
class RebalanceHandler implements ConsumerRebalanceListener {
    
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Commit processed offsets before revocation
        commitProcessedOffsets();
        
        // Cancel in-flight processing
        processingTasks.forEach(Task::cancel);
        
        // Wait for graceful completion
        awaitTaskCompletion(timeout);
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Seek to last committed offset
        for (TopicPartition tp : partitions) {
            long offset = getLastCommittedOffset(tp);
            consumer.seek(tp, offset);
        }
    }
}

Solution 3: Idempotent Processing
├── Store processing state with message ID
├── Check before processing: already processed?
├── Deduplication at consumer level
└── Works regardless of rebalance timing

Best practice combination:
1. Use cooperative rebalancing
2. Implement rebalance listener
3. Design idempotent consumers
4. Use exactly-once when critical
```

### Q10: What happens during a network partition?

**Answer:**

```
Scenario: Network partition splits cluster

Partition: [Broker1, Broker2] | [Broker3, Controller]

Impact on Controller side (Broker3):
├── Broker3 continues operating
├── Partitions with leader on Broker3: Available
├── Controller marks Broker1, Broker2 as dead
├── Triggers leader election for affected partitions
└── If no ISR replica on Broker3 → partition offline

Impact on isolated side (Broker1, Broker2):
├── Cannot reach controller
├── Cannot participate in elections
├── Existing leaders continue serving (stale metadata)
├── Produces may succeed (if leader local)
├── Consumes may succeed (if leader local)
└── Risk: Split-brain if both sides have leaders

Split-brain prevention:
├── min.insync.replicas=2 (majority)
├── Isolated side cannot achieve ISR quorum
├── Produces fail with NOT_ENOUGH_REPLICAS
├── Prevents divergent writes
└── Availability sacrificed for consistency

Recovery:
1. Network heals
2. Isolated brokers reconnect
3. Detect divergent logs via leader epoch
4. Truncate to common point (HW)
5. Re-sync from current leader
6. Rejoin ISR

Data loss scenarios:
- unclean.leader.election.enable=true
- Minority partition elected leader
- Majority partition's writes lost on recovery
- Solution: Keep unclean.leader.election.enable=false
```

---

## 4. Design Evolution

### Q11: How would you evolve from single-cluster to multi-cluster?

**Answer:**

```
Stage 1: Single Cluster
├── All producers/consumers in one cluster
├── Simple operations
├── Limited by single region
└── Single point of failure

Stage 2: Add DR Cluster
├── MirrorMaker 2.0 replication
├── Async replication to DR region
├── Manual failover procedures
├── Test failover regularly
└── RPO: Minutes, RTO: Hours

Stage 3: Active-Active
├── Bidirectional replication
├── Topic naming: {region}-{topic}
├── Producers write to local cluster
├── Consumers aggregate from both
└── Conflict resolution strategy

Stage 4: Global Cluster Federation
├── Cluster per region
├── Central metadata service
├── Intelligent routing
├── Follow-the-sun processing
└── Global consumer groups

Migration strategy:
1. Deploy MirrorMaker 2.0
2. Configure topic replication
3. Validate data consistency
4. Gradually shift traffic
5. Monitor lag and throughput
6. Implement failover automation

Key challenges:
- Offset translation between clusters
- Consumer group coordination
- Schema compatibility
- Monitoring across clusters
```

### Q12: How would you add exactly-once semantics to an existing system?

**Answer:**

```
Current state: At-least-once with manual deduplication

Step 1: Enable Idempotent Producers
├── enable.idempotence=true
├── No code changes needed
├── Transparent deduplication
├── Prerequisite: acks=all, retries>0
└── Risk: None, backward compatible

Step 2: Upgrade Consumer Processing
├── Implement idempotent handlers
├── Use message ID for deduplication
├── Store processed IDs in database
├── Atomic: process + mark processed
└── Works with existing infrastructure

Step 3: Add Transactional Producers (where needed)
├── Identify atomic multi-topic writes
├── Add transactional.id
├── Wrap in beginTransaction/commitTransaction
├── Update consumers: isolation.level=read_committed
└── Gradual rollout by use case

Step 4: Implement Read-Process-Write Pattern
├── For stream processing use cases
├── sendOffsetsToTransaction()
├── Atomic: read + process + write + commit
├── Requires careful error handling
└── Consider Kafka Streams for simplicity

Migration considerations:
- Backward compatibility during transition
- Performance impact of transactions
- Increased latency (2PC overhead)
- Monitoring transaction metrics
- Rollback plan if issues arise

Timeline:
- Step 1: 1 week (config change)
- Step 2: 2-4 weeks (code changes)
- Step 3: 4-8 weeks (architecture changes)
- Step 4: 8-12 weeks (major refactoring)
```

---

## 5. Level-Specific Expectations

### Junior Engineer (0-2 years)

**Expected to know:**
- Basic producer/consumer API
- Topic and partition concepts
- Consumer groups basics
- Simple configuration (bootstrap.servers, group.id)

**Sample question:** "Explain how a consumer group works"

**Good answer:**
```
Consumer group allows multiple consumers to share work:
- Each partition assigned to one consumer
- Adding consumers increases parallelism
- If consumer fails, partitions reassigned
- Offset tracked per group

Example: 4 partitions, 2 consumers
- Consumer 1: Partitions 0, 1
- Consumer 2: Partitions 2, 3

If Consumer 2 fails:
- Consumer 1 gets all 4 partitions
- Continues from last committed offset
```

### Mid-Level Engineer (2-5 years)

**Expected to know:**
- Replication and ISR concepts
- Exactly-once semantics
- Performance tuning
- Monitoring and alerting
- Basic failure scenarios

**Sample question:** "How would you debug high consumer lag?"

**Good answer:**
```
Debugging approach:

1. Identify scope
   - Which consumer group?
   - Which partitions?
   - When did it start?

2. Check consumer health
   - Are consumers running?
   - JVM heap/GC issues?
   - Processing exceptions?

3. Analyze processing time
   - Time per message?
   - External dependencies slow?
   - Database queries?

4. Check for rebalancing
   - Frequent rebalances?
   - session.timeout.ms too low?
   - max.poll.interval.ms exceeded?

5. Verify broker health
   - Under-replicated partitions?
   - Network issues?
   - Disk I/O saturation?

Solutions by cause:
- Slow processing → Optimize code, add consumers
- Frequent rebalance → Tune timeouts, use cooperative
- Broker issues → Scale brokers, rebalance partitions
- Traffic spike → Scale consumers, increase partitions
```

### Senior Engineer (5+ years)

**Expected to know:**
- Distributed systems theory
- Multi-region architecture
- Cost optimization
- Capacity planning
- Complex failure scenarios
- Trade-off analysis

**Sample question:** "Design a message queue that guarantees exactly-once delivery across multiple regions with <100ms latency"

**Good answer:**
```
This is a challenging requirement with inherent trade-offs:

Analysis:
- Exactly-once requires coordination
- Cross-region adds latency (50-200ms typically)
- <100ms conflicts with cross-region consensus

Proposed architecture:

1. Regional clusters with local exactly-once
   ├── Each region has independent Kafka cluster
   ├── Producers write to local cluster
   ├── Local exactly-once with transactions
   └── Latency: <20ms for local operations

2. Async cross-region replication
   ├── MirrorMaker 2.0 for replication
   ├── Eventual consistency across regions
   ├── RPO: Replication lag (seconds)
   └── Not exactly-once across regions

3. For cross-region exactly-once (where needed)
   ├── Two-phase commit with coordinator
   ├── Latency: 100-300ms (cannot meet <100ms)
   ├── Use sparingly for critical transactions
   └── Alternative: Saga pattern with compensation

Trade-off decision:
- Accept eventual consistency for most data
- Use local exactly-once + idempotent consumers
- Cross-region deduplication at application layer
- Reserve 2PC for critical financial transactions

Honest assessment:
"True exactly-once with <100ms across regions violates 
CAP theorem. We must choose between consistency and 
latency. I recommend local exactly-once with eventual 
cross-region consistency for most use cases."
```

### Staff+ Engineer (8+ years)

**Expected to know:**
- System design at scale
- Organizational considerations
- Build vs. buy decisions
- Long-term evolution
- Cross-team coordination

**Sample question:** "Should we build our own message queue or use Kafka/managed service?"

**Good answer:**
```
Decision framework:

Build custom when:
├── Unique requirements not met by existing solutions
├── Core competitive advantage
├── Team has distributed systems expertise
├── Long-term cost savings justify investment
└── Need deep integration with proprietary systems

Use Kafka when:
├── Standard messaging patterns
├── Large ecosystem (connectors, streams)
├── Well-understood operational model
├── Community support and documentation
└── Team familiar with Kafka

Use managed service (Confluent, MSK) when:
├── Limited ops capacity
├── Faster time to market
├── Predictable costs preferred
├── Compliance/security requirements
└── Multi-cloud strategy

Cost analysis (1B messages/day):

Self-managed Kafka:
- Infrastructure: $30K/month
- Engineering: 2 FTE × $15K/month = $30K/month
- Total: $60K/month

Managed service:
- Confluent Cloud: $50-80K/month
- No dedicated engineering
- Total: $50-80K/month

Custom build:
- Initial development: $500K-1M
- Infrastructure: $20K/month
- Engineering: 3 FTE × $15K/month = $45K/month
- Total: $65K/month + amortized dev cost

Recommendation for most companies:
"Start with managed Kafka. Migrate to self-managed 
if scale justifies. Only build custom if you have 
unique requirements AND distributed systems expertise."

Red flags for custom build:
- "Kafka is too complex for our needs"
- "We can build something simpler"
- "It's just a queue, how hard can it be?"
```

---

## 6. Quick-Fire Questions

| Question | Key Points |
|----------|------------|
| Why is Kafka fast? | Sequential I/O, zero-copy, batching, compression, page cache |
| What's the CAP trade-off? | CP system: Consistency over availability during partition |
| How does consumer offset work? | Stored in __consumer_offsets topic, compacted |
| What's ISR? | In-Sync Replicas within lag threshold of leader |
| Why use ZooKeeper/KRaft? | Cluster metadata, leader election, configuration |
| What's a partition leader epoch? | Monotonic counter to detect stale leaders |
| How does log compaction work? | Keeps latest value per key, removes older versions |
| What's the high watermark? | Last offset replicated to all ISR |
| Why min.insync.replicas? | Ensures durability before acknowledging |
| What triggers rebalance? | Member join/leave, subscription change, partition change |

