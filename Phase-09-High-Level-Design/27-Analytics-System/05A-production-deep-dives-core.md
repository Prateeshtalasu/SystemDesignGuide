# Analytics System - Production Deep Dives: Core (Async, Caching, Search)

## STEP 6: ASYNC, MESSAGING & EVENT FLOW

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform that acts as a message broker. Think of it as a high-throughput, fault-tolerant "mailbox" where producers (event senders) write messages to topics (mailboxes), and consumers (event processors) read messages from those topics.

**What Problems It Solves:**

1. **Buffering**: Handles traffic spikes by buffering events when processing is slower than ingestion
2. **Decoupling**: Separates event producers (clients) from consumers (processors), allowing independent scaling
3. **Durability**: Events are persisted to disk, so they're not lost if a consumer crashes
4. **Ordering**: Maintains order of events within a partition
5. **Replayability**: Consumers can re-read events from any point in time

**How It Works Internally:**

- **Topics**: Logical categories of events (e.g., `events.project_123`)
- **Partitions**: Topics are split into partitions for parallelism (like shards)
- **Producers**: Write events to partitions
- **Consumers**: Read events from partitions
- **Consumer Groups**: Multiple consumers working together to process a topic
- **Offsets**: Track position of each consumer in each partition

**Real-World Analogy:**

Imagine a restaurant:
- **Producers** = Customers placing orders
- **Kafka** = Order tickets on a rotating wheel
- **Partitions** = Different sections of the wheel (appetizers, mains, desserts)
- **Consumers** = Kitchen staff processing orders
- **Consumer Groups** = Different kitchen teams (one for appetizers, one for mains)

---

### B) OUR USAGE: How We Use Kafka Here

**Why Async vs Sync for Event Ingestion:**

Events arrive at 1 million/second, but processing (validation, enrichment, storage) takes time. If we processed synchronously:
- Client would wait for full processing (slow response)
- System would be bottlenecked by slowest operation
- One slow event blocks others

With async (Kafka):
- Client gets immediate acknowledgment (202 Accepted)
- Events are buffered in Kafka
- Processors consume at their own pace
- System handles traffic spikes gracefully

**What Events Exist and Why:**

1. **Raw Events Topic** (`events.{project_id}`):
   - Contains all ingested events
   - Used for: Real-time processing, batch processing, archival
   - Why: Single source of truth for all events

2. **Processed Events Topic** (`events.processed.{project_id}`):
   - Contains validated and enriched events
   - Used for: Downstream consumers that need clean data
   - Why: Separation of concerns (validation vs processing)

3. **Metrics Topic** (`metrics.{metric_name}`):
   - Contains pre-aggregated metrics
   - Used for: Real-time dashboards
   - Why: Faster consumption for dashboards

**Topic/Queue Design:**

```
Topic: events.{project_id}
  Partitions: 100
  Replication: 3
  Retention: 7 days
  Compression: snappy

Topic: events.processed.{project_id}
  Partitions: 100
  Replication: 3
  Retention: 30 days
  Compression: snappy

Topic: metrics.{metric_name}
  Partitions: 10
  Replication: 3
  Retention: 24 hours
  Compression: gzip
```

**Partition Key Choice and Reasoning:**

**Partition Key: `eventId`**

**Why eventId?**
- Ensures same event goes to same partition
- Enables idempotent processing (deduplication)
- Maintains order per event (important for replay)

**Hot Partition Risk:**
- If we partition by `userId`, popular users create hot partitions
- If we partition by `eventType`, common events create hot partitions
- Using `eventId` (UUID) ensures even distribution

**Alternative Considered: Round-Robin**
- Pros: Perfect distribution
- Cons: No ordering guarantees, harder deduplication
- Decision: Use `eventId` for better semantics

**Consumer Group Design:**

1. **Stream Processing Consumer Group:**
   - Consumers: Flink workers
   - Processing: Real-time aggregation
   - Parallelism: 100 (one per partition)

2. **Batch Processing Consumer Group:**
   - Consumers: Spark workers
   - Processing: ETL to data warehouse
   - Parallelism: 10 (processes in batches)

3. **Archival Consumer Group:**
   - Consumers: Archive workers
   - Processing: Move to object storage
   - Parallelism: 20

**Ordering Guarantees:**

- **Global Ordering**: Not required (events are independent)
- **Per-Key Ordering**: Not strictly required, but `eventId` ensures same event processed once
- **Time-Ordering**: Not guaranteed (events may arrive out of order)

