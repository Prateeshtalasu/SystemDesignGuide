# 🔗 PHASE 5: DISTRIBUTED SYSTEMS CORE (Week 5)

**Goal**: Understand distributed computing primitives

## Topics:

1. **Time & Clocks**
   - Physical clocks drift
   - Logical clocks (Lamport timestamps)
   - Vector clocks
   - Hybrid logical clocks

2. **Leader Election**
   - Bully algorithm
   - Raft consensus (simplified)
   - ZooKeeper usage

3. **Distributed Consensus**
   - Why it's hard
   - Paxos (high-level)
   - Raft (detailed)
   - When you need consensus

4. **Idempotency**
   - Why critical in distributed systems
   - Idempotent operations
   - Idempotency keys (Stripe example)

5. **Exactly-Once Myths**
   - Why it's impossible
   - At-most-once, at-least-once
   - Idempotent consumers

6. **Distributed Locks**
   - Redis-based (Redlock)
   - ZooKeeper-based
   - Pitfalls and trade-offs

7. **Resilience Patterns**
   - Rate limiting (token bucket, leaky bucket)
   - Circuit breakers (Hystrix)
   - Bulkheads
   - Fallbacks & graceful degradation
   - Retry strategies (exponential backoff)

8. **Distributed Tracing Basics**
   - Why distributed tracing matters
   - Trace vs Span concepts
   - Correlation IDs
   - OpenTelemetry basics

9. **Service Mesh Concepts**
   - What is a service mesh
   - Sidecar pattern
   - Istio, Linkerd basics
   - When you need a service mesh

## Deliverable
Can explain Uber's distributed transaction handling

