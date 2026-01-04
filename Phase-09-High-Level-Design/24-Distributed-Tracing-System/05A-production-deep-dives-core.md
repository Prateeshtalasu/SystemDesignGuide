# Distributed Tracing System - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for span buffering), caching strategy (trace cache), and sampling strategies (head-based, tail-based, adaptive).

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Tracing Context?

In a distributed tracing system, Kafka serves as a buffer between span collection and trace aggregation:

1. **Span Buffering**: Collects spans from multiple collectors before aggregation
2. **Backpressure Handling**: Handles traffic spikes without dropping spans
3. **Replay Capability**: Can replay spans for reprocessing
4. **Ordering**: Per-trace ordering (same trace_id goes to same partition)

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key | Retention |
|-------|---------|------------|-----|-----------|
| `spans` | Raw spans from collectors | 100 | trace_id | 7 days |
| `traces-complete` | Completed traces | 20 | trace_id | 1 day |
| `sampling-decisions` | Sampling decisions | 10 | trace_id | 24 hours |

**Why Trace ID-Based Partitioning?**

```java
public class SpanPartitioner {
    
    public int partition(String traceId, int numPartitions) {
        return Math.abs(traceId.hashCode()) % numPartitions;
    }
}
```

Benefits:
- Same trace_id always goes to same partition
- All spans for a trace processed by same aggregator
- Efficient trace reconstruction (no cross-partition coordination)
- Maintains ordering per trace

**Producer Configuration:**

```java
@Configuration
public class TracingKafkaConfig {
    
    @Bean
    public ProducerFactory<String, Span> spanProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durability settings
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Batch settings for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Kafka Flow

**Normal Flow: Span Ingestion**

```
Step 1: Service Creates Span
┌─────────────────────────────────────────────────────────────┐
│ Order Service:                                              │
│ - Creates span: trace_id="abc123", span_id="span001"       │
│ - Adds tags, logs, timing                                   │
│ - Sends to collector via gRPC                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Collector Receives and Routes
┌─────────────────────────────────────────────────────────────┐
│ Collector:                                                  │
│ - Validates span format                                     │
│ - Checks sampling: Redis lookup "sampled:abc123"           │
│ - If not sampled, discard                                   │
│ - If sampled, calculate partition: hash("abc123") % 100 = 23│
│ - Buffer span (batch for 10ms)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Publish to Kafka
┌─────────────────────────────────────────────────────────────┐
│ Kafka Producer:                                             │
│   Topic: spans                                              │
│   Partition: 23 (for trace_id="abc123")                    │
│   Key: "abc123"                                             │
│   Message: {                                                │
│     "trace_id": "abc123",                                   │
│     "span_id": "span001",                                   │
│     "service_name": "order-service",                        │
│     "operation_name": "getOrder",                           │
│     "start_time": 1705312245000000,                         │
│     "duration": 350000000,                                  │
│     "tags": {...}                                           │
│   }                                                         │
│   Latency: < 1ms                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Aggregator Consumes
┌─────────────────────────────────────────────────────────────┐
│ Aggregator (assigned to partition 23):                      │
│ - Polls partition 23                                        │
│ - Receives batch of spans (all for trace_id="abc123")      │
│ - Groups by trace_id                                        │
│ - Reconstructs trace (handles out-of-order)                │
│ - When trace completes, writes to storage                   │
│ - Commits offset                                            │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Aggregator Crash**

```
Scenario: Aggregator crashes while processing spans

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Aggregator processing partition 23                    │
│ T+30s: Aggregator crashes (OOM, hardware failure)         │
│ T+30s: Kafka offset NOT committed                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+60s: Kafka consumer group rebalance                      │
│ - Coordinator detects aggregator left group                │
│ - Reassigns partition 23 to aggregator 8                   │
│ - Aggregator 8 starts from last committed offset            │
│ - Receives spans again (offset not committed)              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Aggregator 8 Processing:                                    │
│ - Receives spans from partition 23                         │
│ - Groups by trace_id                                        │
│ - Reconstructs traces (idempotent: same spans, same result)│
│ - Writes to storage (idempotent: overwrites if exists)     │
│ - Commits offset after successful write                    │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

- Spans are idempotent: Same span_id + trace_id produces same result
- Aggregator deduplicates spans (by span_id)
- Storage overwrites traces (same trace_id)

**Traffic Spike Handling:**

```
Scenario: Traffic spike (10x normal rate)

┌─────────────────────────────────────────────────────────────┐
│ Normal: 550K spans/second                                   │
│ Spike: 5.5M spans/second (10x)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Buffering:                                            │
│ - 5.5M spans/second buffered in Kafka                      │
│ - 7-day retention = sufficient capacity                    │
│ - No span loss                                              │
│                                                             │
│ Aggregator Scaling:                                         │
│ - Auto-scale: 20 → 200 aggregators                         │
│ - Each aggregator: processes assigned partitions           │
│ - Throughput: 5.5M spans/second                             │
│                                                             │
│ Result:                                                      │
│ - Spans queued in Kafka (no loss)                          │
│ - Aggregation continues at steady rate                      │
│ - All traces eventually processed                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Tracing Context?

Redis provides critical caching for tracing operations:

1. **Trace Cache**: Fast retrieval of recently accessed traces
2. **Sampling State**: Head-based sampling decisions
3. **Service Index Cache**: Frequently accessed service metadata

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Trace Cache | `trace:{trace_id}` | 5 minutes | Cached trace documents |
| Sampling State | `sampled:{trace_id}` | 1 hour | Sampling decisions |
| Service Index | `service:{service_name}` | 10 minutes | Service metadata |