**Offset Management and Replay Strategy:**

- **Offset Storage**: Kafka stores offsets in `__consumer_offsets` topic
- **Commit Strategy**: Commit after successful processing (at-least-once semantics)
- **Replay**: Consumers can reset offsets to reprocess events
- **Failure Handling**: If consumer crashes, resume from last committed offset

**Deduplication Strategy:**

1. **At Ingestion:**
   - Check Redis for `eventId` (24-hour TTL)
   - If exists: Reject duplicate (409 Conflict)
   - If new: Accept, store in Redis

2. **At Processing:**
   - Stream processor checks `eventId` in state store
   - If processed: Skip
   - If new: Process and store `eventId`

**Outbox Pattern / CDC:**

Not needed here because:
- Events are already in Kafka (no DB transaction to sync)
- No need for strong consistency between DB and events
- Events are the source of truth

---

### C) REAL STEP-BY-STEP SIMULATION

**Event Ingestion Pipeline:**

```
1. User clicks "Add to Cart" button
   ↓
2. Client SDK sends event to Ingestion Service
   POST /v1/events
   {
     "eventId": "evt_abc123",
     "eventType": "add_to_cart",
     "userId": "user_123",
     "properties": {"productId": "prod_456", "amount": 29.99}
   }
   ↓
3. Ingestion Service validates:
   - Check API key ✓
   - Validate schema ✓
   - Check rate limits ✓
   - Check Redis for eventId (dedupe) ✓ (new event)
   ↓
4. Write to Kafka:
   - Topic: events.project_123
   - Partition: hash(eventId) % 100 = partition 42
   - Offset: 1,234,567
   ↓
5. Update metadata:
   - Insert into PostgreSQL (async)
   - Store eventId in Redis (24-hour TTL)
   ↓
6. Return 202 Accepted to client
   ↓
7. Background: Stream Processor (Flink) consumes from Kafka
   - Reads from partition 42, offset 1,234,567
   - Processes event: aggregate metrics
   - Updates Redis: increment "add_to_cart" metric
   - Commits offset: 1,234,568
   ↓
8. Background: Batch Processor (Spark) consumes from Kafka
   - Reads events in batches
   - Transforms and enriches
   - Writes to object storage (Parquet)
   - Loads into data warehouse
```

**What if Consumer Crashes Mid-Batch?**

1. **Stream Processor Crashes:**
   - Last committed offset: 1,234,567
   - Events 1,234,568 - 1,234,600 were processed but not committed
   - On restart: Resume from offset 1,234,567
   - **Result**: Events are reprocessed (at-least-once semantics)
   - **Mitigation**: Idempotent processing (check eventId in state store)

2. **Batch Processor Crashes:**
   - Last committed offset: 1,000,000
   - Batch of 10,000 events partially processed
   - On restart: Resume from offset 1,000,000
   - **Result**: Events are reprocessed
   - **Mitigation**: Idempotent writes to object storage (check if file exists)

**What if Duplicate Events Occur?**

1. **Duplicate eventId at Ingestion:**
   - First event: Accepted, stored in Redis
   - Second event (same eventId): Rejected with 409 Conflict
   - **Result**: No duplicate in Kafka

2. **Duplicate eventId in Kafka (rare):**
   - Stream processor checks state store for eventId
   - If exists: Skip processing
   - If new: Process and store eventId
   - **Result**: Idempotent processing

### Hot Partition Mitigation

**Problem:** If a single event type or user generates significantly more events than others, its Kafka partition can become a bottleneck, leading to increased lag for consumers and potential data loss.

**Example Scenario:**
```
Normal event: 100 events/second → Partition 42 (hash(eventId) % 100)
High-volume event: 10,000 events/second → Multiple partitions, but if eventId pattern is predictable, could cluster
Result: Certain partitions receive 100x traffic, become bottleneck for consumers
```

**Detection:**
- Monitor Kafka consumer group lag per partition (e.g., using Prometheus/Grafana)
- Alert when lag for a specific partition exceeds a threshold (e.g., 2x average lag)
- Track message throughput metrics per partition
- Monitor partition leader election frequency (indicates instability)

**Mitigation Strategies:**

1. **Partition Key Design:**
   - We partition by `eventId` (UUID) hash. This generally distributes events evenly.
   - For extremely high-volume event types, consider a composite key (e.g., `eventType:eventId`) if ordering within event types is acceptable.

