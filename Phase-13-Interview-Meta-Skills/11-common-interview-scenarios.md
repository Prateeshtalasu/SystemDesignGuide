# Common Interview Scenarios

## 0️⃣ Prerequisites

Before diving into common interview scenarios, you should understand:

- **Problem Approach Framework**: How to structure your interview (covered in Topic 1)
- **Trade-off Analysis**: How to evaluate and justify decisions (covered in Topic 9)
- **Scalability Patterns**: How to scale systems (covered in Topic 10)
- **System Design Fundamentals**: Core components and patterns (covered in Phases 1-9)

Quick refresher: System design interviews follow predictable patterns. Interviewers ask similar types of questions to probe your understanding of scalability, reliability, and trade-offs. Knowing these common scenarios and preparing thoughtful responses gives you a significant advantage.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Interviewers have limited time (45-60 minutes) to evaluate your system design skills. They use standard scenarios and follow-up questions to efficiently probe different aspects of your knowledge:

- **Scaling scenarios**: Test your understanding of horizontal scaling, sharding, caching
- **Failure scenarios**: Test your understanding of reliability, redundancy, recovery
- **Consistency scenarios**: Test your understanding of distributed systems trade-offs
- **Performance scenarios**: Test your understanding of optimization and bottlenecks
- **Evolution scenarios**: Test your ability to adapt designs to changing requirements

Without preparation for these scenarios, candidates often:
1. **Freeze up**: Don't know how to approach the question
2. **Give shallow answers**: "We'd add more servers"
3. **Miss key considerations**: Forget about data consistency, failure recovery
4. **Ramble without structure**: Don't provide a clear, organized response

---

## 2️⃣ The Six Most Common Scenarios

### Scenario 1: "Design for 1M users, then scale to 1B"

**What the interviewer is testing**:
- Understanding of scaling stages
- Ability to identify bottlenecks
- Knowledge of scaling patterns
- Trade-off awareness

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SCALING STAGES FRAMEWORK                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  STAGE 1: 1M USERS                                                  │
│  ─────────────────                                                  │
│  - Single region deployment                                         │
│  - Load balancer + multiple app servers                             │
│  - Primary DB + read replicas                                       │
│  - Redis cache                                                      │
│  - CDN for static content                                           │
│                                                                      │
│  STAGE 2: 10M USERS                                                 │
│  ──────────────────                                                 │
│  - Database sharding (by user_id)                                   │
│  - Distributed cache (Redis cluster)                                │
│  - Message queues for async processing                              │
│  - Microservices for independent scaling                            │
│  - Multi-AZ deployment                                              │
│                                                                      │
│  STAGE 3: 100M USERS                                                │
│  ───────────────────                                                │
│  - Multi-region deployment                                          │
│  - Global load balancing                                            │
│  - Data replication across regions                                  │
│  - Edge computing for latency                                       │
│                                                                      │
│  STAGE 4: 1B USERS                                                  │
│  ──────────────────                                                 │
│  - Extreme sharding (thousands of shards)                           │
│  - Custom infrastructure                                            │
│  - Data sovereignty compliance                                      │
│  - Dedicated teams per component                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example response**:

"Let me walk through the scaling stages:

**At 1M users**, the architecture is relatively straightforward:
- Load balancer distributing to 5-10 application servers
- PostgreSQL with 2-3 read replicas
- Redis for caching hot data
- CDN for static assets

The main bottleneck at this stage is typically the database. With read replicas and caching, we can handle the read load. Writes go to the primary.

**At 10M users**, we hit database limits:
- Shard the database by user_id. Each shard handles ~3M users.
- Move to distributed Redis cluster for cache scaling
- Introduce Kafka for async processing (notifications, analytics)
- Consider splitting into microservices if different components have different scaling needs

**At 100M users**, we need geographic distribution:
- Deploy to multiple regions (US, EU, Asia)
- Use global load balancing (Route 53) to route users to nearest region
- Replicate data across regions (async for performance, accept eventual consistency)
- Edge caching for frequently accessed data

