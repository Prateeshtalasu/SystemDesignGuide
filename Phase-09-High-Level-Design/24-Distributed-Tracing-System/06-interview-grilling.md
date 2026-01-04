# Distributed Tracing System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a distributed tracing system design.

---

## Trade-off Questions

### Q1: Why use Kafka for span buffering instead of direct writes to storage?

**Answer:**

**Option A: Direct Writes to Elasticsearch**
```
Services → Collectors → Elasticsearch
```

**Option B: Kafka Buffering (Chosen)**
```
Services → Collectors → Kafka → Aggregators → Elasticsearch
```

**Why Kafka Wins:**

| Factor | Direct Writes | Kafka Buffering |
|--------|---------------|-----------------|
| Backpressure | Overwhelms Elasticsearch | Buffers spikes |
| Replay | Impossible | Can replay spans |
| Ordering | Hard to maintain | Per-partition ordering |
| Decoupling | Tight coupling | Loose coupling |
| Failure Recovery | Spans lost | Spans buffered |

**Key insight:** Kafka provides buffering, replay, and decoupling. Essential for high-throughput systems.

---

### Q2: Why eventual consistency for trace storage?

**Answer:**

**Strong Consistency:**
- All spans for a trace must be stored before trace is queryable
- Requires coordination across services
- Slower (wait for all spans)
- Higher latency

**Eventual Consistency (Chosen):**
- Spans stored independently
- Trace queryable when first span arrives (may be incomplete)
- Faster (no coordination)
- Acceptable for observational data