**Trace Caching:**

```java
@Service
public class TraceCacheService {
    
    private final RedisTemplate<String, Trace> redis;
    private static final Duration TTL = Duration.ofMinutes(5);
    
    public Trace getTrace(String traceId) {
        String key = "trace:" + traceId;
        Trace cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Query Elasticsearch
        Trace trace = elasticsearchService.getTrace(traceId);
        
        // Cache with TTL
        if (trace != null) {
            redis.opsForValue().set(key, trace, TTL);
        }
        
        return trace;
    }
}
```

**Sampling State Caching:**

```java
@Service
public class SamplingService {
    
    private final RedisTemplate<String, Boolean> redis;
    private static final Duration TTL = Duration.ofHours(1);
    
    public boolean shouldSample(String traceId, Span firstSpan) {
        String key = "sampled:" + traceId;
        Boolean cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Make sampling decision
        boolean decision = makeSamplingDecision(traceId, firstSpan);
        
        // Cache decision
        redis.opsForValue().set(key, decision, TTL);
        
        return decision;
    }
    
    private boolean makeSamplingDecision(String traceId, Span span) {
        // Always sample errors
        if (span.hasError()) {
            return true;
        }
        
        // 1% head-based sampling
        return traceId.hashCode() % 100 == 0;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Cache Flow

**Cache HIT Path:**

```
Request: GET /v1/traces/abc123
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Query Service:                                              │
│ - Check Redis: "trace:abc123"                               │
│ - Cache HIT: Trace found in Redis                          │
│ - Return trace (latency: < 5ms)                             │
└─────────────────────────────────────────────────────────────┘
```

**Cache MISS Path:**

```
Request: GET /v1/traces/abc123
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Query Service:                                              │
│ - Check Redis: "trace:abc123"                               │
│ - Cache MISS: Not in Redis                                  │
│ - Query Elasticsearch (latency: 100ms)                      │
│ - Trace found in Elasticsearch                              │
│ - Store in Redis with 5-minute TTL                          │
│ - Return trace (total latency: 105ms)                       │
└─────────────────────────────────────────────────────────────┘
```

**Failure Scenarios:**

**Q: What if Redis is down?**

A: 
- Query service falls back to Elasticsearch directly
- Latency increases (100ms vs 5ms)
- No data loss (Redis is cache, not source of truth)
- System continues operating (graceful degradation)

**Q: Why 5-minute TTL?**

A: 
- Balance between cache hit rate and staleness
- Traces don't change after creation (immutable)
- 5 minutes captures recent query patterns
- Reduces Elasticsearch load by 80%

---

## 3. Sampling Strategies

### Head-Based Sampling

**Decision Point:** When first span of trace arrives

**Strategy:**
- Random 1% sampling (hash-based for consistency)
- Always sample errors (100%)
- Decision cached in Redis (1-hour TTL)

**Implementation:**

```java
public boolean shouldSample(String traceId, Span span) {
    // Always sample errors
    if (span.hasError()) {
        return true;
    }
    
    // Consistent sampling: hash-based
    int hash = Math.abs(traceId.hashCode());
    return (hash % 100) == 0;  // 1% sampling
}
```

**Pros:**
- Low overhead (decision at trace start)
- Consistent (same trace_id always sampled or not)
- Simple implementation

**Cons:**
- May miss important traces (slow but no errors)
- Fixed rate (can't adapt to load)

### Tail-Based Sampling

**Decision Point:** After trace completes

**Strategy:**
- Buffer all spans in aggregator
- Evaluate trace after completion
- Keep if: has errors, slow (p95+), or matches patterns
- Otherwise discard

**Implementation:**

```java
public boolean shouldKeepTrace(Trace trace) {
    // Always keep errors
    if (trace.hasError()) {
        return true;
    }
    
    // Keep slow traces (p95+)
    if (trace.getDuration() > p95Latency) {
        return true;
    }
    
    // Keep traces matching patterns (e.g., specific services)
    if (trace.matchesPattern(importantServices)) {
        return true;
    }
    
    // Otherwise discard (sampled out)
    return false;
}
```

**Pros:**
- Never miss important traces (errors, slow requests)
- Adaptive (can adjust based on load)
- Better cost/benefit (keep only important traces)

**Cons:**
- Higher memory (must buffer all spans)
- More complex implementation
- Higher latency (decision after completion)

### Adaptive Sampling

**Strategy:** Adjust sampling rate based on system load

**Implementation:**

```java
public double getSamplingRate() {
    // Get current system load
    double cpuUsage = metricsService.getCpuUsage();
    double memoryUsage = metricsService.getMemoryUsage();
    
    // Reduce sampling under load
    if (cpuUsage > 0.8 || memoryUsage > 0.8) {
        return 0.5;  // 0.5% sampling
    }
    
    // Normal sampling
    return 1.0;  // 1% sampling
}
```

**Benefits:**
- Protects system under load
- Maintains sampling during traffic spikes
- Automatic adaptation

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Kafka Partitioning | Trace ID hash | Efficient trace reconstruction |
| Trace Cache TTL | 5 minutes | Balance hit rate and staleness |
| Sampling Strategy | Hybrid (head + tail) | Cost optimization + important traces |
| Cache Failure | Fallback to Elasticsearch | Graceful degradation |