**At 1B users**, we're in Facebook/Google territory:
- Thousands of database shards
- Custom-built infrastructure
- Dedicated teams for each component
- Complex data sovereignty and compliance requirements

The key insight is that each 10x growth requires rethinking the architecture. What works at 1M won't work at 100M."

---

### Scenario 2: "How would you handle a 10x traffic spike?"

**What the interviewer is testing**:
- Understanding of traffic patterns
- Knowledge of scaling strategies
- Awareness of failure modes under load
- Ability to design for resilience

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TRAFFIC SPIKE HANDLING                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PREVENTION (Before the spike)                                      │
│  ─────────────────────────────                                      │
│  - Auto-scaling configured with appropriate thresholds              │
│  - Load testing to understand system limits                         │
│  - Capacity planning with headroom                                  │
│  - Graceful degradation strategies defined                          │
│                                                                      │
│  ABSORPTION (During the spike)                                      │
│  ─────────────────────────────                                      │
│  - Auto-scaling kicks in (but has lag)                              │
│  - Rate limiting protects critical services                         │
│  - Queue-based processing absorbs burst                             │
│  - Caching reduces backend load                                     │
│                                                                      │
│  DEGRADATION (If overwhelmed)                                       │
│  ─────────────────────────────                                      │
│  - Shed non-critical features                                       │
│  - Return cached/stale data                                         │
│  - Queue requests for later processing                              │
│  - Show "system busy" rather than error                             │
│                                                                      │
│  RECOVERY (After the spike)                                         │
│  ─────────────────────────────                                      │
│  - Process queued requests                                          │
│  - Scale down gradually (not immediately)                           │
│  - Post-mortem analysis                                             │
│  - Update capacity planning                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example response**:

"Handling a 10x traffic spike requires preparation, absorption, and graceful degradation.

**Preparation**:
- Auto-scaling is configured to add instances when CPU exceeds 70%
- We maintain 2x headroom for normal traffic
- Non-critical features have feature flags to disable under load
- We've load-tested to 5x normal traffic

**Absorption**:
When the spike hits:
1. **Auto-scaling** adds more application servers (but takes 2-3 minutes)
2. **Rate limiting** kicks in at the API gateway, protecting backend services
3. **Message queues** absorb write spikes, processing at sustainable rate
4. **Caching** handles repeated reads without hitting the database

**Graceful Degradation**:
If we're still overwhelmed:
1. Disable non-critical features (recommendations, analytics tracking)
2. Return cached data even if slightly stale
3. Queue non-urgent writes (likes, views) for later processing
4. Show users a 'high traffic' message rather than errors

**Example for an e-commerce site during a flash sale**:
- Product browsing: Serve from cache, even if inventory is slightly stale
- Add to cart: Allow, queue inventory check
- Checkout: This is critical, maintain full functionality
- Order history: Degrade to 'check back later'

The key insight is that not all features are equally important. Protecting the critical path (checkout) while degrading non-critical features keeps the business running."

---

### Scenario 3: "What if this component fails?"

**What the interviewer is testing**:
- Understanding of failure modes
- Knowledge of redundancy patterns
- Ability to design for resilience
- Recovery strategies

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FAILURE HANDLING FRAMEWORK                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  For each component, consider:                                      │
│                                                                      │
│  1. DETECTION                                                       │
│     - How do we know it failed?                                     │
│     - Health checks, monitoring, alerts                             │
│                                                                      │
│  2. IMPACT                                                          │
│     - What breaks when this fails?                                  │
│     - Blast radius analysis                                         │
│                                                                      │
│  3. MITIGATION                                                      │
│     - How do we reduce impact?                                      │
│     - Redundancy, failover, circuit breakers                        │
│                                                                      │
│  4. RECOVERY                                                        │
│     - How do we restore service?                                    │
│     - Automatic vs manual recovery                                  │
│                                                                      │
│  5. PREVENTION                                                      │
│     - How do we prevent this failure?                               │
│     - Redundancy, testing, chaos engineering                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example: "What if the database fails?"**

