# Analytics System - Interview Grilling (Tradeoffs, Q&A, Failure Scenarios)

## STEP 14: TRADEOFFS & INTERVIEW QUESTIONS (WITH ANSWERS)

### Why Kafka over RabbitMQ/SQS/PubSub?

**Kafka:**
- Pros: High throughput (millions of events/second), durable, replayable, partition-based parallelism
- Cons: More complex, requires more infrastructure
- **We chose Kafka** because we need high throughput (1M events/second) and replayability

**RabbitMQ:**
- Pros: Simpler, good for lower throughput
- Cons: Lower throughput, not designed for event streaming
- **Why not:** Can't handle our scale

**SQS:**
- Pros: Managed service, simple
- Cons: Lower throughput (300K messages/second), no replay
- **Why not:** Throughput limits, no replay capability

**PubSub:**
- Pros: Managed service, good throughput
- Cons: Vendor lock-in, less control
- **Why not:** Prefer open-source, more control

---

### Why Redis over Memcached/Local Cache?

**Redis:**
- Pros: Rich data structures, persistence, replication, shared across services
- Cons: Network latency, additional infrastructure
- **We chose Redis** because we need shared state (deduplication) and complex data structures (sorted sets for metrics)

**Memcached:**
- Pros: Simpler, faster for simple key-value
- Cons: No persistence, no complex data structures
- **Why not:** Need persistence and sorted sets

**Local Cache:**
- Pros: Zero network latency
- Cons: Not shared, memory per service
- **Why not:** Need shared state for deduplication

---

### Why Eventual Consistency over Strong Consistency?

**Eventual Consistency:**
- Pros: High availability, better performance, handles partitions
- Cons: Data may be slightly stale
- **We chose eventual consistency** because analytics can tolerate slight delays (seconds to minutes)

**Strong Consistency:**
- Pros: Always accurate data
- Cons: Lower availability, slower, fails during partitions
- **Why not:** Analytics doesn't need real-time accuracy, availability is more important

**When to Use Strong Consistency:**
- Event schema registration (prevent conflicts)
- User permissions (security requirement)
- Financial transactions (if applicable)

---

### Why Object Storage + Data Warehouse over Just Database?

**Object Storage + Data Warehouse:**
- Pros: Cost-effective for petabytes, optimized for analytics, columnar format
- Cons: More complex, two systems to manage
- **We chose this** because we have petabytes of data and need analytical queries

**Just Database:**
- Pros: Simpler, single system
- Cons: Expensive at petabyte scale, not optimized for analytics
- **Why not:** Cost and performance at scale

**When to Use Just Database:**
- Small scale (< 1 TB)
- Need ACID transactions
- Simple queries

---

### Why Stream Processing + Batch Processing?

**Both:**
- Pros: Real-time insights + comprehensive historical analysis
- Cons: More complex, two pipelines
- **We chose both** because we need both real-time dashboards and historical analysis

**Just Stream Processing:**
- Pros: Simpler, real-time
- Cons: Can't handle complex historical analysis
- **Why not:** Need historical analysis

**Just Batch Processing:**
- Pros: Simpler, comprehensive
- Cons: No real-time insights
- **Why not:** Need real-time dashboards

---

### What Breaks First Under Load?

**Answer: Kafka Throughput**

**Why:**
- Kafka is the bottleneck (1M events/second)
- If Kafka can't keep up, event ingestion fails
- Other components can scale more easily

**Mitigation:**
- Add more Kafka brokers
- Increase partitions
- Optimize Kafka configuration

**Second Bottleneck: Stream Processor**
- If stream processor can't keep up, real-time metrics become stale
- Mitigation: Add more Flink workers

---

### How Does This System Evolve Over Time?

**MVP Phase (What to Build First):**

1. **Event Ingestion:**
   - Simple API endpoint
   - Write to database (PostgreSQL)
   - Basic validation

2. **Query API:**
   - Simple SQL queries
   - Direct database queries
   - No caching

3. **Storage:**
   - Single database (PostgreSQL)
   - No partitioning
   - Basic indexes

**Production Phase (What to Add):**

1. **Kafka:**
   - Add Kafka for buffering
   - Decouple ingestion from processing

2. **Stream Processing:**
   - Add Flink for real-time aggregation
   - Update Redis with metrics

3. **Data Warehouse:**
   - Move to columnar storage (Snowflake)
   - Partition by time
   - Optimize for analytics

