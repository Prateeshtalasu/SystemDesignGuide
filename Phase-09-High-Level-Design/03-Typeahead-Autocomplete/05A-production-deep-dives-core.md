# Typeahead / Autocomplete - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for search log collection), caching strategy, and search capabilities. These are the fundamental building blocks that enable the typeahead system to meet its extreme latency requirements.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Typeahead Context?

In the typeahead system, Kafka serves a different purpose than in typical applications. Instead of being in the request path, Kafka is used for **offline data collection** to build the suggestion index.

**What problems does Kafka solve here?**

1. **Log Collection**: Captures all search queries for analysis
2. **Buffering**: Handles search traffic spikes without data loss
3. **Decoupling**: Search service doesn't wait for analytics
4. **Replay**: Can rebuild index from historical data

**This is NOT in the critical path:**

```
User types "how" → Suggestion Service → Trie Lookup → Response
                          │
                          │ (async, fire-and-forget)
                          ▼
                        Kafka (search-queries topic)
                          │
                          │ (offline processing)
                          ▼
                    Spark Pipeline → New Trie Index
```

### B) OUR USAGE: How We Use Kafka Here

**Why async for log collection?**

The suggestion request must complete in < 50ms. Logging synchronously would add:
- Kafka produce: +5-10ms
- Acknowledgment: +5ms
- Total: +10-15ms (30% latency increase)

By using async fire-and-forget, we add < 1ms to the request path.

**Events in our system:**

| Event | Purpose | Volume |
|-------|---------|--------|
| `search-query` | Raw search query for aggregation | 10 billion/day |
| `suggestion-shown` | Which suggestions were displayed | 10 billion/day |
| `suggestion-clicked` | Which suggestion was selected | 1 billion/day |

**Topic Design:**

```
Topic: search-queries
Partitions: 100
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
```

**Why 100 partitions?**
- 10 billion events/day = 115K events/second
- Each partition handles ~1.5K events/second comfortably
- 100 partitions allow 100 parallel consumers for processing

**Partition Key Choice:**

We partition by `user_id` hash. This ensures:
- All queries from same user go to same partition
- Enables session-based analysis
- Even distribution across partitions

**Hot Partition Risk:**

Anonymous users (no user_id) could create hot partition. Mitigation:
- Use `session_id` for anonymous users
- Fall back to random partition if no session

**Consumer Group Design:**

```
Consumer Group: query-aggregator
Consumers: 100 instances (one per partition)
Processing: Spark Streaming
```

**Ordering Guarantees:**

- **Per-user ordering**: Guaranteed (same partition)
- **Global ordering**: Not needed for aggregation

**Offset Management:**

Spark Streaming manages offsets via checkpointing:

```java
Dataset<Row> searchEvents = spark
    .readStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "search-queries")
    .option("startingOffsets", "latest")
    .load();

searchEvents
    .writeStream()
    .option("checkpointLocation", "s3://checkpoints/query-agg/")
    .trigger(Trigger.ProcessingTime("1 minute"))
    .start();
```

**Deduplication Strategy:**

Not critical for aggregation. Slight over-counting is acceptable since:
- We're computing popularity scores, not exact counts
- Log-scale scoring smooths out duplicates
- Cost of deduplication > benefit

### C) REAL STEP-BY-STEP SIMULATION

**Event Flow:**

```
User types "how to" in search box
    ↓
Suggestion Service receives GET /suggestions?q=how+to
    ↓
Trie lookup: find("how to") → returns top 10 suggestions
    ↓
[ASYNC] Kafka Producer fires event (no wait):
    {
      "query": "how to",
      "timestamp": 1705312800000,
      "user_id": "abc123",
      "session_id": "xyz789",
      "client": "web",
      "suggestions_shown": ["how to tie a tie", "how to lose weight", ...]
    }
    ↓
Response returned to user (< 50ms total)
    ↓
[OFFLINE] Kafka Consumer (Spark) polls events
    ↓
[OFFLINE] Hourly aggregation job:
    - Count queries by normalized text
    - Calculate trending scores
    - Update query_aggregates table
    ↓
[OFFLINE] Trie Builder job:
    - Read top 5B queries by score
    - Build new Trie with pre-computed suggestions
    - Upload to S3
    ↓
[OFFLINE] Servers detect new version, load new Trie
```

