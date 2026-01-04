# Distributed Message Queue - Production Deep Dives (Core)

## 1. Replication Deep Dive

### Why Replication Exists

```
Problem without replication:
├── Single broker failure = data loss
├── No fault tolerance
├── Single point of failure
└── Cannot survive disk failures

What replication provides:
├── Durability: Data survives broker failures
├── Availability: Partitions remain accessible
├── Fault tolerance: System continues operating
└── Read scaling: Followers can serve reads (optional)
```

### How Replication Works

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Replication Mechanism                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Leader-Based Replication:                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  Producer ───────> Leader ───────> Follower 1                      │      │
│  │                      │                                              │      │
│  │                      └───────────> Follower 2                      │      │
│  │                                                                     │      │
│  │  1. All writes go to leader                                        │      │
│  │  2. Followers pull from leader                                     │      │
│  │  3. Leader tracks ISR (in-sync replicas)                          │      │
│  │  4. High watermark = min(ISR LEOs)                                │      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Key Concepts:                                                                │
│  ├── LEO (Log End Offset): Last offset in replica's log                      │
│  ├── HW (High Watermark): Last committed offset (visible to consumers)       │
│  ├── ISR: Replicas within replica.lag.time.max.ms of leader                 │
│  └── Leader Epoch: Monotonic counter for leader changes                      │
│                                                                               │
│  ISR Management:                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  Follower added to ISR when:                                       │      │
│  │  - Caught up to leader's LEO                                       │      │
│  │  - Fetch request within lag threshold                              │      │
│  │                                                                     │      │
│  │  Follower removed from ISR when:                                   │      │
│  │  - No fetch request for replica.lag.time.max.ms (30s default)     │      │
│  │  - Falls too far behind (legacy: replica.lag.max.messages)        │      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Replication Simulation

```java
// Scenario: acks=all produce with 3 replicas
public class ReplicationSimulation {
    
    public void simulateProduce() {
        // Initial state
        // Leader: LEO=100, HW=100
        // Follower1: LEO=100, HW=100
        // Follower2: LEO=100, HW=100
        // ISR=[Leader, F1, F2]
        
        // Step 1: Producer sends batch (10 messages)
        ProduceRequest request = new ProduceRequest(
            "orders", 0, batch, Acks.ALL);
        
        // Step 2: Leader writes to log
        // Leader: LEO=110, HW=100 (unchanged)
        leaderLog.append(batch);
        
        // Step 3: Followers fetch
        // Follower1: FetchRequest(offset=100)
        // Leader responds with records 100-109
        
        // Step 4: Followers write to log
        // Follower1: LEO=110
        // Follower2: LEO=110
        
        // Step 5: Leader receives next fetch from followers
        // Followers report their LEO in fetch request
        // Leader calculates: HW = min(110, 110, 110) = 110
        
        // Step 6: Leader advances HW to 110
        // Leader: LEO=110, HW=110
        
        // Step 7: Leader responds to producer
        // ProduceResponse(offset=100, error=NONE)
        
        // Step 8: Followers learn new HW from next fetch response
        // Follower1: HW=110
        // Follower2: HW=110
    }
}
```

### Failure Scenarios

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Replication Failure Scenarios                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Scenario 1: Follower Failure                                                 │
│  ─────────────────────────────────────────────────────────────────────────── │
│  Initial: ISR=[L, F1, F2], min.insync.replicas=2                             │
│                                                                               │
│  1. F2 crashes                                                               │
│  2. Controller detects via heartbeat timeout                                 │
│  3. F2 removed from ISR: ISR=[L, F1]                                        │
│  4. Produces continue (ISR size >= min.insync.replicas)                     │
│  5. F2 recovers, fetches from leader                                        │
│  6. F2 catches up, added back to ISR                                        │
│                                                                               │
│  Impact: None (if ISR >= min.insync.replicas)                               │
│                                                                               │
│  Scenario 2: Leader Failure                                                   │
│  ─────────────────────────────────────────────────────────────────────────── │
│  Initial: ISR=[L, F1, F2]                                                    │
│                                                                               │
│  1. Leader crashes                                                           │
│  2. Controller detects via heartbeat                                        │
│  3. Controller elects new leader from ISR (F1)                              │
│  4. F1 becomes leader, increments leader epoch                              │
│  5. Producers/consumers discover new leader via metadata refresh            │
│  6. Old leader recovers, truncates to HW, becomes follower                  │
│                                                                               │
│  Impact: Brief unavailability during election (~seconds)                     │
│                                                                               │
│  Scenario 3: ISR Shrinks Below min.insync.replicas                          │
│  ─────────────────────────────────────────────────────────────────────────── │
│  Initial: ISR=[L, F1, F2], min.insync.replicas=2                             │
│                                                                               │
│  1. F1 and F2 both lag/fail                                                 │
│  2. ISR=[L] (size=1 < min.insync.replicas=2)                               │
│  3. Produces with acks=all fail: NOT_ENOUGH_REPLICAS                        │
│  4. Produces with acks=1 succeed (if enabled)                               │
│  5. Consumers can still read up to HW                                       │
│                                                                               │
│  Impact: Write unavailability for acks=all                                   │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Consumer Group Coordination