4. **Caching:**
   - Add Redis for deduplication
   - Cache query results
   - Cache real-time metrics

**Scale Phase (How It Changes at 10x, 100x):**

**At 10x (10M events/second):**
- Kafka: 10x partitions, more brokers
- Stream Processing: 10x Flink workers
- Storage: Aggressive tiering, compression
- Cost: ~$1.9M/month

**At 100x (100M events/second):**
- Kafka: Custom Kafka deployment, more regions
- Stream Processing: Custom Flink deployment
- Storage: Custom storage solutions, data sampling
- Cost: ~$19M/month (needs optimization)

**Migration Strategies:**

1. **MVP → Production:**
   - Add Kafka gradually (dual-write)
   - Migrate queries to data warehouse
   - Add caching incrementally

2. **Production → Scale:**
   - Horizontal scaling (add more instances)
   - Data tiering (move old data to cold storage)
   - Query optimization (pre-aggregation)

---

### What Would You Do with Unlimited Budget?

**Answer:**

1. **Multi-Region Active-Active:**
   - Zero downtime
   - Instant failover
   - Global low latency

2. **Custom Hardware:**
   - Optimized for analytics workloads
   - Faster processing

3. **Advanced Features:**
   - Real-time ML inference
   - Advanced anomaly detection
   - Predictive analytics

4. **Better Observability:**
   - Full distributed tracing
   - Advanced monitoring
   - AI-powered alerting

---

### What Would You Cut if You Had 2 Weeks to Build?

**Answer:**

1. **Cut: Real-Time Processing**
   - Keep batch processing only
   - Real-time can be added later
   - Saves: Stream processing infrastructure

2. **Cut: Data Warehouse**
   - Keep PostgreSQL only
   - Add data warehouse later
   - Saves: Data warehouse setup

3. **Cut: Advanced Caching**
   - Keep basic caching
   - Add Redis later
   - Saves: Redis infrastructure

4. **Keep:**
   - Event ingestion API
   - Basic storage (PostgreSQL)
   - Simple query API

**MVP in 2 Weeks:**
- Event ingestion → PostgreSQL
- Simple query API
- Basic dashboards

---

### Common Interviewer Pushbacks and Responses

**Pushback 1: "Why not use a managed analytics service like Google Analytics?"**

**Response:**
- Need custom event schemas
- Need to own the data
- Need to integrate with internal systems
- Need custom analytics beyond what GA provides

**Pushback 2: "Why not use a time-series database like InfluxDB?"**

**Response:**
- Need to support complex analytical queries (not just time-series)
- Need to support ad-hoc SQL queries
- Time-series DBs are optimized for metrics, not events
- Data warehouse is more flexible

**Pushback 3: "Why not process everything in real-time?"**

**Response:**
- Batch processing is more cost-effective for historical analysis
- Some queries need full dataset (not just recent data)
- Real-time is only needed for dashboards (< 5% of use cases)
- Batch processing handles complex aggregations better

**Pushback 4: "Why not use a single database for everything?"**

**Response:**
- Petabyte scale is too expensive for single database
- Need different optimizations (row vs columnar)
- Analytics queries are different from transactional queries
- Cost: Database would be 10x more expensive

---

### Level-Specific Answers

**L4 (Junior Engineer): Basic Trade-offs, Correctness, Simple Scaling**

**Focus:**
- Understand basic architecture
- Know why we use Kafka, Redis, data warehouse
- Understand scaling (add more instances)
- Know basic failure scenarios

**Example Question: "Why do we use Kafka?"**
- Answer: "Kafka buffers events so we can handle traffic spikes. It also allows multiple consumers to process events independently."

**L5 (Senior Engineer): Production Concerns, Deep Dives into Bottlenecks**

**Focus:**
- Understand production concerns (monitoring, alerting, security)
- Deep dive into bottlenecks (Kafka throughput, stream processor lag)
- Understand failure scenarios and recovery
- Know cost optimization strategies

**Example Question: "What happens if Kafka goes down?"**
- Answer: "Event ingestion fails, but client SDK queues events. Stream processor stops, but resumes from last committed offset. Real-time metrics become stale, but historical queries still work. We have automatic failover within 30 seconds."

**L6 (Staff/Principal Engineer): Multiple Architectures, Migrations, Org-Level Trade-offs**

**Focus:**
- Understand multiple architecture options
- Know migration strategies
- Understand org-level trade-offs (build vs buy, cost vs performance)
- Know how to evolve system over time

