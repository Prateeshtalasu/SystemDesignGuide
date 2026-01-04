# Search Engine - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for crawl coordination), caching strategy (multi-layer), and search implementation (inverted index, query processing, ranking).

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka in Search Engine Context?

In a search engine, Kafka serves multiple purposes across the indexing and query paths:

**What problems does Kafka solve here?**

1. **Crawl Coordination**: Buffer downloaded pages for processing
2. **Document Pipeline**: Stream documents through processing stages
3. **Query Logging**: Capture search queries for analytics
4. **Index Updates**: Coordinate incremental index updates

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

| Topic | Purpose | Partitions | Retention |
|-------|---------|------------|-----------|
| `crawled-pages` | Raw HTML from crawlers | 200 | 7 days |
| `processed-documents` | Parsed documents | 100 | 3 days |
| `index-updates` | Incremental index changes | 50 | 24 hours |
| `query-logs` | Search query analytics | 100 | 30 days |

**Crawl Pipeline Flow:**

```
Crawler Pods → crawled-pages → Document Processors → processed-documents → Indexers
      │                                                                        │
      │                                                                        ▼
      └─────────────────────────────────────────────────────────────── Index Shards
```

**Why Kafka for Crawl Pipeline?**

1. **Decoupling**: Crawlers don't wait for processors
2. **Backpressure**: Handles traffic spikes gracefully
3. **Replay**: Can reprocess from any point
4. **Ordering**: Per-domain ordering for politeness

**Producer Configuration (Crawler):**

```java
@Configuration
public class CrawlerKafkaConfig {
    
    @Bean
    public ProducerFactory<String, CrawledPage> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durability settings (crawled pages are valuable)
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Batching for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 131072);  // 128KB batches
        config.put(ProducerConfig.LINGER_MS_CONFIG, 50);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Partition Key Strategy:**

```java
public class CrawlEventPartitioner {
    
