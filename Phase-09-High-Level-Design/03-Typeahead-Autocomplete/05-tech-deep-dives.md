# Typeahead / Autocomplete - Technology Deep Dives

## 1. Trie Data Structure

### What is a Trie?

A Trie (pronounced "try", from re**trie**val) is a tree-like data structure where each node represents a character. Paths from root to nodes form prefixes of stored strings.

### Why Trie for Typeahead?

| Data Structure | Prefix Search Time | Space | Notes |
|----------------|-------------------|-------|-------|
| Array + Binary Search | O(log N × M) | O(N × M) | N queries, M avg length |
| Hash Map | O(N × M) | O(N × M) | Must check all keys |
| **Trie** | **O(M)** | O(N × M) shared | M = prefix length |
| Suffix Tree | O(M) | O(N × M × 2) | Overkill for prefixes |

Trie wins because lookup is O(prefix length), independent of total queries.

### Trie Implementation

```java
public class Trie {
    private final TrieNode root;
    
    public Trie() {
        this.root = new TrieNode();
    }
    
    // Insert a query with its score
    public void insert(String query, double score) {
        TrieNode current = root;
        
        for (char c : query.toCharArray()) {
            current = current.getOrCreateChild(c);
            // Update top suggestions at each prefix node
            current.updateTopSuggestions(query, score);
        }
        
        current.setEndOfQuery(true);
        current.setScore(score);
    }
    
    // Get suggestions for a prefix
    public List<Suggestion> getSuggestions(String prefix, int limit) {
        TrieNode node = findNode(prefix);
        
        if (node == null) {
            return Collections.emptyList();
        }
        
        // Return pre-computed top suggestions
        return node.getTopSuggestions().stream()
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    private TrieNode findNode(String prefix) {
        TrieNode current = root;
        
        for (char c : prefix.toCharArray()) {
            current = current.getChild(c);
            if (current == null) {
                return null;  // Prefix not found
            }
        }
        
        return current;
    }
}

public class TrieNode {
    private final Map<Character, TrieNode> children;
    private boolean isEndOfQuery;
    private double score;
    
    // Pre-computed top K suggestions from this prefix
    private final PriorityQueue<Suggestion> topSuggestions;
    private static final int MAX_SUGGESTIONS = 15;
    
    public TrieNode() {
        this.children = new HashMap<>();
        this.topSuggestions = new PriorityQueue<>(
            Comparator.comparingDouble(Suggestion::getScore)
        );
    }
    
    public void updateTopSuggestions(String query, double score) {
        Suggestion suggestion = new Suggestion(query, score);
        
        if (topSuggestions.size() < MAX_SUGGESTIONS) {
            topSuggestions.offer(suggestion);
        } else if (topSuggestions.peek().getScore() < score) {
            topSuggestions.poll();
            topSuggestions.offer(suggestion);
        }
    }
    
    public List<Suggestion> getTopSuggestions() {
        return new ArrayList<>(topSuggestions).stream()
            .sorted(Comparator.comparingDouble(Suggestion::getScore).reversed())
            .collect(Collectors.toList());
    }
}
```

### Memory Optimization

```java
// Optimization 1: Use char array instead of HashMap for children
// Works well for ASCII (a-z, 0-9, space)
public class CompactTrieNode {
    private static final int ALPHABET_SIZE = 37; // a-z + 0-9 + space
    private final CompactTrieNode[] children;
    
    public CompactTrieNode() {
        this.children = new CompactTrieNode[ALPHABET_SIZE];
    }
    
    private int charToIndex(char c) {
        if (c >= 'a' && c <= 'z') return c - 'a';
        if (c >= '0' && c <= '9') return 26 + (c - '0');
        if (c == ' ') return 36;
        return -1;  // Invalid character
    }
}

// Optimization 2: Compress paths with single children
// "hello" becomes one node instead of 5
public class CompressedTrie {
    // Patricia Trie / Radix Tree
    // Stores "hel" + "lo" instead of h-e-l-l-o
}
```

---

## 2. Data Pipeline (Spark)

### Pipeline Architecture