**Failure Scenarios:**

**Q: What if Kafka is down?**

A: Fire-and-forget means we lose events, but:
- Suggestions still work (Trie is in-memory)
- Next hour's index may be slightly stale
- Acceptable trade-off for latency

**Q: What if Spark job fails?**

A: 
- Servers keep serving current index
- Alert fired, manual intervention
- Can replay from Kafka (7-day retention)

**Q: What if consumer lag builds up?**

A: 
- Index updates delayed
- Suggestions become stale (hours, not minutes)
- Add more consumers or increase processing power

### Event Schema

```java
public class SearchQueryEvent {
    private String query;           // Raw query text
    private long timestamp;         // Event time
    private String userId;          // User ID (nullable)
    private String sessionId;       // Session ID
    private String client;          // web, ios, android
    private List<String> suggestionsShown;  // What we showed
    private String selectedSuggestion;      // What user clicked (nullable)
    private int resultCount;        // Search results count
}
```

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, SearchQueryEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Fire-and-forget settings
        config.put(ProducerConfig.ACKS_CONFIG, "0");  // No acknowledgment
        config.put(ProducerConfig.RETRIES_CONFIG, 0);  // No retries
        
        // Batching for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);  // 64KB batches
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // Wait 10ms to batch
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Why acks=0?**

- Logging is not critical (can lose some events)
- Prioritize suggestion latency over log durability
- 99.9% of events still make it through

---

## 2. Caching (Redis) - NOT APPLICABLE

### Why No Redis for Typeahead?

This is a key architectural insight: **the Trie IS the cache**.

**Traditional caching approach:**

```
Request → App → Redis (cache) → Database (if miss) → Response
```

**Typeahead approach:**

```
Request → App → In-Memory Trie → Response
```

**Why Redis would hurt, not help:**

| Aspect | With Redis | With Trie |
|--------|------------|-----------|
| Lookup latency | 1-5ms network | 0.1ms memory |
| Cache miss | Fall back to... what? | N/A, all data in memory |
| Consistency | Cache invalidation needed | Index updated atomically |
| Complexity | Higher | Lower |

**The Trie provides:**
- O(prefix length) lookups (~0.1ms)
- Pre-computed suggestions at each node
- No cache misses (all data in memory)
- No invalidation needed (atomic index swap)

### What About CDN Caching?

CDN caching IS used for popular prefixes:

**CDN Cache Configuration:**

```
Cache Key: /suggestions?q={prefix}&limit={n}&lang={lang}
TTL: 5 minutes
Vary: Accept-Encoding
```

**Why 5 minutes?**
- Popular queries change slowly
- Trending queries update hourly anyway
- Reduces origin load by ~40%

**What's cached at CDN:**

| Prefix | Requests/hour | Cache Benefit |
|--------|---------------|---------------|
| "how" | 10 million | Huge |
| "how to" | 5 million | Huge |
| "how to tie" | 100K | Moderate |
| "how to tie a" | 10K | Small |
| "xyz123abc" | 1 | None |

**Cache Decision Logic:**

```java
public class CacheStrategy {
    
    public CacheControl getCacheControl(String prefix) {
        // Short prefixes are highly cacheable
        if (prefix.length() <= 3) {
            return CacheControl.maxAge(5, TimeUnit.MINUTES).cachePublic();
        }
        
        // Medium prefixes have moderate caching
        if (prefix.length() <= 10) {
            return CacheControl.maxAge(2, TimeUnit.MINUTES).cachePublic();
        }
        
        // Long prefixes are unique, minimal caching
        return CacheControl.maxAge(30, TimeUnit.SECONDS).cachePublic();
    }
}
```

### Browser Caching

Client-side caching reduces requests further:

```javascript
// Browser caches responses
const response = await fetch('/v1/suggestions?q=' + query, {
    headers: {
        'Cache-Control': 'max-age=60'  // 1 minute browser cache
    }
});
```

**Combined cache layers:**

```
Request for "how to"
    ↓
Browser cache? (1 min TTL) → HIT → Done
    ↓ MISS
CDN cache? (5 min TTL) → HIT → Done
    ↓ MISS
Origin (Trie lookup) → Response → Cache at CDN → Cache at Browser
```

---

## 3. Search (Elasticsearch / OpenSearch) - NOT APPLICABLE