    public int partition(CrawledPage page, int numPartitions) {
        // Partition by domain for ordered processing per domain
        String domain = extractDomain(page.getUrl());
        return Math.abs(domain.hashCode()) % numPartitions;
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Crawl Event Flow:**

```
Crawler downloads https://example.com/page1
    ↓
Crawler produces to Kafka:
    Topic: crawled-pages
    Key: "example.com"
    Value: {
      "url": "https://example.com/page1",
      "html": "<html>...</html>",
      "headers": {...},
      "crawled_at": 1705312800000,
      "status_code": 200
    }
    ↓
Document Processor consumes (consumer group: doc-processors)
    ↓
Processor parses HTML, extracts text, metadata
    ↓
Processor produces to Kafka:
    Topic: processed-documents
    Key: "doc_12345"
    Value: {
      "doc_id": "doc_12345",
      "url": "https://example.com/page1",
      "title": "Example Page",
      "content": "...",
      "links": [...],
      "word_count": 1542
    }
    ↓
Indexer consumes (consumer group: indexers)
    ↓
Indexer updates inverted index
```

**Query Logging Flow:**

```java
@Service
public class QueryLogger {
    
    private final KafkaTemplate<String, QueryLogEvent> kafkaTemplate;
    
    @Async  // Fire-and-forget
    public void logQuery(SearchRequest request, SearchResponse response, long latencyMs) {
        QueryLogEvent event = QueryLogEvent.builder()
            .queryId(UUID.randomUUID().toString())
            .queryText(request.getQuery())
            .userId(request.getUserId())
            .resultsCount(response.getTotalResults())
            .latencyMs(latencyMs)
            .timestamp(Instant.now())
            .build();
        
        // Async, no blocking
        kafkaTemplate.send("query-logs", request.getUserId(), event);
    }
}
```

**Failure Scenarios:**

**Q: What if Kafka is down?**

A: For crawl pipeline:
- Crawlers buffer locally (limited)
- Alert fires, ops investigates
- Can replay from S3 backup

For query logging:
- Queries still work (logging is async)
- Lose analytics temporarily
- Acceptable trade-off

**Q: What if consumer lag builds up?**

A: 
- Indexing delays (stale index)
- Add more consumers
- Increase partition count

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis Caching in Search Engine Context?

Search engines use multiple cache layers. Redis serves as the **results cache** - storing pre-computed search results for popular queries.

**What problems does Redis solve here?**

1. **Query Result Caching**: Avoid re-computing popular queries
2. **Spell Dictionary**: Fast lookup for spell checking
3. **Session State**: User preferences, recent searches

### B) OUR USAGE: How We Use Redis Here

**Cache Architecture:**

```
User Query → CDN Cache → Redis Results Cache → Index Servers → Disk
              (30%)           (60%)               (95%)         (100%)
```

**What We Cache:**

| Cache Type | Key | Value | TTL | Size |
|------------|-----|-------|-----|------|
| Query Results | `results:{query_hash}` | Top 100 results | 1 hour | 4 KB |
| Spell Suggestions | `spell:{word}` | Corrections | 24 hours | 100 bytes |
| User Preferences | `user:{id}:prefs` | Settings | 7 days | 500 bytes |
| Trending Queries | `trending:{region}` | Top 100 queries | 5 min | 2 KB |

**Cache Configuration:**

```java
@Configuration
public class RedisCacheConfig {
    
    @Bean
    public RedisTemplate<String, SearchResults> searchResultsTemplate(
            RedisConnectionFactory factory) {
        
        RedisTemplate<String, SearchResults> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Use JSON serialization with compression
        Jackson2JsonRedisSerializer<SearchResults> serializer = 
            new Jackson2JsonRedisSerializer<>(SearchResults.class);
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GzipRedisSerializer(serializer));
        
        return template;
    }
}
```

**Cache Strategy: Cache-Aside with Negative Caching**

```java
@Service
public class SearchCacheService {
    
    private final RedisTemplate<String, SearchResults> redis;
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    private static final Duration NEGATIVE_CACHE_TTL = Duration.ofMinutes(5);
    
    public SearchResults getCachedResults(String query) {
        String key = "results:" + hashQuery(query);
        return redis.opsForValue().get(key);
    }
    
    public void cacheResults(String query, SearchResults results) {
        String key = "results:" + hashQuery(query);
        
        if (results.isEmpty()) {
            // Cache negative results for shorter time
            redis.opsForValue().set(key, results, NEGATIVE_CACHE_TTL);
        } else {
            redis.opsForValue().set(key, results, CACHE_TTL);
        }
    }
    
    private String hashQuery(String query) {
        // Normalize and hash for consistent cache keys
        String normalized = query.toLowerCase().trim().replaceAll("\\s+", " ");
        return DigestUtils.sha256Hex(normalized).substring(0, 16);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Cache Hit Flow:**

```
User searches "best pizza NYC"
    ↓
Query Coordinator receives request
    ↓
Check Redis: GET results:a1b2c3d4e5f6g7h8
    ↓
Cache HIT! Return cached results
    ↓
Total latency: 5ms (vs 200ms without cache)
```

**Cache Miss Flow:**

```
User searches "rare query xyz123"
    ↓
Query Coordinator receives request
    ↓
Check Redis: GET results:x9y8z7...
    ↓
Cache MISS
    ↓
Query all index shards in parallel
    ↓
Merge and rank results (150ms)
    ↓
Cache results: SET results:x9y8z7... (TTL: 5 min for rare queries)
    ↓
Return results to user
```

**Spell Check Cache:**

```java
@Service
public class SpellCheckService {
    
    private final RedisTemplate<String, List<String>> redis;
    
    public List<String> getSuggestions(String word) {
        // Check cache first
        String key = "spell:" + word.toLowerCase();
        List<String> cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return cached;
        }
        
        // Compute suggestions (expensive)
        List<String> suggestions = computeSuggestions(word);
        
        // Cache for 24 hours
        redis.opsForValue().set(key, suggestions, Duration.ofHours(24));
        
        return suggestions;
    }
}
```

**Cache Invalidation Strategy:**

```java
@Service
public class CacheInvalidationService {
    
    // Invalidate when index updates
    @EventListener
    public void onIndexUpdate(IndexUpdateEvent event) {
        // Clear affected query caches
        for (String term : event.getAffectedTerms()) {
            // Use pattern matching to clear related queries
            Set<String> keys = redis.keys("results:*" + term + "*");
            if (!keys.isEmpty()) {
                redis.delete(keys);
            }
        }
    }
    