### Why Consumer Groups Exist

```
Problem without coordination:
├── Manual partition assignment
├── No automatic failover
├── Duplicate processing
├── No load balancing
└── Complex offset management

What consumer groups provide:
├── Automatic partition assignment
├── Consumer failure detection
├── Rebalancing on membership changes
├── Coordinated offset commits
└── Exactly-once semantics (with transactions)
```

### Consumer Group Protocol

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Consumer Group Protocol                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Group Coordinator:                                                           │
│  - One broker per consumer group                                             │
│  - Determined by: hash(group.id) % __consumer_offsets partitions            │
│  - Manages group membership and offsets                                      │
│                                                                               │
│  Protocol Flow:                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  Consumer                         Coordinator                       │      │
│  │     │                                  │                           │      │
│  │     │  1. FindCoordinator              │                           │      │
│  │     │─────────────────────────────────>│                           │      │
│  │     │                                  │                           │      │
│  │     │  2. JoinGroup                    │                           │      │
│  │     │  (subscriptions, protocols)      │                           │      │
│  │     │─────────────────────────────────>│                           │      │
│  │     │                                  │                           │      │
│  │     │  [Coordinator waits for all      │                           │      │
│  │     │   members or rebalance timeout]  │                           │      │
│  │     │                                  │                           │      │
│  │     │  3. JoinGroup Response           │                           │      │
│  │     │  (leader gets member list)       │                           │      │
│  │     │<─────────────────────────────────│                           │      │
│  │     │                                  │                           │      │
│  │     │  4. SyncGroup                    │                           │      │
│  │     │  (leader sends assignments)      │                           │      │
│  │     │─────────────────────────────────>│                           │      │
│  │     │                                  │                           │      │
│  │     │  5. SyncGroup Response           │                           │      │
│  │     │  (partition assignment)          │                           │      │
│  │     │<─────────────────────────────────│                           │      │
│  │     │                                  │                           │      │
│  │     │  6. Heartbeat (periodic)         │                           │      │
│  │     │─────────────────────────────────>│                           │      │
│  │     │                                  │                           │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Partition Assignment Strategies:                                             │
│  ├── Range: Assign partitions in ranges per topic                           │
│  ├── RoundRobin: Distribute partitions evenly                               │
│  ├── Sticky: Minimize partition movement                                     │
│  └── CooperativeSticky: Incremental rebalancing                             │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Rebalancing Deep Dive

```java
// Rebalance Triggers
public enum RebalanceTrigger {
    MEMBER_JOIN,        // New consumer joins
    MEMBER_LEAVE,       // Consumer leaves (graceful or timeout)
    SUBSCRIPTION_CHANGE,// Topics added/removed
    PARTITION_CHANGE    // Topic partition count changed
}

// Rebalance Process
public class RebalanceSimulation {
    
    public void simulateRebalance() {
        // Initial state: 2 consumers, 4 partitions
        // Consumer1: [P0, P1]
        // Consumer2: [P2, P3]
        
        // Step 1: Consumer3 joins
        // Coordinator receives JoinGroup from Consumer3
        
        // Step 2: Coordinator triggers rebalance
        // All consumers receive REBALANCE_IN_PROGRESS on heartbeat
        
        // Step 3: Consumers revoke partitions
        // Consumer1 commits offsets, stops processing
        // Consumer2 commits offsets, stops processing
        
        // Step 4: All consumers send JoinGroup
        // Coordinator waits for all (max.poll.interval.ms)
        
        // Step 5: Leader (Consumer1) calculates new assignment
        // Using StickyAssignor to minimize movement:
        // Consumer1: [P0]         (kept P0, released P1)
        // Consumer2: [P2, P3]     (kept both)
        // Consumer3: [P1]         (got P1)
        
        // Step 6: Leader sends assignment via SyncGroup
        // Coordinator distributes to all members
        
        // Step 7: Consumers resume processing
        // Each consumer fetches from assigned partitions
    }
}
```

