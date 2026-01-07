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

## 4. Log Aggregation Implementation

### Log Aggregation Service

```java
@Service
public class LogAggregationService {
    
    private final KafkaTemplate<String, LogEvent> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private final LogParser logParser;
    
    /**
     * Aggregate logs from multiple services.
     * Handles batching, parsing, and routing to Kafka.
     */
    public void aggregateLogs(List<RawLog> rawLogs) {
        // Batch logs by service for efficient processing
        Map<String, List<RawLog>> logsByService = rawLogs.stream()
            .collect(Collectors.groupingBy(RawLog::getServiceName));
        
        // Process each service's logs
        for (Map.Entry<String, List<RawLog>> entry : logsByService.entrySet()) {
            String serviceName = entry.getKey();
            List<RawLog> serviceLogs = entry.getValue();
            
            // Parse logs
            List<LogEvent> parsedLogs = serviceLogs.stream()
                .map(logParser::parse)
                .filter(Objects::nonNull)  // Filter parsing errors
                .collect(Collectors.toList());
            
            // Send to Kafka (batched by service)
            for (LogEvent logEvent : parsedLogs) {
                kafkaTemplate.send("logs", serviceName, logEvent);
            }
        }
    }
    
    /**
     * Aggregate logs with deduplication.
     */
    public void aggregateLogsWithDeduplication(List<RawLog> rawLogs) {
        Set<String> seenLogIds = ConcurrentHashMap.newKeySet();
        
        List<LogEvent> uniqueLogs = rawLogs.stream()
            .map(logParser::parse)
            .filter(Objects::nonNull)
            .filter(log -> {
                String logId = generateLogId(log);
                return seenLogIds.add(logId);  // Only add if not seen
            })
            .collect(Collectors.toList());
        
        // Send to Kafka
        for (LogEvent logEvent : uniqueLogs) {
            kafkaTemplate.send("logs", logEvent.getServiceName(), logEvent);
        }
    }
    
    private String generateLogId(LogEvent log) {
        return log.getServiceName() + ":" + log.getTimestamp() + ":" + 
               DigestUtils.md5Hex(log.getMessage());
    }
}
```

### Log Parser Implementation

```java
@Component
public class LogParser {
    
    private final List<LogFormatParser> parsers;
    
    public LogParser() {
        this.parsers = Arrays.asList(
            new JSONLogParser(),
            new StructuredLogParser(),
            new PlainTextLogParser()
        );
    }
    
    /**
     * Parse raw log into structured LogEvent.
     */
    public LogEvent parse(RawLog rawLog) {
        for (LogFormatParser parser : parsers) {
            if (parser.canParse(rawLog)) {
                try {
                    return parser.parse(rawLog);
                } catch (Exception e) {
                    // Try next parser
                    continue;
                }
            }
        }
        
        // Fallback: parse as plain text
        return parseAsPlainText(rawLog);
    }
    
    /**
     * Parse JSON log format.
     */
    private static class JSONLogParser implements LogFormatParser {
        private final ObjectMapper objectMapper = new ObjectMapper();
        
        @Override
        public boolean canParse(RawLog rawLog) {
            try {
                objectMapper.readTree(rawLog.getContent());
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        
        @Override
        public LogEvent parse(RawLog rawLog) throws Exception {
            JsonNode json = objectMapper.readTree(rawLog.getContent());
            
            return LogEvent.builder()
                .timestamp(parseTimestamp(json.get("timestamp").asText()))
                .level(LogLevel.valueOf(json.get("level").asText().toUpperCase()))
                .serviceName(rawLog.getServiceName())
                .message(json.get("message").asText())
                .fields(extractFields(json))
                .build();
        }
        
        private Map<String, String> extractFields(JsonNode json) {
            Map<String, String> fields = new HashMap<>();
            json.fields().forEachRemaining(entry -> {
                if (!entry.getKey().equals("timestamp") && 
                    !entry.getKey().equals("level") && 
                    !entry.getKey().equals("message")) {
                    fields.put(entry.getKey(), entry.getValue().asText());
                }
            });
            return fields;
        }
    }
    
    /**
     * Parse structured log format (e.g., "2024-01-15 10:30:00 [INFO] service: message").
     */
    private static class StructuredLogParser implements LogFormatParser {
        private static final Pattern STRUCTURED_PATTERN = Pattern.compile(
            "(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}) \\[(\\w+)\\] (\\w+): (.+)"
        );
        
        @Override
        public boolean canParse(RawLog rawLog) {
            return STRUCTURED_PATTERN.matcher(rawLog.getContent()).matches();
        }
        
        @Override
        public LogEvent parse(RawLog rawLog) throws Exception {
            Matcher matcher = STRUCTURED_PATTERN.matcher(rawLog.getContent());
            if (!matcher.find()) {
                throw new IllegalArgumentException("Invalid structured log format");
            }
            
            return LogEvent.builder()
                .timestamp(parseTimestamp(matcher.group(1)))
                .level(LogLevel.valueOf(matcher.group(2).toUpperCase()))
                .serviceName(matcher.group(3))
                .message(matcher.group(4))
                .build();
        }
    }
    
    /**
     * Parse plain text log (fallback).
     */
    private LogEvent parseAsPlainText(RawLog rawLog) {
        return LogEvent.builder()
            .timestamp(Instant.now())
            .level(LogLevel.INFO)  // Default level
            .serviceName(rawLog.getServiceName())
            .message(rawLog.getContent())
            .build();
    }
    
    interface LogFormatParser {
        boolean canParse(RawLog rawLog);
        LogEvent parse(RawLog rawLog) throws Exception;
    }
}
```