2. **Dynamic Repartitioning (Advanced):**
   - For persistent hot partitions, Kafka allows increasing the number of partitions.
   - A rebalancing tool can then redistribute existing data or new data to spread the load. This is a complex operational task.

3. **Consumer Scaling:**
   - Ensure enough Kafka consumers are running to process all partitions efficiently.
   - While adding more consumers won't help a single hot partition directly (one consumer per partition), it ensures overall throughput.

4. **Rate Limiting (Ingestion Side):**
   - Implement rate limiting on the ingestion service per `eventType` or `userId` to prevent a single entity from overwhelming Kafka.
   - This would involve throttling event ingestion for a specific event type if it exceeds a defined threshold.

5. **Dedicated Topics/Clusters (Extreme Cases):**
   - For "super-hot" event types (e.g., high-frequency telemetry), consider moving them to a dedicated Kafka topic with more partitions or even a separate Kafka cluster.
   - This isolates the load and prevents it from impacting other event types.

**Implementation:**

```java
@Service
public class HotPartitionDetector {
    
    public void detectAndMitigate() {
        Map<Integer, Long> partitionLag = getPartitionLag();
        double avgLag = partitionLag.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);
        
        partitionLag.entrySet().stream()
            .filter(e -> e.getValue() > avgLag * 2)
            .forEach(e -> {
                int partition = e.getKey();
                long lag = e.getValue();
                
                // Alert ops team
                alertOps("Hot partition detected: " + partition + ", lag: " + lag);
                
                // Investigate root cause
                analyzePartition(partition);
            });
    }
}
```

