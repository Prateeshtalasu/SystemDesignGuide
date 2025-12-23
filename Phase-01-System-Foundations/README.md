# 📚 PHASE 1: SYSTEM FOUNDATIONS (Week 1)

**Goal**: Understand what "system" means and core metrics

## Topics:

1. **What is a distributed system?**

   - Definition from first principles
   - Difference from monolithic
   - Why distribution is hard

2. **Request-Response Lifecycle**

   - Browser to server (complete flow)
   - DNS, TCP handshake, HTTP
   - Round-trip time components

3. **Core Metrics**

   - Latency vs Throughput vs Bandwidth
   - Percentiles (p50, p95, p99)
   - How to measure in production

4. **Scalability**

   - Vertical vs Horizontal scaling
   - When to use each
   - Cost implications

5. **Availability & Reliability**

   - 99.9% vs 99.99% vs 99.999%
   - Downtime calculations
   - SLA vs SLO vs SLI

6. **CAP Theorem**

   - Consistency, Availability, Partition Tolerance
   - Real examples (not theoretical)
   - Why it matters in interviews

7. **Failure Modes**

   - Network failures
   - Partial failures
   - Timeouts & retries
   - Cascading failures
   - Circuit breakers (introduction)

8. **Load Testing & Performance**

   - Load vs Stress vs Spike testing
   - QPS (Queries Per Second) calculations
   - Bottleneck identification
   - Performance testing tools (JMeter, Gatling)

9. **Monitoring Basics**

   - What to monitor (latency, errors, traffic)
   - Health checks
   - Alerting basics
   - Metrics vs Logs vs Traces

10. **Chaos Engineering (Introduction)**
    - Why chaos engineering matters
    - Failure injection
    - Netflix Chaos Monkey concept

## Deliverable

Can explain why Netflix/Uber need distributed systems