    // Time-based invalidation for freshness
    @Scheduled(fixedRate = 300000)  // Every 5 minutes
    public void invalidateTrendingCache() {
        redis.delete("trending:*");
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when a cached search result expires and many requests simultaneously try to regenerate it, overwhelming the backend (database or search index).

**Scenario:**
```
T+0s:   Popular query cache expires (e.g., "best pizza")
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit search index simultaneously
T+1s:   Index overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread fetches from search index on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock (another thread may have populated it)
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class SearchCacheService {
    
    private final RedisTemplate<String, SearchResults> redis;
    private final DistributedLock lockService;
    
    public SearchResults getCachedResults(String query) {
        String key = "results:" + hashQuery(query);
        
        // 1. Try cache first
        SearchResults cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:results:" + hashQuery(query);
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Fetch from search index (only one thread does this)
            SearchResults results = executeSearch(query);
            
            // 5. Populate cache with TTL jitter (add ±10% random jitter)
            Duration baseTtl = Duration.ofHours(1);
            long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
            Duration ttl = baseTtl.plusSeconds(jitter);
            
            redis.opsForValue().set(key, results, ttl);
            
            return results;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time

---

## 3. Search (Inverted Index)

### A) CONCEPT: What is the Inverted Index?

The inverted index is the core data structure enabling fast keyword search. It maps terms to documents containing them.

### B) OUR USAGE: How We Implement Search

**Query Processing Pipeline:**

```java
public class QueryProcessor {
    
    private Tokenizer tokenizer;
    private StopWordFilter stopWordFilter;
    private Stemmer stemmer;
    private SpellChecker spellChecker;
    private SynonymExpander synonymExpander;
    
    public ParsedQuery parse(String rawQuery) {
        // 1. Tokenize
        List<String> tokens = tokenizer.tokenize(rawQuery);
        // "best pizza in NYC" → ["best", "pizza", "in", "NYC"]
        
        // 2. Normalize (lowercase, remove accents)
        tokens = tokens.stream()
            .map(String::toLowerCase)
            .map(this::removeAccents)
            .collect(Collectors.toList());
        // ["best", "pizza", "in", "nyc"]
        
        // 3. Remove stop words
        tokens = stopWordFilter.filter(tokens);
        // ["best", "pizza", "nyc"]
        
        // 4. Stem/lemmatize
        tokens = tokens.stream()
            .map(stemmer::stem)
            .collect(Collectors.toList());
        // ["best", "pizza", "nyc"]
        
        // 5. Spell check
        SpellCheckResult spellResult = spellChecker.check(tokens);
        if (spellResult.hasSuggestions()) {
            // Store for "Did you mean?" suggestion
        }
        
        // 6. Synonym expansion (optional)
        Set<String> expandedTerms = synonymExpander.expand(tokens);
        // ["best", "pizza", "nyc", "new york city", "top", "pizzeria"]
        
        return new ParsedQuery(tokens, expandedTerms, spellResult);
    }
}
```

**Boolean Query Execution:**

```java
public class BooleanQueryExecutor {
    
    private InvertedIndex index;
    
    public List<Integer> executeAND(List<String> terms) {
        // Get posting lists sorted by length (shortest first)
        List<PostingList> postings = terms.stream()
            .map(index::getPostingList)
            .sorted(Comparator.comparingInt(PostingList::size))
            .collect(Collectors.toList());
        
        // Start with shortest list
        Set<Integer> result = new HashSet<>(postings.get(0).getDocIds());
        
        // Intersect with remaining lists
        for (int i = 1; i < postings.size(); i++) {
            result.retainAll(postings.get(i).getDocIds());
            
            // Early termination if result is empty
            if (result.isEmpty()) break;
        }
        
        return new ArrayList<>(result);
    }
}
```

**Skip Lists for Fast Intersection:**

```
Query: "new AND york"

Posting list for "new":    [5, 12, 45, 89, 102, 156, 234, 567, 890, ...]
Posting list for "york":   [12, 45, 78, 156, 234, 345, 567, 789, ...]

Without skip list: O(n + m) - scan both lists
With skip list: O(k log n) - skip to next potential match
```

```java
public class SkipListPostingIterator {
    
    private int[] docIds;
    private int[] skipPointers;  // Skip every sqrt(n) elements
    private int currentPos = 0;
    
    public int skipTo(int targetDocId) {
        // Use skip pointers to jump ahead
        while (skipPointers[currentPos / skipInterval] < targetDocId) {
            currentPos += skipInterval;
        }
        
        // Linear scan within block
        while (currentPos < docIds.length && docIds[currentPos] < targetDocId) {
            currentPos++;
        }
        
        return currentPos < docIds.length ? docIds[currentPos] : -1;
    }
}
```

### C) Ranking Algorithm (BM25)

**BM25 Implementation:**

```java
public class BM25Scorer {
    
    private static final double K1 = 1.2;  // Term frequency saturation
    private static final double B = 0.75;  // Length normalization
    
    private long totalDocuments;
    private double averageDocLength;
    private InvertedIndex index;
    
    public double score(int docId, List<String> queryTerms) {
        double score = 0.0;
        int docLength = index.getDocumentLength(docId);
        
        for (String term : queryTerms) {
            double idf = calculateIDF(term);
            int tf = index.getTermFrequency(docId, term);
            
            double tfComponent = (tf * (K1 + 1)) / 
                (tf + K1 * (1 - B + B * (docLength / averageDocLength)));
            
            score += idf * tfComponent;
        }
        
        return score;
    }
    
    private double calculateIDF(String term) {
        int docFreq = index.getDocumentFrequency(term);
        
        // BM25 IDF formula
        return Math.log((totalDocuments - docFreq + 0.5) / (docFreq + 0.5) + 1);
    }
}
```

**Multi-Signal Ranking:**

```java
public class MultiSignalRanker {
    
    private BM25Scorer bm25Scorer;
    private PageRankStore pageRankStore;
    private FreshnessScorer freshnessScorer;
    private SpamDetector spamDetector;
    
    public double calculateFinalScore(int docId, ParsedQuery query) {
        // Text relevance (BM25)
        double textScore = bm25Scorer.score(docId, query.getTerms());
        
        // Normalize to 0-1 range
        textScore = sigmoid(textScore);
        
        // Link authority (PageRank)
        double pageRank = pageRankStore.getScore(docId);
        
        // Freshness (for time-sensitive queries)
        double freshness = 0.5;  // Default neutral
        if (query.isTimeSensitive()) {
            freshness = freshnessScorer.score(docId);
        }
        
        // Field boosts (title match is more important)
        double fieldBoost = calculateFieldBoost(docId, query);
        
        // Spam penalty
        double spamPenalty = spamDetector.getSpamScore(docId);
        
        // Weighted combination
        double finalScore = 
            (textScore * 0.35) +
            (pageRank * 0.25) +
            (freshness * 0.15) +
            (fieldBoost * 0.15) +
            (1 - spamPenalty) * 0.10;
        
        return finalScore;
    }
}
```

**Top-K Selection with Heap:**

```java
public class TopKSelector {
    
    public List<ScoredDocument> selectTopK(Iterator<ScoredDocument> candidates, int k) {
        // Min-heap of size k
        PriorityQueue<ScoredDocument> heap = new PriorityQueue<>(
            k, Comparator.comparingDouble(ScoredDocument::getScore)
        );
        
        while (candidates.hasNext()) {
            ScoredDocument doc = candidates.next();
            
            if (heap.size() < k) {
                heap.offer(doc);
            } else if (doc.getScore() > heap.peek().getScore()) {
                heap.poll();
                heap.offer(doc);
            }
        }
        
        // Convert to sorted list (highest score first)
        List<ScoredDocument> result = new ArrayList<>(heap);
        result.sort(Comparator.comparingDouble(ScoredDocument::getScore).reversed());
        
        return result;
    }
}
```

---

## 4. Web Crawler Design

### Crawler Architecture

```java
public class WebCrawler {
    
    private URLFrontier frontier;
    private RobotsCache robotsCache;
    private DuplicateDetector duplicateDetector;
    private DocumentStore documentStore;
    private ExecutorService crawlerPool;
    
    public void crawl() {
        while (!frontier.isEmpty()) {
            // Get next URL respecting politeness
            CrawlTask task = frontier.getNext();
            
            crawlerPool.submit(() -> {
                try {
                    processCrawlTask(task);
                } catch (Exception e) {
                    handleCrawlError(task, e);
                }
            });
        }
    }
    
    private void processCrawlTask(CrawlTask task) {
        String url = task.getUrl();
        String domain = extractDomain(url);
        
        // 1. Check robots.txt
        RobotsRules rules = robotsCache.getRules(domain);
        if (!rules.isAllowed(url)) {
            log.info("URL blocked by robots.txt: {}", url);
            return;
        }
        
        // 2. Respect crawl delay
        waitForPoliteness(domain, rules.getCrawlDelay());
        
        // 3. Fetch page
        HttpResponse response = httpClient.fetch(url);
        
        if (response.getStatusCode() != 200) {
            handleNon200Response(task, response);
            return;
        }
        
        // 4. Check for duplicates
        String contentHash = hash(response.getBody());
        if (duplicateDetector.isDuplicate(contentHash)) {
            log.info("Duplicate content detected: {}", url);
            return;
        }
        
        // 5. Store document
        documentStore.store(url, response.getBody(), response.getHeaders());
        
        // 6. Extract and queue new URLs
        List<String> links = extractLinks(response.getBody(), url);
        for (String link : links) {
            if (shouldCrawl(link)) {
                frontier.add(link, calculatePriority(link));
            }
        }
    }
}
```

### URL Frontier with Priority

```java
public class URLFrontier {
    
    // Priority queues (1 = highest priority)
    private Map<Integer, Queue<String>> priorityQueues;
    
    // Per-domain queues for politeness
    private Map<String, Queue<String>> domainQueues;
    
    // Last crawl time per domain
    private Map<String, Long> lastCrawlTime;
    
    // Minimum delay between requests to same domain
    private static final long DEFAULT_CRAWL_DELAY_MS = 1000;
    
    public synchronized CrawlTask getNext() {
        // Find domain that's ready to be crawled
        for (int priority = 1; priority <= 10; priority++) {
            Queue<String> queue = priorityQueues.get(priority);
            
            for (String url : queue) {
                String domain = extractDomain(url);
                
                if (isReadyToCrawl(domain)) {
                    queue.remove(url);
                    lastCrawlTime.put(domain, System.currentTimeMillis());
                    return new CrawlTask(url, priority);
                }
            }
        }
        
        return null;  // No URLs ready
    }
}
```

### Duplicate Detection (SimHash)

```java
public class SimHashDuplicateDetector {
    
    private static final int HASH_BITS = 64;
    private Map<Long, Set<String>> hashBuckets;
    
    public boolean isDuplicate(String content) {
        long simhash = computeSimHash(content);
        
        // Check for near-duplicates (hamming distance <= 3)
        for (int i = 0; i < 4; i++) {
            long bucket = simhash ^ (1L << (i * 16));
            Set<String> candidates = hashBuckets.get(bucket);
            
            if (candidates != null) {
                for (String candidateUrl : candidates) {
                    long candidateHash = getStoredHash(candidateUrl);
                    if (hammingDistance(simhash, candidateHash) <= 3) {
                        return true;
                    }
                }
            }
        }
        
        return false;
    }
    
    private int hammingDistance(long a, long b) {
        return Long.bitCount(a ^ b);
    }
}
```

---

## 5. Spell Correction

### Edit Distance with BK-Tree

```java
public class SpellChecker {
    
    private BKTree dictionary;
    private Map<String, Integer> wordFrequencies;
    
    public List<String> suggest(String word, int maxDistance) {
        // Find words within edit distance
        List<String> candidates = dictionary.search(word, maxDistance);
        
        // Rank by frequency
        candidates.sort((a, b) -> 
            wordFrequencies.getOrDefault(b, 0) - wordFrequencies.getOrDefault(a, 0)
        );
        
        return candidates.subList(0, Math.min(5, candidates.size()));
    }
}

public class BKTree {
    
    private BKNode root;
    
    public List<String> search(String word, int maxDistance) {
        List<String> results = new ArrayList<>();
        searchRecursive(root, word, maxDistance, results);
        return results;
    }
    
    private void searchRecursive(BKNode node, String word, int maxDistance, List<String> results) {
        int distance = editDistance(word, node.word);
        
        if (distance <= maxDistance) {
            results.add(node.word);
        }
        
        // Only search children within distance range
        for (int d = distance - maxDistance; d <= distance + maxDistance; d++) {
            if (node.children.containsKey(d)) {
                searchRecursive(node.children.get(d), word, maxDistance, results);
            }
        }
    }
}
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Async Messaging | Kafka | 200 partitions for crawl, acks=all |
| Results Cache | Redis Cluster | 1-hour TTL, 40GB capacity |
| Inverted Index | Custom binary | Delta + VByte compression |
| Query Processing | Pipeline | Tokenize → Stem → Expand |
| Ranking | BM25 + PageRank | k1=1.2, b=0.75, weighted combine |
| Crawler | Distributed | 1s delay, robots.txt respect |
| Spell Check | BK-Tree | Max distance 2 |