### Why No Elasticsearch for Typeahead?

Elasticsearch is designed for full-text search, not prefix completion. Using it would be:

1. **Slower**: ES prefix queries are O(log N), Trie is O(prefix length)
2. **Overkill**: We don't need ranking, fuzzy matching, or full-text
3. **More complex**: Cluster management, sharding, replication
4. **Higher latency**: Network call vs in-memory lookup

**Comparison:**

| Aspect | Elasticsearch | In-Memory Trie |
|--------|---------------|----------------|
| Lookup latency | 10-50ms | 0.1-1ms |
| Memory per query | Shared cluster | ~100 bytes |
| Scaling | Horizontal (complex) | Vertical (simple) |
| Pre-computed results | No | Yes |
| Fuzzy matching | Yes | No (not needed) |

### When Would ES Make Sense?

If requirements changed to include:
- Fuzzy matching ("hwo to" → "how to")
- Typo correction
- Synonym expansion
- Full-text search within suggestions

Then we might add ES as a **fallback** for:
- No Trie matches
- Misspelled queries
- Long-tail queries

**Hypothetical hybrid architecture:**

```
Request for "hwo to" (misspelled)
    ↓
Trie lookup → No matches
    ↓
Elasticsearch fuzzy query → "how to" suggestions
    ↓
Response (slower, but better than nothing)
```

But for the current requirements (prefix matching only), Trie is optimal.

---

## 4. Ranking Algorithm

### Scoring Components

```java
public class SuggestionScorer {
    
    public double calculateScore(QueryStats stats, UserContext user) {
        double baseScore = calculateBaseScore(stats);
        double trendingBoost = calculateTrendingBoost(stats);
        double personalBoost = calculatePersonalizationBoost(stats.getQuery(), user);
        double freshnessBoost = calculateFreshnessBoost(stats);
        
        return baseScore * trendingBoost * personalBoost * freshnessBoost;
    }
    
    // Base score from frequency (log-scaled)
    private double calculateBaseScore(QueryStats stats) {
        double weightedCount = 
            stats.getHourlyCount() * 10.0 +
            stats.getDailyCount() * 5.0 +
            stats.getWeeklyCount() * 2.0 +
            stats.getMonthlyCount() * 1.0;
        
        return Math.log10(weightedCount + 1);
    }
    
    // Boost for trending queries
    private double calculateTrendingBoost(QueryStats stats) {
        double growthRate = (double) stats.getHourlyCount() / 
            (stats.getPreviousHourCount() + 1);
        
        if (growthRate > 5.0) return 2.0;      // Viral
        if (growthRate > 2.0) return 1.5;      // Trending
        if (growthRate > 1.5) return 1.2;      // Growing
        return 1.0;
    }
    
    // Boost for user's recent searches
    private double calculatePersonalizationBoost(String query, UserContext user) {
        if (user == null) return 1.0;
        
        // Exact match with recent search
        if (user.getRecentSearches().contains(query)) {
            return 3.0;
        }
        
        // Prefix match
        for (String recent : user.getRecentSearches()) {
            if (recent.startsWith(query)) {
                return 2.0;
            }
        }
        
        return 1.0;
    }
    
    // Slight boost for recent queries
    private double calculateFreshnessBoost(QueryStats stats) {
        long hoursSinceLastSeen = 
            Duration.between(stats.getLastSeen(), Instant.now()).toHours();
        
        if (hoursSinceLastSeen < 1) return 1.1;
        if (hoursSinceLastSeen < 24) return 1.05;
        return 1.0;
    }
}
```

---

## 5. Content Filtering

### Filter Pipeline

```java
@Service
public class QueryFilter {
    
    private final Set<String> blockedQueries;
    private final Set<String> blockedWords;
    private final Pattern profanityPattern;
    private final Pattern piiPattern;
    
    public FilterResult filter(String query) {
        String normalized = normalize(query);
        
        // Check exact blocklist
        if (blockedQueries.contains(normalized)) {
            return FilterResult.blocked("BLOCKLIST");
        }
        
        // Check blocked words
        for (String word : normalized.split("\\s+")) {
            if (blockedWords.contains(word)) {
                return FilterResult.blocked("BLOCKED_WORD");
            }
        }
        
        // Check profanity
        if (profanityPattern.matcher(normalized).find()) {
            return FilterResult.blocked("PROFANITY");
        }
        
        // Check PII (emails, phones, SSN)
        if (piiPattern.matcher(normalized).find()) {
            return FilterResult.blocked("PII");
        }
        
        return FilterResult.allowed();
    }
    
    @Scheduled(fixedRate = 3600000)  // Hourly
    public void refreshBlocklists() {
        this.blockedQueries = loadBlockedQueries();
        this.blockedWords = loadBlockedWords();
    }
}
```