## 5. Log Indexing Implementation

### Elasticsearch Indexing Service

```java
@Service
public class LogIndexingService {
    
    private final ElasticsearchClient elasticsearchClient;
    private final ObjectMapper objectMapper;
    
    /**
     * Index log events to Elasticsearch.
     */
    public void indexLogs(List<LogEvent> logEvents) {
        // Batch index for efficiency
        BulkRequest.Builder bulkRequest = new BulkRequest.Builder();
        
        for (LogEvent logEvent : logEvents) {
            String indexName = generateIndexName(logEvent.getTimestamp());
            
            bulkRequest.operations(op -> op
                .index(idx -> idx
                    .index(indexName)
                    .id(generateDocumentId(logEvent))
                    .document(toDocument(logEvent))
                )
            );
        }
        
        // Execute bulk index
        try {
            BulkResponse response = elasticsearchClient.bulk(bulkRequest.build());
            if (response.errors()) {
                log.error("Some log indexing failed: {}", response);
            }
        } catch (Exception e) {
            log.error("Failed to index logs", e);
            throw new LogIndexingException("Failed to index logs", e);
        }
    }
    
    /**
     * Generate index name based on timestamp (daily indices).
     */
    private String generateIndexName(Instant timestamp) {
        LocalDate date = timestamp.atZone(ZoneOffset.UTC).toLocalDate();
        return String.format("logs-%04d-%02d-%02d", 
            date.getYear(), date.getMonthValue(), date.getDayOfMonth());
    }
    
    /**
     * Generate unique document ID.
     */
    private String generateDocumentId(LogEvent logEvent) {
        String content = logEvent.getServiceName() + ":" + 
                        logEvent.getTimestamp() + ":" + 
                        logEvent.getMessage();
        return DigestUtils.sha256Hex(content);
    }
    
    /**
     * Convert LogEvent to Elasticsearch document.
     */
    private Map<String, Object> toDocument(LogEvent logEvent) {
        Map<String, Object> doc = new HashMap<>();
        doc.put("timestamp", logEvent.getTimestamp().toString());
        doc.put("level", logEvent.getLevel().name());
        doc.put("service", logEvent.getServiceName());
        doc.put("message", logEvent.getMessage());
        
        if (logEvent.getFields() != null && !logEvent.getFields().isEmpty()) {
            doc.put("fields", logEvent.getFields());
        }
        
        return doc;
    }
    
    /**
     * Create index template for log indices.
     */
    public void createIndexTemplate() {
        try {
            elasticsearchClient.indices().putTemplate(t -> t
                .name("logs-template")
                .indexPatterns("logs-*")
                .template(tmpl -> tmpl
                    .mappings(m -> m
                        .properties("timestamp", p -> p.date(d -> d))
                        .properties("level", p -> p.keyword(k -> k))
                        .properties("service", p -> p.keyword(k -> k))
                        .properties("message", p -> p.text(txt -> txt.analyzer("standard")))
                        .properties("fields", p -> p.object(o -> o.dynamic(DynamicMapping.True)))
                    )
                    .settings(s -> s
                        .numberOfShards("5")
                        .numberOfReplicas("1")
                        .refreshInterval("30s")
                    )
                )
            );
        } catch (Exception e) {
            log.error("Failed to create index template", e);
        }
    }
}
```

### Kafka Consumer for Log Indexing

```java
@Component
public class LogIndexingConsumer {
    
    private final LogIndexingService indexingService;
    private final int batchSize = 1000;
    private final List<LogEvent> batch = new ArrayList<>();
    private final ScheduledExecutorService scheduler;
    
    @KafkaListener(topics = "logs", groupId = "log-indexing")
    public void consume(ConsumerRecord<String, LogEvent> record) {
        LogEvent logEvent = record.value();
        batch.add(logEvent);
        
        // Index when batch is full
        if (batch.size() >= batchSize) {
            indexBatch();
        }
    }
    
    /**
     * Index batch of logs.
     */
    private synchronized void indexBatch() {
        if (batch.isEmpty()) return;
        
        List<LogEvent> logsToIndex = new ArrayList<>(batch);
        batch.clear();
        
        try {
            indexingService.indexLogs(logsToIndex);
        } catch (Exception e) {
            log.error("Failed to index batch, will retry", e);
            // Could implement retry logic or dead letter queue
        }
    }
    
    /**
     * Periodic batch indexing (for low-volume scenarios).
     */
    @PostConstruct
    public void startPeriodicIndexing() {
        scheduler.scheduleAtFixedRate(
            this::indexBatch,
            5, 5, TimeUnit.SECONDS
        );
    }
}
```

## Summary

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Kafka Partitioning | Service hash | Efficient batching |
| Search Cache TTL | 1 minute | Balance hit rate and freshness |
| Search Engine | Elasticsearch | Full-text search, time-series optimized |
| Log Parsing | Multi-format parser | Support JSON, structured, plain text |
| Log Indexing | Daily indices | Efficient time-range queries, easy retention |
| Batch Indexing | 1000 logs/batch | Balance throughput and latency |