### Consumer Offset Management

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Offset Management                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  __consumer_offsets Topic:                                                    │
│  - Internal topic for storing consumer offsets                               │
│  - 50 partitions by default                                                  │
│  - Compacted (keeps latest offset per consumer-topic-partition)             │
│                                                                               │
│  Offset Commit Record:                                                        │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │ Key: [group, topic, partition]                                      │      │
│  │ Value: {                                                            │      │
│  │   "offset": 12345,                                                  │      │
│  │   "metadata": "",                                                   │      │
│  │   "commit_timestamp": 1705320600000,                               │      │
│  │   "expire_timestamp": -1                                           │      │
│  │ }                                                                   │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Commit Strategies:                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  1. Auto Commit (enable.auto.commit=true)                          │      │
│  │     - Commits periodically (auto.commit.interval.ms)               │      │
│  │     - Risk: Message loss on crash (committed but not processed)   │      │
│  │                                                                     │      │
│  │  2. Manual Sync Commit (commitSync)                                │      │
│  │     - Blocks until commit succeeds                                 │      │
│  │     - Guarantees at-least-once                                     │      │
│  │     - Higher latency                                               │      │
│  │                                                                     │      │
│  │  3. Manual Async Commit (commitAsync)                              │      │
│  │     - Non-blocking                                                 │      │
│  │     - May lose commits on failure                                  │      │
│  │     - Use with callback for error handling                        │      │
│  │                                                                     │      │
│  │  4. Transactional Commit                                           │      │
│  │     - Atomic with produce (exactly-once)                          │      │
│  │     - sendOffsetsToTransaction()                                  │      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Exactly-Once Semantics

### Why Exactly-Once Matters

```
Problem without exactly-once:
├── Duplicate messages on producer retry
├── Duplicate processing on consumer restart
├── Inconsistent state across systems
└── Complex deduplication logic

What exactly-once provides:
├── Idempotent produces (no duplicates)
├── Transactional produces (atomic batches)
├── Read-process-write atomicity
└── Consistent state across failures
```

### Idempotent Producer

```java
// How idempotence works
public class IdempotentProducer {
    
    /**
     * Producer ID (PID): Unique identifier assigned by broker
     * Sequence Number: Monotonically increasing per partition
     * 
     * Broker deduplication:
     * - Tracks (PID, partition) -> last sequence number
     * - Rejects out-of-order or duplicate sequences
     */
    
    // Configuration
    Properties props = new Properties();
    props.put("enable.idempotence", true);  // Enables idempotence
    props.put("acks", "all");               // Required
    props.put("retries", Integer.MAX_VALUE);// Required
    props.put("max.in.flight.requests.per.connection", 5); // Max with idempotence
    
    // Simulation
    public void simulateIdempotence() {
        // Producer sends batch with sequence 0
        // Network timeout, producer retries
        // Broker receives both:
        //   - First: seq=0 -> Accept, store
        //   - Retry: seq=0 -> Duplicate, return success (no re-store)
        
        // Result: Exactly one copy in log
    }
}
```

### Transactional Producer

```java
// Transactional produce flow
public class TransactionalProducer {
    
    public void transactionalProduce() {
        Properties props = new Properties();
        props.put("transactional.id", "order-processor-1");
        props.put("enable.idempotence", true);
        
        Producer<String, String> producer = new KafkaProducer<>(props);
        producer.initTransactions();
        
        try {
            producer.beginTransaction();
            
            // Multiple produces in one transaction
            producer.send(new ProducerRecord<>("orders", "order-1", "data1"));
            producer.send(new ProducerRecord<>("payments", "pay-1", "data2"));
            producer.send(new ProducerRecord<>("inventory", "inv-1", "data3"));
            
            // Commit atomically
            producer.commitTransaction();
            
        } catch (Exception e) {
            producer.abortTransaction();
        }
    }
}

// Transaction Coordinator
public class TransactionCoordinator {
    
    /**
     * __transaction_state topic:
     * - Stores transaction metadata
     * - Partitioned by transactional.id
     * 
     * Transaction states:
     * - Empty -> Ongoing -> PrepareCommit -> CompleteCommit
     * - Empty -> Ongoing -> PrepareAbort -> CompleteAbort
     */
    
    // Two-phase commit
    public void commitTransaction(String transactionalId) {
        // Phase 1: Prepare
        // - Write PrepareCommit to __transaction_state
        // - Write commit markers to all partitions
        
        // Phase 2: Complete
        // - Write CompleteCommit to __transaction_state
        // - Transaction is now committed
    }
}
```