---

## 6. Index Distribution

### Zookeeper Coordination

```java
@Service
public class IndexVersionManager {
    
    private final CuratorFramework curator;
    private final String versionPath = "/typeahead/current_version";
    
    // Watch for version changes
    @PostConstruct
    public void watchVersionChanges() {
        curator.getData()
            .usingWatcher((CuratorWatcher) event -> {
                if (event.getType() == EventType.NodeDataChanged) {
                    String newVersion = getCurrentVersion();
                    loadNewIndex(newVersion);
                }
            })
            .forPath(versionPath);
    }
    
    public String getCurrentVersion() {
        byte[] data = curator.getData().forPath(versionPath);
        return new String(data);
    }
    
    public void updateVersion(String version) {
        curator.setData().forPath(versionPath, version.getBytes());
    }
}
```

### Index Loading

```java
@Service
public class TrieLoader {
    
    private volatile Trie currentTrie;
    private final S3Client s3Client;
    
    public void loadNewIndex(String version) {
        log.info("Loading new index version: {}", version);
        
        // Download from S3
        String key = "v" + version + "/trie.bin";
        Path tempFile = Files.createTempFile("trie", ".bin");
        
        s3Client.getObject(
            GetObjectRequest.builder()
                .bucket("typeahead-index")
                .key(key)
                .build(),
            tempFile
        );
        
        // Deserialize
        Trie newTrie = deserializeTrie(tempFile);
        
        // Atomic swap
        Trie oldTrie = this.currentTrie;
        this.currentTrie = newTrie;
        
        // Clean up
        Files.delete(tempFile);
        log.info("Index updated to version: {}", version);
        
        // Old trie will be garbage collected
    }
    
    public Trie getCurrentTrie() {
        return currentTrie;
    }
}
```

---

## 7. Debouncing (Client-Side)

### Why Debounce?

Without debouncing, typing "hello" sends 5 requests:
```
h → request
he → request
hel → request
hell → request
hello → request
```

With 150ms debounce:
```
h (wait 150ms)
he (wait 150ms)
hel (wait 150ms)
hell (wait 150ms)
hello (150ms passed) → request
```

### Implementation

```javascript
// Client-side debouncing
function debounce(func, wait) {
    let timeout;
    return function(...args) {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
    };
}

const fetchSuggestions = debounce(async (query) => {
    if (query.length < 2) return;
    
    const response = await fetch(`/v1/suggestions?q=${encodeURIComponent(query)}`);
    const data = await response.json();
    displaySuggestions(data.suggestions);
}, 150);  // 150ms debounce

// Usage
searchInput.addEventListener('input', (e) => {
    fetchSuggestions(e.target.value);
});
```

**Why 150ms?**
- Fast enough to feel responsive
- Slow enough to reduce requests by ~80%
- Matches average typing speed

---

## Summary

| Component | Technology | Status | Key Configuration |
|-----------|------------|--------|-------------------|
| Async Messaging | Kafka | Active (offline) | 100 partitions, acks=0, 7-day retention |
| Caching (Redis) | N/A | Not needed | Trie IS the cache |
| Caching (CDN) | CloudFront | Active | 5-min TTL for popular prefixes |
| Search (ES) | N/A | Not needed | Trie is faster for prefix matching |

### Why This Architecture?

1. **Trie eliminates need for Redis**: In-memory lookup is faster than any cache
2. **Kafka is offline only**: No impact on request latency
3. **CDN handles popular queries**: 40% hit rate reduces origin load
4. **No Elasticsearch**: Prefix matching doesn't need full-text search

### Key Insight

The typeahead system is **read-optimized to the extreme**:
- All computation happens offline (Trie building)
- Runtime is pure lookup (O(prefix length))
- No database calls, no cache calls, no network calls for core lookup