"Let me walk through our database failure handling:

**Detection**:
- Health checks every 10 seconds from application servers
- Monitoring on replication lag, query latency, connection count
- Alerts trigger when health checks fail or metrics exceed thresholds

**Impact**:
- If primary fails: Writes are blocked, reads can continue from replicas
- If all replicas fail: Complete outage for that data
- Blast radius: All services depending on this database

**Mitigation**:
- **Redundancy**: Primary + 2 read replicas in different AZs
- **Automatic failover**: If primary fails, promote replica to primary (RDS Multi-AZ does this automatically)
- **Connection pooling**: Limits connection exhaustion
- **Circuit breaker**: If DB is slow, fail fast rather than queue requests

**Recovery**:
- Automatic: Replica promotion takes 1-2 minutes
- Manual: If automatic fails, on-call engineer promotes replica
- Data recovery: Point-in-time recovery from backups if needed

**Prevention**:
- Regular failover testing (monthly)
- Chaos engineering (randomly kill replicas in staging)
- Capacity monitoring (alert before we hit limits)

**Graceful degradation during failure**:
- Serve cached data where possible
- Queue writes for later processing
- Show users 'temporarily unavailable' for affected features
- Redirect to read replicas for read-only operations"

---

### Scenario 4: "How do you ensure data consistency?"

**What the interviewer is testing**:
- Understanding of consistency models
- Knowledge of distributed systems challenges
- Ability to choose appropriate consistency level
- Trade-off awareness (CAP theorem)

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY STRATEGIES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  STRONG CONSISTENCY                                                 │
│  ──────────────────                                                 │
│  - All reads see the latest write                                   │
│  - Techniques: Synchronous replication, distributed transactions    │
│  - Trade-off: Higher latency, lower availability                    │
│  - Use for: Payments, inventory, authentication                     │
│                                                                      │
│  EVENTUAL CONSISTENCY                                               │
│  ────────────────────                                               │
│  - Reads may see stale data temporarily                             │
│  - Techniques: Async replication, conflict resolution               │
│  - Trade-off: Potential stale reads, need conflict handling         │
│  - Use for: Social feeds, analytics, caching                        │
│                                                                      │
│  CAUSAL CONSISTENCY                                                 │
│  ──────────────────                                                 │
│  - Related operations are seen in order                             │
│  - Techniques: Vector clocks, causal ordering                       │
│  - Trade-off: More complex than eventual, less strict than strong   │
│  - Use for: Messaging, collaborative editing                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example response**:

"Consistency requirements vary by feature. Let me explain our approach:

**For payments (strong consistency required)**:
- Use synchronous replication to ensure payment is recorded before confirming
- Use database transactions to ensure atomicity
- Accept higher latency (extra 50-100ms) for correctness
- If we can't confirm consistency, reject the transaction rather than risk double-charge

**For user profiles (read-your-writes consistency)**:
- When user updates profile, write to primary
- Read from primary for that user's session (sticky session or cache)
- Other users can read from replicas (eventual consistency is fine)
- Ensures user sees their own changes immediately

**For social feed (eventual consistency acceptable)**:
- Async replication to read replicas
- Cache aggressively (even stale data is okay)
- Accept that new posts might take seconds to appear in followers' feeds
- Trade-off: Better performance and availability

**For distributed operations (saga pattern)**:
- When order spans multiple services (inventory, payment, shipping)
- Use saga pattern with compensating transactions
- If payment fails after inventory reserved, release inventory
- Accept eventual consistency across services

**Conflict resolution**:
- For concurrent updates, use last-write-wins with timestamps
- For critical data, use optimistic locking (version numbers)
- For collaborative editing, use CRDTs or operational transformation

The key insight is that different parts of the system can have different consistency requirements. We don't need strong consistency everywhere, and accepting eventual consistency where appropriate improves performance and availability."

---

