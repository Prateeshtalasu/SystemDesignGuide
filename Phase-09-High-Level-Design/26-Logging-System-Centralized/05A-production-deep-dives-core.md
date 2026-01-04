# Centralized Logging System - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for log buffering), caching strategy (search cache), and search engine (Elasticsearch).

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Logging Context?

Kafka serves as a buffer between log collection and storage:

1. **Log Buffering**: Collects logs from collectors before storage
2. **Backpressure Handling**: Handles traffic spikes without dropping logs
3. **Replay Capability**: Can replay logs for reprocessing
4. **Decoupling**: Separates collection from storage

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Key | Retention |
|-------|---------|------------|-----|-----------|
| `logs` | Raw logs from collectors | 200 | service_name | 1 day |

**Why Service-Based Partitioning?**

Benefits:
- Same service logs go to same partition
- Efficient batching (similar logs together)
- Parallel processing across partitions

### C) REAL STEP-BY-STEP SIMULATION: Kafka Flow

**Normal Flow: Log Ingestion**

```
Step 1: Service Emits Log
┌─────────────────────────────────────────────────────────────┐
│ Order Service: ERROR Payment processing failed             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Collector Receives
┌─────────────────────────────────────────────────────────────┐
│ Collector:                                                  │
│ - Parses log format                                         │
│ - Calculates partition: hash("order-service") % 200 = 45   │
│ - Buffers logs (batch for 10ms)                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Publish to Kafka
┌─────────────────────────────────────────────────────────────┐
│ Kafka Producer:                                             │
│   Topic: logs                                              │
│   Partition: 45                                            │
│   Key: "order-service"                                      │
│   Message: {log data}                                       │
│   Latency: < 1ms                                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Storage Consumer
┌─────────────────────────────────────────────────────────────┐
│ Storage Consumer (assigned to partition 45):                │
│ - Consumes batch of logs                                    │
│ - Writes to Elasticsearch                                  │
│ - Commits offset                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Logging Context?

Redis provides caching for:

1. **Search Results**: Cache frequently accessed search results
2. **Query Metadata**: Cache query parsing results

### B) OUR USAGE: How We Use Redis Here

**Cache Types:**

| Cache | Key | TTL | Purpose |
|-------|-----|-----|---------|
| Search Results | `search:{query_hash}` | 1 minute | Cached search results |

**Search Caching:**

```java
@Service
public class SearchCacheService {
    
    private final RedisTemplate<String, SearchResult> redis;
    private static final Duration TTL = Duration.ofMinutes(1);
    
    public SearchResult getCachedSearch(String query) {
        String key = "search:" + hash(query);
        return redis.opsForValue().get(key);
    }
    
    public void cacheSearch(String query, SearchResult result) {
        String key = "search:" + hash(query);
        redis.opsForValue().set(key, result, TTL);
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when a cached search result expires and many requests simultaneously try to regenerate it, overwhelming Elasticsearch.

**Scenario:**
```
T+0s:   Popular search query cache expires
T+0s:   1,000 users search for same query simultaneously
T+0s:   All 1,000 requests see cache MISS
T+0s:   All 1,000 requests hit Elasticsearch simultaneously
T+1s:   Elasticsearch overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread queries Elasticsearch on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class SearchCacheService {
    
    private final RedisTemplate<String, SearchResult> redis;
    private final DistributedLock distributedLock;
    private final ElasticsearchService elasticsearch;
    
    public SearchResult getCachedSearch(String query) {
        String key = "search:" + hash(query);
        
        // 1. Try cache first
        SearchResult cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:" + key;
        boolean acquired = distributedLock.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is computing, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fallback: Query Elasticsearch anyway (better than blocking)
            return elasticsearch.search(query);
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Query Elasticsearch (only one thread does this)
            SearchResult result = elasticsearch.search(query);
            
            // 5. Cache with TTL jitter (prevent simultaneous expiration)
            long baseTtl = 60; // 1 minute
            long jitter = (long)(baseTtl * 0.1 * (Math.random() * 2 - 1));  // ±10%
            long finalTtl = baseTtl + jitter;
            
            redis.opsForValue().set(key, result, Duration.ofSeconds(finalTtl));
            
            return result;
            
        } finally {
            distributedLock.unlock(lockKey);
        }
    }
}
```

---

## 3. Search Engine (Elasticsearch)

### Full-Text Search

**Index Mapping:**

```json
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "service": { "type": "keyword" },
      "message": { "type": "text", "analyzer": "standard" },
      "fields": { "type": "object" }
    }
  }
}
```

**Query Execution:**

```java
@Service
public class SearchService {
    
    public SearchResult search(SearchRequest request) {
        // Build Elasticsearch query
        BoolQueryBuilder query = QueryBuilders.boolQuery();
        
        // Full-text search
        if (request.getQuery() != null) {
            query.must(QueryBuilders.matchQuery("message", request.getQuery()));
        }
        
        // Service filter
        if (request.getService() != null) {
            query.must(QueryBuilders.termQuery("service", request.getService()));
        }
        
        // Level filter
        if (request.getLevel() != null) {
            query.must(QueryBuilders.termQuery("level", request.getLevel()));
        }
        
        // Time range
        query.must(QueryBuilders.rangeQuery("timestamp")
            .gte(request.getStartTime())
            .lte(request.getEndTime()));
        
        // Execute query
        SearchResponse response = elasticsearchClient.search(
            SearchRequest.of(s -> s
                .index("logs-*")
                .query(query)
                .size(request.getLimit())
            ),
            LogDocument.class
        );
        
        return SearchResult.from(response);
    }
}
```

---

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Kafka Partitioning | Service hash | Efficient batching |
| Search Cache TTL | 1 minute | Balance hit rate and freshness |
| Search Engine | Elasticsearch | Full-text search, time-series optimized |

