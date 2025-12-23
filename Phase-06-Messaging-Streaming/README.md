# 📬 PHASE 6: MESSAGING & STREAMING (Week 6)

**Goal**: Master async communication and event-driven systems

## Topics:

1. **Queue vs Pub/Sub**
   - Point-to-point messaging
   - Publish-subscribe pattern
   - When to use each

2. **Message Delivery**
   - Push vs Pull models
   - Ordering guarantees (total order vs partition order)
   - At-least-once vs at-most-once

3. **Dead Letter Queue (DLQ)**
   - Poison messages
   - Retry policies
   - When to use

4. **Consumer Groups**
   - Load balancing across consumers
   - Partition assignment

5. **Kafka Deep Dive**
   - Topics, Partitions, Offsets
   - Producers (acks, retries, idempotence)
   - Consumers (poll loop, commit strategies)
   - Replication (ISR, leader election)
   - Log compaction
   - Exactly-once semantics (transactions)

6. **Other Messaging Systems**
   - RabbitMQ (AMQP)
   - Redis Streams
   - AWS SQS/SNS
   - Google Pub/Sub

7. **Advanced Patterns**
   - Change Data Capture (CDC)
   - Transactional Outbox pattern
   - Saga pattern (orchestration vs choreography)
   - Event Sourcing
   - CQRS (Command Query Responsibility Segregation)

8. **Stream Processing**
   - Windowing (tumbling, sliding, session)
   - Stateful processing
   - Backpressure handling

9. **Schema Evolution**
   - Avro schemas
   - Protobuf
   - Forward/backward compatibility

10. **Message Deduplication**
    - Idempotency keys
    - Deduplication windows
    - Exactly-once processing patterns

11. **Kafka Advanced Topics**
    - Consumer lag monitoring
    - Rebalancing strategies
    - Partition assignment strategies (range, round-robin, sticky)
    - Message ordering guarantees (per-partition ordering)
    - Compression (snappy, gzip, lz4)
    - Batch processing in Kafka

12. **Message Queue Patterns**
    - Priority queues
    - Delayed messages
    - Message routing
    - Topic vs Queue semantics

## Deliverable
Can design LinkedIn's messaging infrastructure

