# Distributed Message Queue (Kafka-like) - Problem Requirements

## 1. Problem Statement

Design a distributed message queue system similar to Apache Kafka that provides high-throughput, fault-tolerant, durable message streaming. The system must handle millions of messages per second, provide strong ordering guarantees, and support multiple consumers with different consumption patterns.

### What is a Distributed Message Queue?

A distributed message queue is infrastructure that enables asynchronous communication between services by storing and forwarding messages. Unlike traditional queues, distributed message queues are designed for massive scale, durability, and fault tolerance.

**Why it exists:**
- Decouple producers and consumers
- Handle traffic spikes without data loss
- Enable event-driven architectures
- Provide audit trail of all events
- Support real-time and batch processing

**What breaks without it:**
- Synchronous calls cause cascading failures
- Traffic spikes overwhelm downstream services
- No replay capability for debugging
- Tight coupling between services
- Lost messages during failures

---

## 2. Functional Requirements

### 2.1 Core Features (Must Have - P0)

#### Message Operations
| Feature | Description | Priority |
|---------|-------------|----------|
| Publish Message | Producers send messages to topics | P0 |
| Consume Message | Consumers read messages from topics | P0 |
| Acknowledge | Consumers confirm message processing | P0 |
| Seek | Consumers can seek to specific offset | P0 |

#### Topic Management
| Feature | Description | Priority |
|---------|-------------|----------|
| Create Topic | Create new topics with configuration | P0 |
| Delete Topic | Remove topics and their data | P0 |
| Partitioning | Distribute topic across partitions | P0 |
| Replication | Replicate partitions for durability | P0 |

#### Consumer Groups
| Feature | Description | Priority |
|---------|-------------|----------|
| Consumer Groups | Multiple consumers share workload | P0 |
| Offset Management | Track consumer progress | P0 |
| Rebalancing | Redistribute partitions on changes | P0 |

### 2.2 Important Features (Should Have - P1)

| Feature | Description | Priority |
|---------|-------------|----------|
| Message Retention | Time or size-based retention | P1 |
| Compaction | Keep only latest value per key | P1 |
| Transactions | Atomic multi-partition writes | P1 |
| Exactly-Once | Exactly-once processing semantics | P1 |
| Quotas | Rate limiting per client | P1 |
| ACLs | Access control for topics | P1 |

### 2.3 Nice-to-Have Features (P2)

| Feature | Description | Priority |
|---------|-------------|----------|
| Schema Registry | Schema validation and evolution | P2 |
| Kafka Connect | Connectors for data integration | P2 |
| Kafka Streams | Stream processing library | P2 |
| Tiered Storage | Archive old data to object storage | P2 |

---

## 3. Non-Functional Requirements

### 3.1 Performance Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| Write Throughput | 10 million messages/sec | High-volume event streaming |
| Read Throughput | 30 million messages/sec | Multiple consumers |
| Write Latency (p99) | < 10ms | Real-time applications |
| Read Latency (p99) | < 5ms | Fast consumption |
| End-to-End Latency | < 50ms | Producer to consumer |

### 3.2 Scalability Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| Topics | 100,000 | Large multi-tenant deployment |
| Partitions per Topic | 10,000 | High parallelism |
| Total Partitions | 1,000,000 | Cluster-wide |
| Message Size | Up to 10 MB | Support large payloads |
| Retention | Up to 7 days (configurable) | Replay capability |

### 3.3 Availability and Reliability

| Metric | Target | Rationale |
|--------|--------|-----------|
| Availability | 99.99% | Critical infrastructure |
| Durability | 99.999999999% | No message loss |
| Replication Factor | 3 (configurable) | Survive 2 failures |
| RPO | 0 (no data loss) | Synchronous replication |
| RTO | < 30 seconds | Fast failover |

### 3.4 Consistency Guarantees

| Guarantee | Description |
|-----------|-------------|
| Ordering | Messages ordered within partition |
| Durability | Acknowledged messages never lost |
| At-Least-Once | Default delivery guarantee |
| Exactly-Once | With transactions enabled |