```java
// Spark job for hourly aggregation
public class QueryAggregationJob {
    
    public static void main(String[] args) {
        SparkSession spark = SparkSession.builder()
            .appName("QueryAggregation")
            .getOrCreate();
        
        // Read from Kafka
        Dataset<Row> searchEvents = spark
            .readStream()
            .format("kafka")
            .option("kafka.bootstrap.servers", "kafka:9092")
            .option("subscribe", "search-queries")
            .load()
            .selectExpr("CAST(value AS STRING) as json")
            .select(from_json(col("json"), schema).as("data"))
            .select("data.*");
        
        // Normalize and filter
        Dataset<Row> normalized = searchEvents
            .withColumn("normalized_query", normalizeUDF(col("query")))
            .filter(not(isBlockedUDF(col("normalized_query"))));
        
        // Aggregate by hour
        Dataset<Row> hourlyAggregates = normalized
            .withWatermark("timestamp", "1 hour")
            .groupBy(
                window(col("timestamp"), "1 hour"),
                col("normalized_query")
            )
            .count()
            .withColumnRenamed("count", "hourly_count");
        
        // Write to data warehouse
        hourlyAggregates
            .writeStream()
            .format("parquet")
            .option("path", "s3://typeahead-data/hourly/")
            .option("checkpointLocation", "s3://typeahead-checkpoints/")
            .trigger(Trigger.ProcessingTime("1 hour"))
            .start();
    }
}
```

### Trie Building Job

```java
public class TrieBuilderJob {
    
    public void buildTrie() {
        // Load aggregated data
        Dataset<Row> queries = spark.read()
            .parquet("s3://typeahead-data/aggregated/")
            .orderBy(col("score").desc())
            .limit(5_000_000_000L);  // Top 5 billion
        
        // Build Trie in parallel
        // Partition by first character for parallelism
        Map<Character, Trie> partialTries = queries
            .rdd()
            .mapToPair(row -> {
                String query = row.getString(0);
                double score = row.getDouble(1);
                char firstChar = query.charAt(0);
                return new Tuple2<>(firstChar, new Tuple2<>(query, score));
            })
            .groupByKey()
            .mapValues(this::buildPartialTrie)
            .collectAsMap();
        
        // Merge partial tries
        Trie fullTrie = mergeTries(partialTries);
        
        // Serialize and upload
        byte[] serialized = serializeTrie(fullTrie);
        uploadToS3(serialized, "s3://typeahead-index/v" + version + "/trie.bin");
    }
    
    private Trie buildPartialTrie(Iterable<Tuple2<String, Double>> queries) {
        Trie trie = new Trie();
        for (Tuple2<String, Double> query : queries) {
            trie.insert(query._1, query._2);
        }
        return trie;
    }
}
```

---

## 3. Index Distribution

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

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class TypeaheadMetrics {
    
    private final MeterRegistry registry;
    
    // Latency histogram
    private final Timer suggestionLatency;
    
    // Counters
    private final Counter totalRequests;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    // Gauges
    private final AtomicLong trieSize;
    private final AtomicLong indexVersion;
    
    public TypeaheadMetrics(MeterRegistry registry) {
        this.suggestionLatency = Timer.builder("suggestions.latency")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .register(registry);
        
        this.totalRequests = Counter.builder("suggestions.requests")
            .tag("status", "total")
            .register(registry);
        
        this.trieSize = registry.gauge("trie.size.bytes", new AtomicLong());
    }
    
    public void recordRequest(long latencyMs, boolean cacheHit) {
        suggestionLatency.record(latencyMs, TimeUnit.MILLISECONDS);
        totalRequests.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
    }
}
```

### Health Checks

```java
@Component
public class TypeaheadHealthIndicator implements HealthIndicator {
    
    private final TrieLoader trieLoader;
    private final IndexVersionManager versionManager;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check Trie is loaded
        Trie trie = trieLoader.getCurrentTrie();
        if (trie == null) {
            return Health.down()
                .withDetail("error", "Trie not loaded")
                .build();
        }
        
        details.put("trie_loaded", true);
        details.put("index_version", versionManager.getCurrentVersion());
        details.put("node_count", trie.getNodeCount());
        
        // Check latency
        long testLatency = measureTestQuery();
        details.put("test_latency_ms", testLatency);
        
        if (testLatency > 100) {
            return Health.down()
                .withDetails(details)
                .withDetail("error", "High latency")
                .build();
        }
        
        return Health.up().withDetails(details).build();
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

With 300ms debounce:
```
h (wait 300ms)
he (wait 300ms)
hel (wait 300ms)
hell (wait 300ms)
hello (300ms passed) → request
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

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Data Structure | Trie | Pre-computed suggestions at each node |
| Pipeline | Spark Streaming | Hourly aggregation, 5B queries |
| Coordination | Zookeeper | Version management, watch updates |
| Scoring | Custom algorithm | Popularity + trending + personalization |
| Filtering | Blocklists + regex | Profanity, PII, illegal content |
| Monitoring | Micrometer | P50/P99 latency, cache hit rate |
| Client | Debouncing | 150ms delay to reduce requests |