### Read-Process-Write Pattern

```java
// Exactly-once stream processing
public class ExactlyOnceProcessor {
    
    public void process() {
        Properties consumerProps = new Properties();
        consumerProps.put("isolation.level", "read_committed");
        consumerProps.put("enable.auto.commit", false);
        
        Properties producerProps = new Properties();
        producerProps.put("transactional.id", "processor-1");
        
        Consumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
        Producer<String, String> producer = new KafkaProducer<>(producerProps);
        
        producer.initTransactions();
        consumer.subscribe(Arrays.asList("input-topic"));
        
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            
            if (!records.isEmpty()) {
                producer.beginTransaction();
                
                try {
                    for (ConsumerRecord<String, String> record : records) {
                        // Process and produce output
                        String result = process(record.value());
                        producer.send(new ProducerRecord<>("output-topic", record.key(), result));
                    }
                    
                    // Commit offsets as part of transaction
                    producer.sendOffsetsToTransaction(
                        getOffsetsToCommit(records),
                        consumer.groupMetadata()
                    );
                    
                    producer.commitTransaction();
                    
                } catch (Exception e) {
                    producer.abortTransaction();
                }
            }
        }
    }
}
```

---

## 4. Log Compaction Deep Dive

### Why Log Compaction Exists

```
Problem with retention-based deletion:
├── Lose historical state
├── Cannot rebuild state from scratch
├── Changelog topics grow unbounded
└── Cannot use as source of truth

What compaction provides:
├── Keeps latest value per key
├── Bounded storage growth
├── Full state reconstruction
└── Changelog/CDC support
```

### How Compaction Works

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Log Compaction Process                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Log Structure:                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  [Clean Segments]              [Dirty Segments]    [Active]        │      │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────┐ │      │
│  │  │ Compacted    │  │ Dirty        │  │ Dirty        │  │ Active │ │      │
│  │  │ (no dups)    │  │ (has dups)   │  │ (has dups)   │  │        │ │      │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └────────┘ │      │
│  │                                                                     │      │
│  │  Cleaner Head ─────────────────────────────────────> Active Segment│      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Compaction Process:                                                          │
│  1. Log Cleaner thread scans dirty segments                                  │
│  2. Builds offset map: key -> latest offset                                  │
│  3. Copies records, skipping older versions of same key                      │
│  4. Replaces dirty segments with compacted segments                          │
│  5. Tombstones (null values) retained for delete.retention.ms               │
│                                                                               │
│  Configuration:                                                               │
│  ├── cleanup.policy=compact                                                  │
│  ├── min.cleanable.dirty.ratio=0.5 (50% dirty triggers compaction)         │
│  ├── delete.retention.ms=86400000 (tombstone retention)                     │
│  └── segment.ms=604800000 (force segment roll for compaction)              │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Compaction Simulation

```java
public class CompactionSimulation {
    
    public void simulateCompaction() {
        // Initial log state
        // Offset | Key | Value
        // 0      | A   | v1
        // 1      | B   | v1
        // 2      | A   | v2
        // 3      | C   | v1
        // 4      | B   | v2
        // 5      | A   | v3
        // 6      | B   | null (tombstone)
        // 7      | D   | v1
        
        // Compaction runs...
        
        // After compaction:
        // Offset | Key | Value
        // 3      | C   | v1    (only version)
        // 5      | A   | v3    (latest)
        // 6      | B   | null  (tombstone, retained temporarily)
        // 7      | D   | v1    (only version)
        
        // After delete.retention.ms:
        // Offset | Key | Value
        // 3      | C   | v1
        // 5      | A   | v3
        // 7      | D   | v1
        // (B's tombstone removed)
    }
}
```

---

## 5. Performance Optimization