**Example Question: "How would you redesign this for 100x scale?"**
- Answer: "At 100x, we'd need custom Kafka deployment with more regions, custom Flink deployment, custom storage solutions with data sampling, and aggressive cost optimization. We'd also consider different architectures like event sourcing or CQRS."

---

## STEP 15: FAILURE SCENARIOS & RESILIENCE (DETAILED)

### Single Points of Failure Identification

**Critical Single Points of Failure:**

1. **Kafka Cluster:**
   - Impact: Event ingestion fails
   - Mitigation: 3x replication, automatic failover
   - Recovery: Automatic (within 30 seconds)

2. **PostgreSQL Primary:**
   - Impact: Metadata writes fail
   - Mitigation: Automatic failover to replica
   - Recovery: Automatic (within 30 seconds)

3. **Data Warehouse:**
   - Impact: Historical queries fail
   - Mitigation: Managed service with high availability
   - Recovery: Automatic (managed service)

4. **Redis Cluster:**
   - Impact: Real-time queries fail, deduplication fails
   - Mitigation: 3x replication, automatic failover
   - Recovery: Automatic (within 10 seconds)

---

### What Happens if Each Major Component Fails?

**Kafka Cluster Failure:**

**Symptoms:**
- Event ingestion returns 503 errors
- Stream processor stops processing
- Real-time metrics stop updating

**What Degrades:**
- Event ingestion (fails)
- Real-time metrics (stale)
- Stream processing (stops)

**What Stays Up:**
- Historical queries (data warehouse unaffected)
- Cached query results (still work)
- Event metadata (PostgreSQL unaffected)

**Recovery:**
- Automatic failover (within 30 seconds)
- Stream processor resumes from last committed offset
- Events queued in client SDK are retried

**Data Loss:**
- Events in Kafka: None (replicated)
- Events queued in client: None (retried)
- Real-time metrics: Slight staleness (acceptable)

---

**PostgreSQL Primary Failure:**

**Symptoms:**
- Metadata writes fail
- Event ingestion may slow down (if checking metadata)

**What Degrades:**
- Metadata writes (fail)
- Event ingestion (may slow, but still works)

**What Stays Up:**
- Event ingestion to Kafka (still works)
- Queries (read from replicas)
- Stream processing (unaffected)

**Recovery:**
- Automatic failover to replica (within 30 seconds)
- Metadata writes resume
- No data loss (async replication)

---

**Data Warehouse Failure:**

**Symptoms:**
- Historical queries fail
- Batch processing may fail

**What Degrades:**
- Historical queries (fail)
- Batch processing (may fail)

**What Stays Up:**
- Event ingestion (unaffected)
- Real-time queries (Redis unaffected)
- Stream processing (unaffected)

**Recovery:**
- Managed service handles recovery automatically
- Queries resume when service is back
- No data loss (managed service)

---

**Redis Cluster Failure:**

**Symptoms:**
- Real-time queries fail or are slow
- Event deduplication fails (may accept duplicates)

**What Degrades:**
- Real-time queries (fall back to data warehouse, slower)
- Event deduplication (may accept duplicates, but processing is idempotent)

**What Stays Up:**
- Event ingestion (still works)
- Historical queries (data warehouse unaffected)
- Stream processing (unaffected)

**Recovery:**
- Automatic failover (within 10 seconds)
- Real-time queries resume
- Deduplication resumes

**Data Loss:**
- None (Redis is cache, not source of truth)

---

### Recovery Strategy for Each Component

**Kafka Recovery:**

1. **Automatic Failover:**
   - Kafka detects broker failure
   - Reassigns partitions to healthy brokers
   - Consumers reconnect automatically

2. **Manual Recovery (if needed):**
   - Identify failed broker
   - Replace with new broker
   - Rebalance partitions

3. **Data Recovery:**
   - Events are replicated (3x)
   - No data loss
   - Consumers resume from last committed offset

**PostgreSQL Recovery:**

1. **Automatic Failover:**
   - Patroni or RDS detects primary failure
   - Promotes replica to primary
   - Updates DNS/connection strings

2. **Manual Recovery (if needed):**
   - Identify failed primary
   - Promote replica
   - Replace failed instance

3. **Data Recovery:**
   - Async replication ensures no data loss
   - Replica catches up automatically

**Redis Recovery:**

1. **Automatic Failover:**
   - Redis Sentinel detects master failure
   - Promotes replica to master
   - Clients reconnect automatically