**Monitoring:**
- Partition lag per partition (Kafka metrics)
- Partition throughput (events/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**How Does Retry Work?**

1. **Client Retry (5xx errors):**
   - Exponential backoff: 1s, 2s, 4s, 8s
   - Max retries: 3
   - Same eventId used (idempotent)

2. **Kafka Consumer Retry:**
   - Automatic: Kafka retries on network errors
   - Manual: Consumer can seek to earlier offset
   - Dead Letter Queue: Failed events after max retries

**What Breaks First and What Degrades Gracefully?**

**Breaks First:**
- Kafka cluster: Events cannot be ingested (system down)
- PostgreSQL: Metadata writes fail (events still ingested, metadata updated later)
- Redis: Deduplication fails (may accept duplicates, but processing is idempotent)

**Degrades Gracefully:**
- Stream processor lag: Real-time queries may be slightly stale
- Batch processor lag: Historical queries may be missing recent data
- Query service overload: Queries queued, non-critical queries return 503

---

## STEP 7: CACHING (REDIS) + CACHE FLOW SIMULATION

### A) CONCEPT: What is Redis and Caching?

Redis is an in-memory data structure store that acts as a cache. Think of it as a super-fast "notebook" where you write down frequently accessed information so you don't have to look it up in the slower "filing cabinet" (database) every time.

**What Caching Means:**

Caching is storing frequently accessed data in fast storage (memory) to avoid slow lookups in slower storage (disk/database).

**Caching Patterns:**

1. **Cache-Aside (Lazy Loading):**
   - Application checks cache first
   - If miss: Query database, store in cache, return result
   - If hit: Return from cache
   - **We use this pattern**

2. **Write-Through:**
   - Write to cache and database simultaneously
   - Ensures cache is always up-to-date
   - **We don't use this** (too slow for high throughput)

3. **Write-Back:**
   - Write to cache first, database later (async)
   - Fast writes, but risk of data loss
   - **We don't use this** (need durability)

**What Consistency Tradeoff We Accept:**

- **Eventual Consistency**: Cache may be slightly stale (seconds to minutes)
- **Acceptable**: Analytics queries don't need real-time accuracy
- **Benefit**: Fast queries, reduced database load

---

### B) OUR USAGE: How We Use Redis Here

**Exact Keys and Values:**

1. **Event Deduplication:**
   ```
   Key: event:{eventId}
   Value: "processed"
   TTL: 24 hours
   ```

2. **Real-Time Metric Aggregates:**
   ```
   Key: metric:{metric_name}:{granularity}:{timestamp}
   Value: JSON string
   Example: metric:daily_active_users:1m:2024-01-15T10:30:00Z
   Value: {"count": 1250, "unique_users": 850}
   TTL: 2x window size (e.g., 2 minutes for 1-minute window)
   ```

3. **Query Result Cache:**
   ```
   Key: query:{query_hash}
   Value: Serialized query result
   TTL: 1 hour
   ```

4. **Schema Cache:**
   ```
   Key: schema:{project_id}:{event_type}:{version}
   Value: JSON schema definition
   TTL: 24 hours
   ```

**Who Reads Redis, Who Writes Redis:**

**Reads:**
- Ingestion Service: Check eventId for deduplication
- Query Service: Get real-time metrics, cached query results
- Stream Processor: Update metric aggregates

**Writes:**
- Ingestion Service: Store eventId (deduplication)
- Stream Processor: Update metric aggregates
- Query Service: Cache query results

**TTL Behavior and Why:**

1. **Event Deduplication (24 hours):**
   - Why: Events are processed within seconds, but need to prevent duplicates for 24 hours
   - TTL: 24 hours (matches deduplication window)

2. **Metric Aggregates (2x window size):**
   - Why: Keep aggregates until next window completes
   - Example: 1-minute window → 2-minute TTL
   - Rationale: Allows queries to access previous window while current window is being computed

3. **Query Results (1 hour):**
   - Why: Balance between freshness and performance
   - TTL: 1 hour (acceptable staleness for analytics)

**Eviction Policy:**

**Policy: `allkeys-lru` (Least Recently Used)**

**Why:**
- Evicts least recently used keys when memory is full
- Ensures frequently accessed data stays in cache
- Prevents cache from growing unbounded

**Alternative Considered: `allkeys-lfu` (Least Frequently Used)**
- Pros: Keeps frequently accessed data
- Cons: May evict recently added important data
- Decision: Use LRU for better temporal locality

**Invalidation Strategy:**

1. **On Event Update:** Not applicable (events are immutable)

2. **On Metric Update:**
   - Stream processor overwrites metric aggregates
   - No explicit invalidation needed (TTL handles it)

3. **On Query Result Update:**
   - Query results are invalidated by TTL
   - Manual invalidation: Delete key if data changes

**Why Redis vs Local Cache vs DB Indexes:**

**Redis:**
- Pros: Shared across services, fast, supports complex data structures
- Cons: Network latency, additional infrastructure
- **We use this** for shared state and real-time aggregates

**Local Cache:**
- Pros: Zero network latency
- Cons: Not shared, memory per service
- **We don't use this** (need shared state)

**DB Indexes:**
- Pros: Strong consistency
- Cons: Slower than Redis
- **We use this** for persistent storage, not caching

---

### C) REAL STEP-BY-STEP SIMULATION

**Cache HIT Path (Real-Time Metric Query):**

```
1. Client requests metric:
   GET /v1/metrics/daily_active_users?startTime=2024-01-15T10:30:00Z&endTime=2024-01-15T10:35:00Z
   ↓
2. Query Service receives request
   ↓
3. Query Service checks Redis:
   Key: metric:daily_active_users:1m:2024-01-15T10:30:00Z
   Result: HIT (exists in Redis)
   ↓
4. Query Service aggregates from Redis:
   - Reads 5 time windows (10:30, 10:31, 10:32, 10:33, 10:34)
   - All HITs
   - Aggregates values
   ↓
5. Return response:
   {
     "metric": "daily_active_users",
     "data": [
       {"timestamp": "2024-01-15T10:30:00Z", "value": 1250},
       {"timestamp": "2024-01-15T10:31:00Z", "value": 1280},
       ...
     ]
   }
   ↓
6. Latency: < 10ms (all from Redis)
```

**Cache MISS Path (Query Result Cache):**

```
1. Client requests query:
   GET /v1/query?sql=SELECT COUNT(*) FROM events WHERE eventType='purchase'
   ↓
2. Query Service receives request
   ↓
3. Query Service checks Redis:
   Key: query:{hash_of_sql_query}
   Result: MISS (not in cache)
   ↓
4. Query Service executes query:
   - Routes to data warehouse
   - Executes SQL query
   - Takes 2 seconds
   ↓
5. Query Service stores result in Redis:
   Key: query:{hash_of_sql_query}
   Value: Serialized query result
   TTL: 1 hour
   ↓
6. Return response to client
   ↓
7. Next request (within 1 hour):
   - Cache HIT
   - Latency: < 10ms (from Redis)
```

**What if Redis is Down?**

1. **Event Deduplication:**
   - Ingestion Service: Cannot check for duplicates
   - **Fallback**: Accept events (processing is idempotent)
   - **Impact**: May accept duplicates, but processing handles it

2. **Real-Time Metrics:**
   - Query Service: Cannot get real-time metrics from Redis
   - **Fallback**: Query data warehouse (slower, but works)
   - **Impact**: Real-time queries become slower (seconds instead of milliseconds)

3. **Query Results:**
   - Query Service: Cannot cache results
   - **Fallback**: Always query data warehouse
   - **Impact**: All queries are slower, increased database load

**What Happens to DB Load and Latency?**

**When Redis is Up:**
- 80% of queries hit cache (Redis)
- 20% of queries hit database
- Average latency: 50ms (80% × 10ms + 20% × 200ms)

**When Redis is Down:**
- 0% of queries hit cache
- 100% of queries hit database
- Average latency: 200ms
- **DB load increases 5x** (from 20% to 100%)

**What is the Circuit Breaker Behavior?**

**Circuit Breaker Configuration:**
- Failure threshold: 50% failures in 1 minute
- Timeout: 1 second (for Redis operations)
- Half-open interval: 30 seconds

**Behavior:**
1. **Closed (Normal):**
   - All requests go to Redis
   - Monitor failures

2. **Open (Redis Down):**
   - Circuit opens after 50% failures
   - All requests bypass Redis, go directly to database
   - Fail fast (no waiting for Redis timeout)

3. **Half-Open (Testing):**
   - After 30 seconds, try one request to Redis
   - If success: Close circuit (Redis is back)
   - If failure: Open circuit again

**Why This TTL?**

1. **Event Deduplication (24 hours):**
   - Events are processed within seconds
   - But need to prevent duplicates for 24 hours (in case of replay)
   - 24 hours covers any reasonable replay window

2. **Metric Aggregates (2x window size):**
   - 1-minute window → 2-minute TTL
   - Allows queries to access previous window while current window is being computed
   - Ensures no gaps in time-series data

3. **Query Results (1 hour):**
   - Balance between freshness and performance
   - Analytics queries don't need real-time accuracy
   - 1 hour is acceptable staleness

**Why Cache-Aside vs Write-Through vs Write-Back?**

**Cache-Aside (We Use):**
- Pros: Simple, works well for read-heavy workloads
- Cons: Cache miss adds latency
- **Why**: Analytics is read-heavy, cache misses are acceptable

**Write-Through:**
- Pros: Cache always up-to-date
- Cons: Every write hits both cache and database (slow)
- **Why Not**: Too slow for high-throughput event ingestion

**Write-Back:**
- Pros: Fast writes
- Cons: Risk of data loss if cache crashes
- **Why Not**: Need durability for events

**Negative Caching (NOT_FOUND):**

**When to Use:**
- Cache "not found" results to avoid repeated database queries
- Example: User doesn't exist → Cache "NOT_FOUND" for 5 minutes

**Our Usage:**
- Not applicable for events (events either exist or don't)
- Use for schema lookups: Cache "schema not found" for 1 hour

---

## STEP 8: SEARCH (ELASTICSEARCH / OPENSEARCH) + INVERTED INDEX SIMULATION

### Is Search Required?

**Answer: Not Required for Core Analytics**

**Why:**
- Analytics queries are primarily aggregations (COUNT, SUM, AVG)
- Time-range queries are handled by partitioning
- User-specific queries use indexes on `userId`
- No full-text search needed for events

**What Would Change if Search Were Added Later:**

If we needed to search event properties (e.g., "find all events where product name contains 'laptop'"):

1. **Add Elasticsearch:**
   - Index event properties
   - Support full-text search
   - Support complex queries

2. **Indexing Pipeline:**
   - Stream processor indexes events to Elasticsearch
   - Maintains index in sync with events

3. **Query Service:**
   - Route search queries to Elasticsearch
   - Combine with aggregation queries

**For Now:**
- Use PostgreSQL indexes for simple property lookups
- Use data warehouse for complex analytical queries
- No need for full-text search

---

## Summary

**Async/Messaging:**
- Kafka for event buffering and distribution
- Stream processing for real-time aggregation
- Batch processing for ETL

**Caching:**
- Redis for event deduplication, real-time metrics, query results
- Cache-aside pattern for read-heavy workload
- TTL-based invalidation

**Search:**
- Not required for core analytics
- PostgreSQL indexes sufficient for property lookups
- Data warehouse for complex queries