---

## 4. Clarifying Questions for Interviewers

### Message Semantics

**Q1: What delivery guarantees do we need?**
> A: At-least-once by default, exactly-once with transactions.

**Q2: What ordering guarantees?**
> A: Total order within partition, no ordering across partitions.

**Q3: How long should we retain messages?**
> A: Configurable per topic, default 7 days, up to unlimited.

### Scale

**Q4: What's the expected message throughput?**
> A: 10M messages/second write, 30M messages/second read.

**Q5: What's the average message size?**
> A: 1 KB average, up to 10 MB maximum.

**Q6: How many topics and partitions?**
> A: 100K topics, 1M total partitions across cluster.

### Consumers

**Q7: How do consumers track progress?**
> A: Offset-based, stored in internal topic or external system.

**Q8: What happens when a consumer fails?**
> A: Rebalance partitions to remaining consumers in group.

**Q9: Can multiple consumer groups read same topic?**
> A: Yes, each group maintains independent offsets.

### Operations

**Q10: How do we handle broker failures?**
> A: Automatic leader election, ISR-based replication.

---

## 5. Success Metrics

### Performance Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Write Throughput | > 10M msg/sec | Messages written per second |
| Read Throughput | > 30M msg/sec | Messages read per second |
| P99 Write Latency | < 10ms | Producer acknowledgment time |
| P99 Read Latency | < 5ms | Consumer fetch time |

### Reliability Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Message Loss Rate | 0% | Lost messages / Total messages |
| Availability | 99.99% | Uptime percentage |
| Replication Lag | < 100ms | Follower behind leader |
| Leader Election Time | < 10 seconds | Time to elect new leader |

### Operational Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Partition Balance | < 10% skew | Even distribution |
| Consumer Lag | < 10,000 | Messages behind |
| Under-Replicated Partitions | 0 | Partitions below RF |

---

## 6. User Stories

### Producer Stories

```
As a producer,
I want to publish messages to a topic,
So that consumers can process them asynchronously.

As a producer,
I want acknowledgment when my message is replicated,
So that I know it won't be lost.

As a producer,
I want to send messages with keys,
So that related messages go to the same partition.
```

### Consumer Stories

```
As a consumer,
I want to read messages from a topic,
So that I can process events.

As a consumer,
I want to commit my offset,
So that I don't reprocess messages after restart.

As a consumer,
I want to seek to a specific offset,
So that I can replay messages.
```

### Operations Stories

```
As an operator,
I want to add brokers to the cluster,
So that I can increase capacity.

As an operator,
I want to monitor consumer lag,
So that I can detect processing issues.

As an operator,
I want to configure retention per topic,
So that I can manage storage costs.
```

---

## 7. Out of Scope

### Explicitly Excluded

| Feature | Reason |
|---------|--------|
| Message Transformation | Stream processing (Kafka Streams) |
| Complex Routing | Use separate routing layer |
| Request-Reply | Not a message queue pattern |
| Priority Queues | Partition-based ordering instead |

### Future Considerations

| Feature | Timeline |
|---------|----------|
| Tiered Storage | Phase 2 |
| Multi-Region Replication | Phase 2 |
| Serverless Mode | Phase 3 |

---

## 8. System Context

### Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Distributed Message Queue                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Producers                    Brokers              Consumers    │
│  ┌──────────┐               ┌─────────┐          ┌──────────┐  │
│  │ Service A│──────────────>│         │─────────>│Consumer  │  │
│  └──────────┘               │ Broker 1│          │ Group A  │  │
│                             │         │          └──────────┘  │
│  ┌──────────┐               └────┬────┘                        │
│  │ Service B│────────┐          │                              │
│  └──────────┘        │     ┌────┴────┐          ┌──────────┐  │
│                      ├────>│         │─────────>│Consumer  │  │
│  ┌──────────┐        │     │ Broker 2│          │ Group B  │  │
│  │ Service C│────────┤     │         │          └──────────┘  │
│  └──────────┘        │     └────┬────┘                        │
│                      │          │                              │
│                      │     ┌────┴────┐          ┌──────────┐  │
│                      └────>│         │─────────>│Consumer  │  │
│                            │ Broker 3│          │ Group C  │  │
│                            │         │          └──────────┘  │
│                            └─────────┘                        │
│                                                                  │
│  Controller Cluster (Metadata Management)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │Controller│  │Controller│  │Controller│                      │
│  │    1     │  │    2     │  │    3     │                      │
│  └──────────┘  └──────────┘  └──────────┘                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Message Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                       Message Flow                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Producer sends message with key                             │
│     ┌──────────┐                                                │
│     │ Producer │ ─── Message(key="user123", value="...") ──>   │
│     └──────────┘                                                │
│                                                                  │
│  2. Partition determined by key hash                            │
│     partition = hash(key) % num_partitions                      │
│     partition = hash("user123") % 10 = 3                        │
│                                                                  │
│  3. Message written to partition leader                         │
│     ┌─────────────────────────────────────────────┐            │
│     │ Topic: orders                                │            │
│     │ ┌─────────┐ ┌─────────┐ ┌─────────┐        │            │
│     │ │Partition│ │Partition│ │Partition│ ...    │            │
│     │ │   0     │ │   1     │ │   2     │        │            │
│     │ └─────────┘ └─────────┘ └─────────┘        │            │
│     │                                             │            │
│     │ ┌─────────┐ <── Message written here       │            │
│     │ │Partition│                                 │            │
│     │ │   3     │ [msg1][msg2][msg3][NEW]        │            │
│     │ └─────────┘                                 │            │
│     └─────────────────────────────────────────────┘            │
│                                                                  │
│  4. Replicated to followers                                     │
│     Leader (Broker 1) ──> Follower (Broker 2)                  │
│                      ──> Follower (Broker 3)                   │
│                                                                  │
│  5. Acknowledged to producer (after ISR replication)           │
│                                                                  │
│  6. Consumer fetches from partition                             │
│     ┌──────────┐                                                │
│     │ Consumer │ <── Fetch(offset=100, max_bytes=1MB)          │
│     └──────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Assumptions and Constraints

### Assumptions

1. **Storage**: SSD storage for low latency
2. **Network**: 10 Gbps network between brokers
3. **Clients**: Well-behaved clients with proper error handling
4. **Partitioning**: Producers choose partition or use key-based
5. **Ordering**: Ordering only required within partition

### Constraints

1. **Message Size**: Maximum 10 MB per message
2. **Retention**: Minimum 1 hour, maximum unlimited
3. **Partitions**: Once created, cannot reduce partition count
4. **Replication**: Minimum RF=1, recommended RF=3
5. **Consistency**: Trade-off between latency and durability

---

## 10. Key Technical Challenges

### The Fundamental Challenges

```
1. Durability vs Latency
   - Synchronous replication = durable but slow
   - Asynchronous replication = fast but may lose data
   - Solution: ISR (In-Sync Replicas) with configurable acks

2. Ordering vs Parallelism
   - Single partition = ordered but limited throughput
   - Multiple partitions = parallel but no global order
   - Solution: Partition by key for related message ordering

3. Exactly-Once Delivery
   - Network failures cause retries
   - Retries cause duplicates
   - Solution: Idempotent producers + transactions

4. Consumer Coordination
   - Multiple consumers need to share work
   - Failures require rebalancing
   - Solution: Consumer groups with coordinator
```

### Technical Deep Dives Required

```
1. Log-Structured Storage
   - Append-only segments
   - Index for offset lookup
   - Compaction for key-value

2. Replication Protocol
   - Leader election
   - ISR management
   - Catch-up replication

3. Consumer Protocol
   - Group coordination
   - Partition assignment
   - Offset management

4. Controller Design
   - Metadata management
   - Broker coordination
   - Failure detection
```