2. **Manual Recovery (if needed):**
   - Identify failed master
   - Promote replica
   - Replace failed instance

3. **Data Recovery:**
   - Redis is cache (not source of truth)
   - Data repopulated from source (Kafka, data warehouse)

---

### Data Loss Prevention Mechanisms

**Event Ingestion:**
- Kafka replication (3x)
- Client SDK queuing (retries on failure)
- Idempotent processing (deduplication)

**Event Processing:**
- Kafka offsets (resume from last committed)
- Idempotent processing (check eventId)
- Batch processing with transactions

**Storage:**
- Object storage replication (3x, cross-region)
- Data warehouse backups (managed service)
- PostgreSQL replication (async)

**Query Results:**
- Cache is not source of truth (can be regenerated)
- Query results can be recomputed

---

### Partial System Failure Handling

**Scenario: One Region Fails**

**Impact:**
- Events in that region fail
- Queries in that region fail

**Handling:**
- DNS failover to secondary region (< 5 minutes)
- Events queued in client SDK (retried in new region)
- Queries routed to secondary region

**Data Loss:**
- Events in flight: May be lost (acceptable, < 1%)
- Events queued in client: Not lost (retried)

---

**Scenario: Stream Processor Lags**

**Impact:**
- Real-time metrics become stale
- Queries show outdated data

**Handling:**
- Monitor lag (alert if > 5 minutes)
- Scale up Flink workers
- Optimize processing logic

**Data Loss:**
- None (events are in Kafka, can be reprocessed)

---

### Cascading Failure Prevention

**Circuit Breakers:**
- Prevent cascading failures by failing fast
- Open circuit when downstream service fails
- Prevents overwhelming failed service

**Rate Limiting:**
- Prevent overload by limiting requests
- Protects downstream services
- Allows system to recover

**Timeout and Retry:**
- Timeout prevents hanging requests
- Retry with backoff prevents thundering herd
- Idempotent operations safe to retry

**Load Shedding:**
- Drop non-critical requests under load
- Prioritize critical operations
- Allows system to recover

---

### Circuit Breaker Configuration Details

**Ingestion Service → Kafka:**

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)  // Open after 50% failures
    .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before half-open
    .slidingWindowSize(100)  // Check last 100 requests
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .build();
```

**Query Service → Redis:**

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(10))
    .timeoutDuration(Duration.ofMillis(100))  // Fast timeout
    .build();
```

**Query Service → Data Warehouse:**

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(30)  // Lower threshold (queries are expensive)
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .timeoutDuration(Duration.ofSeconds(30))
    .build();
```

---

### Disaster Recovery Plan (RTO/RPO)

**RTO (Recovery Time Objective): 1 hour**

**Steps:**
1. Detection: Automated monitoring (< 1 minute)
2. DNS Failover: Route traffic to secondary region (< 5 minutes)
3. Data Sync: Catch up on missed events (< 30 minutes)
4. Verification: Verify system health (< 10 minutes)
5. Communication: Notify stakeholders (< 5 minutes)

**RPO (Recovery Point Objective): 5 minutes**

**What This Means:**
- Maximum data loss: 5 minutes of events
- Events queued in client SDK: Not lost (retried)
- Events in Kafka: Not lost (replicated)
- Real-time metrics: May be stale (acceptable)

**Validation Drills:**

1. **Quarterly DR Drill:**
   - Simulate region failure
   - Execute failover procedure
   - Verify RTO/RPO
   - Document lessons learned

2. **Monthly Component Failure Test:**
   - Simulate single component failure
   - Verify automatic recovery
   - Measure recovery time

3. **Weekly Backup Verification:**
   - Verify backups are working
   - Test restore procedure
   - Verify data integrity

---

## Summary

**Tradeoffs:**
- Kafka over alternatives (throughput, replayability)
- Redis over alternatives (shared state, data structures)
- Eventual consistency (availability over accuracy)
- Object storage + data warehouse (cost, performance)

**System Evolution:**
- MVP: Simple ingestion + database
- Production: Add Kafka, stream processing, data warehouse
- Scale: Custom solutions, optimization

**Failure Scenarios:**
- Each component has automatic recovery
- Graceful degradation (some features work, others don't)
- No data loss (replication, queuing)

**Interview Readiness:**
- Understand tradeoffs and can defend decisions
- Know failure scenarios and recovery
- Can explain system evolution
- Level-appropriate depth