### Zero-Copy Transfer

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Zero-Copy Transfer                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Traditional Copy (4 copies):                                                 │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  Disk ──> Kernel Buffer ──> User Buffer ──> Socket Buffer ──> NIC │      │
│  │       (1)              (2)             (3)               (4)       │      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Zero-Copy (sendfile):                                                        │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │                                                                     │      │
│  │  Disk ──> Kernel Buffer ──────────────────────────────────> NIC   │      │
│  │       (1)              (DMA, no CPU copy)                         │      │
│  │                                                                     │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                               │
│  Benefits:                                                                    │
│  ├── 2-4x throughput improvement                                             │
│  ├── Reduced CPU usage                                                       │
│  ├── Lower memory bandwidth                                                  │
│  └── Better cache utilization                                                │
│                                                                               │
│  Implementation:                                                              │
│  - Java: FileChannel.transferTo()                                            │
│  - Linux: sendfile() system call                                             │
│  - Requires: SSL disabled (or use kernel TLS)                               │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Page Cache Utilization

```java
// Kafka relies heavily on OS page cache
public class PageCacheOptimization {
    
    /**
     * Why page cache matters:
     * - Kafka writes sequentially to disk
     * - OS caches recent writes in memory
     * - Consumers reading recent data hit cache
     * - No application-level caching needed
     * 
     * Optimization:
     * - Reserve 25-50% of RAM for page cache
     * - Don't allocate too much heap to JVM
     * - Use XFS or ext4 filesystem
     * - Disable swap
     */
    
    // Memory allocation example (64GB server)
    // JVM Heap: 6GB
    // OS + Other: 8GB
    // Page Cache: 50GB (for hot data)
}
```

### Batching and Compression

```java
// Producer batching configuration
Properties props = new Properties();
props.put("batch.size", 65536);           // 64KB batch
props.put("linger.ms", 5);                // Wait up to 5ms for batch
props.put("compression.type", "lz4");     // LZ4 compression
props.put("buffer.memory", 67108864);     // 64MB buffer

// Compression comparison
// Type     | CPU Usage | Compression Ratio | Speed
// none     | lowest    | 1.0x              | fastest
// lz4      | low       | 2-3x              | very fast
// snappy   | low       | 2-3x              | very fast
// zstd     | medium    | 3-5x              | fast
// gzip     | high      | 3-5x              | slow

// Consumer fetching
props.put("fetch.min.bytes", 1048576);    // 1MB minimum
props.put("fetch.max.wait.ms", 500);      // Wait up to 500ms
props.put("max.partition.fetch.bytes", 1048576); // 1MB per partition
```

---

## 6. Failure Scenario Simulations

### Broker Failure During Produce

```
Scenario: Leader broker crashes while processing produce request

Timeline:
T0: Producer sends produce request (acks=all)
T1: Leader writes to local log
T2: Follower1 fetches and writes
T3: Leader crashes before responding
T4: Controller detects failure (heartbeat timeout)
T5: Controller elects Follower1 as new leader
T6: Producer times out, retries to new leader
T7: New leader checks sequence number
    - If duplicate: Returns success (idempotent)
    - If new: Appends to log

Result with idempotence: Exactly one copy
Result without idempotence: Possible duplicate
```

### Consumer Failure During Processing

```
Scenario: Consumer crashes after processing but before commit

Timeline:
T0: Consumer polls, receives messages [offset 100-109]
T1: Consumer processes messages
T2: Consumer crashes before commitSync()
T3: Coordinator detects failure (session timeout)
T4: Rebalance triggered
T5: Another consumer gets partition
T6: New consumer starts from last committed offset (99)
T7: Messages 100-109 reprocessed

Result: At-least-once delivery (duplicates possible)
Mitigation: Idempotent processing or exactly-once semantics
```

### Network Partition

```
Scenario: Network partition splits cluster

Partition: [Broker1, Broker2] | [Broker3, Controller]

For partition with Controller:
- Broker3 continues operating
- Topics with leader on Broker3 available
- Topics with leader on Broker1/2 unavailable

For partition without Controller:
- Broker1, Broker2 cannot elect new leaders
- Existing leaders continue serving reads
- Writes may fail (depending on ISR location)

Resolution:
1. Network heals
2. Brokers reconnect to controller
3. Leaders re-established
4. Followers truncate divergent logs to HW
5. Normal operation resumes

Data loss possible if:
- unclean.leader.election.enable=true
- Leader elected from non-ISR replica
```

