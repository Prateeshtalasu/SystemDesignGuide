# 📬 PHASE 6: MESSAGING & STREAMING (Week 6)

**Goal**: Master async communication and event-driven systems

**Learning Objectives**:
- Understand messaging patterns and delivery guarantees
- Master Kafka architecture and operations
- Design event-driven architectures
- Handle stream processing at scale

**Estimated Time**: 15-20 hours

---

## Topics:

1. **Queue vs Pub/Sub**
   - Point-to-point messaging
   - Publish-subscribe pattern
   - When to use each
   - Fan-out patterns

2. **Message Delivery**
   - Push vs Pull models
   - Ordering guarantees (total order vs partition order)
   - At-least-once vs at-most-once
   - Delivery acknowledgment patterns

3. **Dead Letter Queue (DLQ)**
   - Poison messages
   - Retry policies
   - When to use
   - DLQ processing strategies

4. **Consumer Groups**
   - Load balancing across consumers
   - Partition assignment
   - Consumer group rebalancing
   - Static membership

5. **Kafka Deep Dive**
   - Topics, Partitions, Offsets
   - Producers (acks, retries, idempotence)
   - Consumers (poll loop, commit strategies)
   - Replication (ISR, leader election)
   - Log compaction
   - Exactly-once semantics (transactions)
   - Kafka internals (segments, indexes)

6. **Other Messaging Systems**
   - RabbitMQ (AMQP, exchanges, queues)
   - Redis Streams (vs Redis Pub/Sub)
   - AWS SQS/SNS
   - Google Pub/Sub
   - Apache Pulsar
   - Comparison matrix

7. **Advanced Patterns**
   - Change Data Capture (CDC)
   - Transactional Outbox pattern
   - Saga pattern (orchestration vs choreography)
   - Event Sourcing
   - CQRS (Command Query Responsibility Segregation)
   - Inbox pattern

8. **Stream Processing**
   - Windowing (tumbling, sliding, session)
   - Stateful processing
   - Backpressure handling
   - Watermarks and late data
   - Stream-table duality

9. **Schema Evolution**
   - Avro schemas
   - Protobuf
   - JSON Schema
   - Forward/backward compatibility
   - Schema Registry

10. **Message Deduplication**
    - Idempotency keys
    - Deduplication windows
    - Exactly-once processing patterns
    - Bloom filters for deduplication

11. **Kafka Advanced Topics**
    - Consumer lag monitoring
    - Rebalancing strategies
    - Partition assignment strategies (range, round-robin, sticky)
    - Message ordering guarantees (per-partition ordering)
    - Compression (snappy, gzip, lz4, zstd)
    - Batch processing in Kafka
    - Kafka Connect

12. **Message Queue Patterns**
    - Priority queues
    - Delayed messages
    - Message routing
    - Topic vs Queue semantics
    - Request-reply pattern

13. **Stream Processing Frameworks**
    - Kafka Streams
    - Apache Flink
    - Apache Spark Streaming
    - Comparison and when to use each
    - Stateful vs stateless processing

14. **Event-Driven Microservices**
    - Event notification vs event-carried state transfer
    - Event choreography
    - Event orchestration
    - Eventual consistency handling
    - Compensating transactions

---

## Cross-References:
- **CDC**: See Phase 3 for database CDC
- **Saga Pattern**: Deep dive in Phase 10 (Microservices)
- **CQRS**: Deep dive in Phase 10 (Microservices)
- **Idempotency**: See Phase 1, Topic 13

---

## Practice Problems:
1. Design a notification system using Kafka
2. Implement the transactional outbox pattern
3. Design an event-driven order processing system
4. Handle exactly-once processing with Kafka transactions

---

## Common Interview Questions:
- "How does Kafka ensure message ordering?"
- "What happens when a Kafka consumer fails?"
- "When would you use event sourcing?"
- "How do you handle schema changes in a message queue?"

---

## Deliverable
Can design LinkedIn's messaging infrastructure
