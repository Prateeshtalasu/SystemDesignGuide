# 🔗 PHASE 5: DISTRIBUTED SYSTEMS CORE (Week 5)

**Goal**: Understand distributed computing primitives

**Learning Objectives**:
- Understand time and ordering in distributed systems
- Master consensus algorithms and leader election
- Implement resilience patterns for fault tolerance
- Design systems that handle partial failures gracefully

**Estimated Time**: 15-20 hours

---

## Topics:

1. **Time & Clocks**
   - Physical clocks drift
   - Logical clocks (Lamport timestamps)
   - Vector clocks
   - Hybrid logical clocks (HLC)
   - TrueTime (Google Spanner)

2. **Leader Election**
   - Bully algorithm
   - Ring algorithm
   - Raft consensus (simplified)
   - ZooKeeper usage
   - Leader lease

3. **Distributed Consensus**
   - Why it's hard (FLP impossibility)
   - Paxos (high-level)
   - Raft (detailed walkthrough)
   - Multi-Paxos
   - When you need consensus

4. **Idempotency Deep Dive**
   - Why critical in distributed systems
   - Idempotent operations design
   - Idempotency keys (Stripe example)
   - Database-level idempotency
   - *Reference*: See Phase 1, Topic 13 for introduction

5. **Exactly-Once Semantics**
   - Why true exactly-once is impossible
   - At-most-once delivery
   - At-least-once delivery
   - Idempotent consumers
   - Kafka exactly-once

6. **Distributed Locks**
   - Redis-based (Redlock algorithm)
   - ZooKeeper-based
   - Database-based locks
   - Fencing tokens
   - Pitfalls and trade-offs

7. **Resilience Patterns**
   - Rate limiting (token bucket, leaky bucket, sliding window)
   - Circuit breakers (Hystrix, Resilience4j)
   - Bulkheads
   - Fallbacks & graceful degradation
   - Retry strategies (exponential backoff with jitter)
   - Timeout patterns

8. **Distributed Tracing Basics**
   - Why distributed tracing matters
   - Trace vs Span concepts
   - Correlation IDs
   - OpenTelemetry basics
   - Sampling strategies

9. **Service Mesh Concepts**
   - What is a service mesh
   - Sidecar pattern
   - Istio, Linkerd basics
   - When you need a service mesh
   - *Reference*: Deep dive in Phase 10

10. **Gossip Protocol**
    - How gossip works
    - Epidemic protocols
    - Use cases (Cassandra, DynamoDB, Consul)
    - Failure detection with gossip
    - Cracking the rumor problem

11. **Split Brain Problem**
    - What causes split brain
    - Detection mechanisms
    - Prevention strategies
    - Quorum-based solutions
    - STONITH (Shoot The Other Node In The Head)

12. **Quorum Systems**
    - Read/Write quorums
    - NRW notation
    - Sloppy quorum
    - Hinted handoff
    - Quorum in Cassandra and DynamoDB

13. **Failure Detection**
    - Heartbeat mechanisms
    - Phi accrual failure detector
    - Timeout-based detection
    - Gossip-based detection
    - False positives vs false negatives

---

## Cross-References:
- **Idempotency**: Introduction in Phase 1, Topic 13
- **Consistency Models**: See Phase 1, Topic 12
- **Service Mesh**: Deep dive in Phase 10
- **Rate Limiting**: Production patterns in Phase 11.5

---

## Practice Problems:
1. Implement a distributed lock with fencing tokens
2. Design a circuit breaker with half-open state
3. Implement leader election using ZooKeeper
4. Design a gossip protocol for cluster membership

---

## Common Interview Questions:
- "How does Raft handle leader failure?"
- "What's the difference between Paxos and Raft?"
- "How do you prevent split brain?"
- "When would you use a distributed lock vs optimistic locking?"

---

## Deliverable
Can explain Uber's distributed transaction handling