### Scenario 5: "What are the bottlenecks?"

**What the interviewer is testing**:
- Ability to identify performance constraints
- Understanding of system behavior under load
- Knowledge of profiling and optimization
- Prioritization skills

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BOTTLENECK ANALYSIS FRAMEWORK                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  IDENTIFY (Where is the bottleneck?)                                │
│  ───────────────────────────────────                                │
│  - Measure latency at each component                                │
│  - Identify the slowest component                                   │
│  - Check resource utilization (CPU, memory, I/O, network)           │
│                                                                      │
│  COMMON BOTTLENECKS:                                                │
│  ┌─────────────┬────────────────────────────────────────┐          │
│  │ Component   │ Typical Bottleneck                     │          │
│  ├─────────────┼────────────────────────────────────────┤          │
│  │ Database    │ Query performance, connection limits   │          │
│  │ Network     │ Bandwidth, latency                     │          │
│  │ Application │ CPU, memory, thread pool               │          │
│  │ External API│ Rate limits, latency                   │          │
│  │ Storage     │ IOPS, throughput                       │          │
│  └─────────────┴────────────────────────────────────────┘          │
│                                                                      │
│  RESOLVE (How to fix?)                                              │
│  ─────────────────────                                              │
│  - Optimize the bottleneck component                                │
│  - Add caching to reduce load                                       │
│  - Scale horizontally or vertically                                 │
│  - Redesign to avoid the bottleneck                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example response**:

"Let me analyze the potential bottlenecks in this system:

**Current architecture analysis**:

1. **Database (likely primary bottleneck)**
   - Single PostgreSQL handling all reads and writes
   - At 10,000 QPS, database will be overwhelmed
   - Solution: Add read replicas, implement caching, consider sharding

2. **Application servers (secondary bottleneck)**
   - CPU-bound operations (JSON serialization, business logic)
   - Memory pressure from large objects
   - Solution: Horizontal scaling, optimize hot paths

3. **Network (potential bottleneck at scale)**
   - Cross-AZ latency adds 1-2ms per hop
   - Bandwidth limits for large payloads
   - Solution: CDN for static content, compress responses

4. **External API calls (hidden bottleneck)**
   - Third-party payment processor has 500ms latency
   - Rate limited to 100 requests/second
   - Solution: Async processing, caching, batching

**Prioritization**:

Based on our current load, I'd address bottlenecks in this order:

1. **Database optimization** (highest impact)
   - Add indexes for slow queries
   - Implement Redis caching for hot data
   - Add read replicas

2. **Async processing** (medium impact)
   - Move non-critical operations to background queues
   - Reduces response time and handles spikes

3. **Application optimization** (lower impact)
   - Profile and optimize hot code paths
   - Increase connection pool sizes

**How I'd measure**:
- APM tools (DataDog, New Relic) for latency breakdown
- Database slow query logs
- Resource monitoring (CPU, memory, I/O)
- Load testing to find breaking points"

---

### Scenario 6: "How would you monitor this?"

**What the interviewer is testing**:
- Understanding of observability
- Knowledge of monitoring tools and metrics
- Ability to design for operability
- Incident response awareness

**How to approach**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MONITORING FRAMEWORK                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  THE THREE PILLARS OF OBSERVABILITY                                 │
│  ──────────────────────────────────                                 │
│                                                                      │
│  1. METRICS (What is happening?)                                    │
│     - Request rate, error rate, latency (RED)                       │
│     - CPU, memory, disk, network (USE)                              │
│     - Business metrics (orders, signups)                            │
│                                                                      │
│  2. LOGS (Why did it happen?)                                       │
│     - Application logs (errors, warnings)                           │
│     - Access logs (who accessed what)                               │
│     - Audit logs (who changed what)                                 │
│                                                                      │
│  3. TRACES (Where did it happen?)                                   │
│     - Distributed tracing across services                           │
│     - Request flow visualization                                    │
│     - Latency breakdown by component                                │
│                                                                      │
│  KEY METRICS TO MONITOR                                             │
│  ──────────────────────                                             │
│  - Latency: p50, p95, p99                                          │
│  - Error rate: 4xx, 5xx                                            │
│  - Throughput: requests/second                                      │
│  - Saturation: queue depth, CPU usage                              │
│                                                                      │
│  ALERTING STRATEGY                                                  │
│  ─────────────────                                                  │
│  - Alert on symptoms, not causes                                    │
│  - Page for user-impacting issues only                             │
│  - Use multiple severity levels                                     │
│  - Avoid alert fatigue                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example response**:

"I'd implement monitoring across three pillars: metrics, logs, and traces.

**Metrics (using Prometheus + Grafana)**:

*Application metrics*:
- Request rate by endpoint
- Error rate (4xx, 5xx) by endpoint
- Latency percentiles (p50, p95, p99)
- Active connections

*Infrastructure metrics*:
- CPU, memory, disk utilization
- Network I/O
- Container/pod health

*Business metrics*:
- Orders per minute
- Payment success rate
- User signups

*Database metrics*:
- Query latency
- Connection pool utilization
- Replication lag

**Logs (using ELK stack)**:
- Structured JSON logs with correlation IDs
- Log levels: ERROR, WARN, INFO, DEBUG
- Centralized aggregation for searching
- Retention: 30 days hot, 1 year cold storage

**Traces (using Jaeger)**:
- Distributed tracing across all services
- Trace sampling at 1% for normal traffic, 100% for errors
- Latency breakdown by service

**Alerting strategy**:

| Severity | Condition | Action |
|----------|-----------|--------|
| P1 (Page) | Error rate > 5% for 5 min | Wake up on-call |
| P1 (Page) | p99 latency > 5s for 5 min | Wake up on-call |
| P2 (Urgent) | Error rate > 1% for 15 min | Slack notification |
| P3 (Warning) | CPU > 80% for 30 min | Ticket for next day |

**Dashboards**:
- Overview dashboard: Key metrics at a glance
- Service-specific dashboards: Deep dive per service
- On-call dashboard: Current alerts and status

The key insight is to alert on symptoms (high error rate) not causes (high CPU). High CPU might be fine if users aren't impacted."

---

## 3️⃣ Quick Response Templates

### Template for Scaling Questions

```
"To scale from X to Y, I'd address these areas:

1. DATABASE: [Current state] → [Scaled state]
   - Trade-off: [What we gain vs lose]

2. APPLICATION: [Current state] → [Scaled state]
   - Trade-off: [What we gain vs lose]

3. CACHING: [Current state] → [Scaled state]
   - Trade-off: [What we gain vs lose]

The first bottleneck will be [component] because [reason].
I'd address it by [solution]."
```

### Template for Failure Questions

```
"If [component] fails:

1. DETECTION: We detect via [method]
2. IMPACT: This affects [blast radius]
3. MITIGATION: We have [redundancy/failover]
4. RECOVERY: We recover by [automatic/manual process]
5. PREVENTION: We prevent via [testing/redundancy]

During failure, we gracefully degrade by [degradation strategy]."
```

### Template for Consistency Questions

```
"For [feature], I'd use [consistency level] because:

- The requirement is [requirement]
- The trade-off is [latency/availability impact]
- If consistency is violated, [consequence]
- We ensure consistency via [mechanism]

For [other feature], I'd use [different level] because [different requirements]."
```

---

## 4️⃣ One Clean Mental Summary

System design interviews use common scenarios to probe your understanding. The six most common are: scaling (1M to 1B users), traffic spikes (10x), component failures, data consistency, bottleneck identification, and monitoring.

For each scenario, have a framework ready. For scaling: identify bottlenecks at each stage. For failures: detection, impact, mitigation, recovery, prevention. For consistency: match the consistency level to the feature requirements.

Practice these scenarios until your responses are structured and confident. The interviewer isn't looking for a perfect answer, they're looking for structured thinking and awareness of trade-offs.

