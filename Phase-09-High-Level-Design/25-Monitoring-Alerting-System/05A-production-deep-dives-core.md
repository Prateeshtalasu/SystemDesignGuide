# Monitoring & Alerting System - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for metric buffering), caching strategy (query cache), and alerting engine (rule evaluation).

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Monitoring Context?

Kafka serves as a buffer between metric collection and storage:

1. **Metric Buffering**: Collects metrics from collectors before storage
2. **Backpressure Handling**: Handles traffic spikes without dropping metrics
3. **Replay Capability**: Can replay metrics for reprocessing
4. **Decoupling**: Separates collection from storage

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key | Retention |
|-------|---------|------------|-----|-----------|
| `metrics` | Raw metrics from collectors | 200 | metric_name | 1 day |
| `alerts` | Alert events | 20 | alert_id | 7 days |

**Why Metric Name-Based Partitioning?**

Benefits:
- Same metric name goes to same partition
- Efficient batching (similar metrics together)
- Parallel processing across partitions

### C) REAL STEP-BY-STEP SIMULATION: Kafka Flow

**Normal Flow: Metric Ingestion**

```
Step 1: Service Emits Metric
┌─────────────────────────────────────────────────────────────┐
│ Order Service:                                              │
│ - Increments counter: http_requests_total{status="200"}    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Collector Receives
┌─────────────────────────────────────────────────────────────┐
│ Collector:                                                  │
│ - Scrapes /metrics endpoint                                 │
│ - Validates metric format                                   │
│ - Calculates partition: hash("http_requests_total") % 200 = 45
│ - Buffers metrics (batch for 10ms)                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Publish to Kafka
┌─────────────────────────────────────────────────────────────┐
│ Kafka Producer:                                             │
│   Topic: metrics                                            │
│   Partition: 45                                             │
│   Key: "http_requests_total"                                │
│   Message: {metric data}                                    │
│   Latency: < 1ms                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Storage Consumer
┌─────────────────────────────────────────────────────────────┐
│ Storage Consumer (assigned to partition 45):                │
│ - Consumes batch of metrics                                 │
│ - Writes to time-series DB                                 │
│ - Commits offset                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Monitoring Context?

Redis provides caching for:

1. **Query Results**: Cache frequently accessed query results
2. **Alert State**: Store active alerts for fast lookup
3. **Metric Metadata**: Cache metric names and labels

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Query Results | `query:{query_hash}` | 1 minute | Cached query results |
| Alert State | `alert:{alert_id}` | Until resolved | Active alert state |
| Metric Metadata | `metric:{metric_name}` | 10 minutes | Metric metadata |

**Query Caching:**

```java
@Service
public class QueryCacheService {
    
    private final RedisTemplate<String, QueryResult> redis;
    private static final Duration TTL = Duration.ofMinutes(1);
    
    public QueryResult getCachedQuery(String query) {
        String key = "query:" + hash(query);
        QueryResult cached = redis.opsForValue().get(key);
        return cached;
    }
    
    public void cacheQuery(String query, QueryResult result) {
        String key = "query:" + hash(query);
        redis.opsForValue().set(key, result, TTL);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: Cache Flow

**Cache HIT Path:**

```
Request: Query metrics
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Query Engine:                                               │
│ - Check Redis cache: "query:abc123"                         │
│ - Cache HIT: Result found in Redis                         │
│ - Return result (latency: < 5ms)                            │
└─────────────────────────────────────────────────────────────┘
```

**Cache MISS Path:**

```
Request: Query metrics
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Query Engine:                                               │
│ - Check Redis cache: "query:abc123"                         │
│ - Cache MISS: Not in Redis                                  │
│ - Execute query against time-series DB (latency: 50ms)     │
│ - Store result in Redis with 1-minute TTL                   │
│ - Return result (total latency: 55ms)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Alerting Engine

### Alert Rule Evaluation

**Evaluation Cycle:**

```java
@Service
public class AlertingEngine {
    
    @Scheduled(fixedRate = 15000)  // Every 15 seconds
    public void evaluateRules() {
        List<AlertRule> rules = ruleRepository.findAll();
        
        for (AlertRule rule : rules) {
            try {
                evaluateRule(rule);
            } catch (Exception e) {
                log.error("Error evaluating rule: " + rule.getId(), e);
            }
        }
    }
    
    private void evaluateRule(AlertRule rule) {
        // Execute query expression
        QueryResult result = queryEngine.execute(rule.getExpr());
        
        // Check if condition is met
        if (result.matchesCondition(rule.getThreshold())) {
            // Check if alert should fire (for duration)
            AlertState state = getOrCreateAlertState(rule.getId());
            state.recordConditionMet();
            
            if (state.shouldFire(rule.getForDuration())) {
                fireAlert(rule, result);
            }
        } else {
            // Condition not met, reset state
            resetAlertState(rule.getId());
        }
    }
}
```

**Alert State Management:**

```java
@Service
public class AlertStateService {
    
    private final RedisTemplate<String, AlertState> redis;
    
    public AlertState getOrCreateAlertState(String ruleId) {
        String key = "alert_state:" + ruleId;
        AlertState state = redis.opsForValue().get(key);
        
        if (state == null) {
            state = new AlertState(ruleId);
            redis.opsForValue().set(key, state);
        }
        
        return state;
    }
    
    public void resetAlertState(String ruleId) {
        String key = "alert_state:" + ruleId;
        redis.delete(key);
    }
}
```

### Alert Grouping and Deduplication

**Problem:** Multiple alerts for same issue (e.g., same service, different instances)

**Solution:** Alert grouping by labels

```java
@Service
public class AlertGroupingService {
    
    public void fireAlert(AlertRule rule, QueryResult result) {
        // Generate alert ID from labels (deduplication key)
        String alertId = generateAlertId(rule, result);
        
        // Check if alert already exists
        Alert existingAlert = alertRepository.findById(alertId);
        if (existingAlert != null && existingAlert.isActive()) {
            // Alert already exists, update timestamp
            existingAlert.updateTimestamp();
            return;
        }
        
        // Create new alert
        Alert alert = Alert.builder()
            .id(alertId)
            .ruleId(rule.getId())
            .labels(rule.getLabels())
            .annotations(rule.getAnnotations())
            .value(result.getValue())
            .state("firing")
            .firedAt(Instant.now())
            .build();
        
        alertRepository.save(alert);
        notificationService.send(alert);
    }
    
    private String generateAlertId(AlertRule rule, QueryResult result) {
        // Generate ID from rule ID + label values (for grouping)
        return rule.getId() + ":" + hashLabels(result.getLabels());
    }
}
```

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Kafka Partitioning | Metric name hash | Efficient batching |
| Query Cache TTL | 1 minute | Balance hit rate and freshness |
| Alert Evaluation | Every 15 seconds | Balance responsiveness and load |
| Alert Grouping | By labels | Reduce alert fatigue |