**Why Eventual Consistency:**
- Tracing is observational (doesn't affect user requests)
- Partial traces are better than no traces
- Out-of-order spans are common (network delays)
- System must be highly available

---

### Q3: How do you handle out-of-order spans?

**Answer:**

**Problem:** Spans arrive out of order due to network delays, service processing times

**Solution: Trace Aggregator with Timeout:**

```java
@Service
public class TraceAggregator {
    
    private final Map<String, TraceBuilder> activeTraces = new ConcurrentHashMap<>();
    private static final Duration TRACE_TIMEOUT = Duration.ofMinutes(5);
    
    public void addSpan(Span span) {
        String traceId = span.getTraceId();
        TraceBuilder builder = activeTraces.computeIfAbsent(traceId, 
            k -> new TraceBuilder(traceId));
        
        builder.addSpan(span);
        
        // Check if trace is complete
        if (builder.isComplete()) {
            completeTrace(traceId, builder.build());
        }
    }
    
    @Scheduled(fixedRate = 60000)
    public void flushStaleTraces() {
        long cutoff = System.currentTimeMillis() - TRACE_TIMEOUT.toMillis();
        activeTraces.entrySet().removeIf(entry -> {
            TraceBuilder builder = entry.getValue();
            if (builder.getLastUpdateTime() < cutoff) {
                // Trace is stale, mark as complete (may be incomplete)
                completeTrace(entry.getKey(), builder.build());
                return true;
            }
            return false;
        });
    }
}
```

**Approach:**
1. Buffer spans by trace_id in aggregator
2. Wait for trace completion (all spans) or timeout (5 minutes)
3. When complete or timeout, write to storage
4. Accept that some traces may be incomplete (better than waiting forever)

---

### Q4: Why head-based + tail-based sampling instead of just one?

**Answer:**

**Head-Based Sampling Only:**
- Decision at trace start (low overhead)
- May miss important traces (slow but no errors)
- Fixed rate (can't adapt)

**Tail-Based Sampling Only:**
- Decision after trace completes (captures all important traces)
- High memory (must buffer all spans)
- Higher latency (decision after completion)

**Hybrid Approach (Chosen):**

1. **Head-based (1% sampling)**: Low overhead, reduces volume
2. **Tail-based (errors, slow traces)**: Never miss important traces
3. **Result**: Best of both worlds

**Example:**
- Normal traces: 1% sampled (head-based)
- Error traces: 100% sampled (tail-based)
- Slow traces (p95+): 100% sampled (tail-based)
- Total: ~1.1% sampling rate (cost-efficient, captures all errors)

---

### Q5: How would you scale from 1M to 100M requests/second?

**Answer:**

**Current State (1M requests/second):**
- 50 collectors
- 20 aggregators
- 10 query services
- Elasticsearch cluster: 20 nodes

**Scaling to 100M requests/second:**

1. **Linear Scaling**
   ```
   Collectors: 50 → 5,000
   Aggregators: 20 → 2,000
   Query services: 10 → 1,000
   ```

2. **Storage Scaling**
   ```
   Elasticsearch: 20 nodes → 200 nodes
   S3 storage: 6 PB → 600 PB
   ```

3. **Kafka Scaling**
   ```
   Partitions: 100 → 10,000
   Brokers: 10 → 100
   ```

4. **Sampling Optimization**
   ```
   Sampling rate: 1% → 0.1% (for normal traffic)
   Error sampling: 100% (unchanged)
   ```

5. **Architecture Changes**
   - Regional deployment (collectors close to services)
   - Tiered storage (hot/cold/warm)
   - Better compression (reduce storage 2x)

**Bottlenecks:**
- Storage costs dominate (linear growth)
- Query performance (more traces to search)
- Network bandwidth (span ingestion)

---

## Failure Scenarios

### Scenario 1: Elasticsearch Cluster Failure

**Problem:** Entire Elasticsearch cluster fails

**Impact:**
- Traces cannot be stored
- Queries fail
- Spans buffered in Kafka (7-day retention)

**Recovery:**
1. Spans continue buffering in Kafka
2. Alert fires (cluster down)
3. Restore from backup or rebuild cluster
4. Replay spans from Kafka
5. System recovers (may lose some traces during downtime)

**Mitigation:**
- Multi-region deployment
- Automatic failover
- Backup and restore procedures

### Scenario 2: Sampling Service Overload

**Problem:** Sampling decisions become bottleneck

**Impact:**
- Span ingestion slows down
- Latency increases

**Recovery:**
1. Increase Redis capacity (more replicas)
2. Cache sampling decisions longer (reduce Redis load)
3. Move to client-side sampling (reduce server load)

**Mitigation:**
- Redis cluster (high availability)
- Sampling decision caching
- Client-side sampling option

### Scenario 3: Trace Aggregator Memory Exhaustion

**Problem:** Too many active traces in memory

**Impact:**
- Aggregators OOM crash
- Traces lost (not yet stored)

**Recovery:**
1. Aggregators auto-restart
2. Reduce trace timeout (flush traces faster)
3. Increase aggregator memory
4. Scale out aggregators (reduce traces per aggregator)

**Mitigation:**
- Monitor memory usage
- Auto-scaling aggregators
- Shorter trace timeout for old traces

---

## Level-Specific Expectations

### L4 (Entry Level) Answers

**Q: How does distributed tracing work?**
- Services create spans with trace IDs
- Spans collected and stored
- Traces reconstructed for viewing
- Used for debugging and performance analysis

**Q: What is sampling?**
- Not all traces stored (too expensive)
- Sample subset (e.g., 1%)
- Always sample errors

**Q: How do you query traces?**
- Query by trace ID (fast)
- Search by service, operation, tags (slower)
- Use Elasticsearch for indexing

### L5 (Mid Level) Answers

**Q: How do you handle out-of-order spans?**
- Aggregator buffers spans by trace_id
- Waits for completion or timeout
- Accepts incomplete traces (better than waiting)

**Q: How do you scale storage?**
- Time-based indices (daily rotation)
- Hot storage (recent, fast) + cold storage (old, cheap)
- Compression and deduplication

**Q: How do you optimize costs?**
- Sampling (reduce volume)
- Compression (reduce storage)
- Tiered storage (cheaper for old data)
- Cache tuning (reduce queries)

### L6 (Senior Level) Answers

**Q: How do you design for global scale?**
- Regional deployment (reduce latency)
- Multi-region replication (disaster recovery)
- Hierarchical aggregation (reduce cross-region traffic)
- Adaptive sampling (protect system under load)

**Q: How do you handle data privacy?**
- PII redaction (automatic)
- Access controls (RBAC)
- Encryption (at rest and in transit)
- Retention policies (GDPR compliance)

**Q: How do you integrate with other systems?**
- OpenTelemetry standard (vendor-agnostic)
- Multiple protocol support (Jaeger, Zipkin)
- Correlation IDs (trace ID in logs/metrics)
- API for integrations

---

## Summary

Key takeaways:
- Kafka buffering essential for high-throughput systems
- Eventual consistency acceptable for observational data
- Hybrid sampling (head + tail) balances cost and coverage
- Out-of-order spans handled with timeout-based aggregation
- Storage costs dominate at scale (optimize aggressively)

