# Search Engine - Technology Deep Dives

## Overview

This document covers the technical implementation details for key components: inverted index, query processing, ranking algorithms, and crawler design.

---

## 1. Inverted Index Deep Dive

### What is an Inverted Index?

An inverted index maps terms to the documents containing them. It's "inverted" because instead of document → terms, we store term → documents.

```
Forward Index (Document-centric):
  Doc1 → [the, quick, brown, fox]
  Doc2 → [the, lazy, dog]

Inverted Index (Term-centric):
  the   → [Doc1, Doc2]
  quick → [Doc1]
  brown → [Doc1]
  fox   → [Doc1]
  lazy  → [Doc2]
  dog   → [Doc2]
```

### Index Structure

```java
public class InvertedIndex {
    
    // In-memory term dictionary
    private Map<String, TermEntry> termDictionary;
    
    // Memory-mapped posting lists on disk
    private MappedByteBuffer postingLists;
    
    public static class TermEntry {
        String term;
        int documentFrequency;  // Number of docs containing term
        long postingListOffset; // Offset in posting file
        int postingListLength;  // Length in bytes
    }
    
    public static class Posting {
        int docId;
        short termFrequency;    // Times term appears in doc
        int[] positions;        // Word positions for phrase queries
        byte fieldFlags;        // Which fields contain term (title, body, etc.)
    }
}
```

### Posting List Compression

**Delta Encoding:**

```
Original doc IDs:  [100, 150, 152, 200, 201, 205]
Delta encoded:     [100,  50,   2,  48,   1,   4]
```

**Variable Byte Encoding (VByte):**

```java
public class VByteEncoder {
    
    public byte[] encode(int[] numbers) {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        
        for (int num : numbers) {
            while (num >= 128) {
                out.write((num & 0x7F) | 0x80);  // Set continuation bit
                num >>>= 7;
            }
            out.write(num);  // Last byte, no continuation bit
        }
        
        return out.toByteArray();
    }
    
    public int[] decode(byte[] bytes) {
        List<Integer> numbers = new ArrayList<>();
        int num = 0;
        int shift = 0;
        
        for (byte b : bytes) {
            num |= (b & 0x7F) << shift;
            if ((b & 0x80) == 0) {
                numbers.add(num);
                num = 0;
                shift = 0;
            } else {
                shift += 7;
            }
        }
        
        return numbers.stream().mapToInt(i -> i).toArray();
    }
}
```

**Compression Ratios:**

| Technique      | Bits per posting | Compression ratio |
| -------------- | ---------------- | ----------------- |
| Uncompressed   | 32               | 1x                |
| Delta + VByte  | 8-12             | 3-4x              |
| PForDelta      | 4-8              | 4-8x              |
| SIMD-BP128     | 4-6              | 5-8x              |

### Skip Lists for Fast Intersection

For queries with multiple terms, we need to intersect posting lists efficiently.

```
Query: "new AND york"

Posting list for "new":    [5, 12, 45, 89, 102, 156, 234, 567, 890, ...]
Posting list for "york":   [12, 45, 78, 156, 234, 345, 567, 789, ...]

Without skip list: O(n + m) - scan both lists
With skip list: O(k log n) - skip to next potential match
```

**Skip List Structure:**

```
Level 2:  5 ─────────────────────────────> 102 ─────────────────────> 567
Level 1:  5 ────────> 45 ────────> 102 ────────> 234 ────────> 567
Level 0:  5 → 12 → 45 → 89 → 102 → 156 → 234 → 567 → 890 → ...
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

---

## 2. Query Processing

### Query Parsing Pipeline

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

### Boolean Query Execution

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
    
    public List<Integer> executeOR(List<String> terms) {
        Set<Integer> result = new HashSet<>();
        
        for (String term : terms) {
            result.addAll(index.getPostingList(term).getDocIds());
        }
        
        return new ArrayList<>(result);
    }
}
```

### Phrase Query Handling

```java
public class PhraseQueryExecutor {
    
    public List<Integer> executePhraseQuery(String phrase) {
        String[] terms = phrase.split("\\s+");
        
        // Get posting lists with positions
        List<PositionalPostingList> postings = Arrays.stream(terms)
            .map(index::getPositionalPostingList)
            .collect(Collectors.toList());
        
        // Find documents containing all terms
        Set<Integer> candidates = intersectDocIds(postings);
        
        // Check position constraints
        List<Integer> results = new ArrayList<>();
        for (int docId : candidates) {
            if (hasConsecutivePositions(docId, postings)) {
                results.add(docId);
            }
        }
        
        return results;
    }
    
    private boolean hasConsecutivePositions(int docId, List<PositionalPostingList> postings) {
        // Get positions for first term
        int[] firstPositions = postings.get(0).getPositions(docId);
        
        for (int startPos : firstPositions) {
            boolean match = true;
            
            // Check if subsequent terms appear at consecutive positions
            for (int i = 1; i < postings.size(); i++) {
                int[] positions = postings.get(i).getPositions(docId);
                if (!contains(positions, startPos + i)) {
                    match = false;
                    break;
                }
            }
            
            if (match) return true;
        }
        
        return false;
    }
}
```

---

## 3. Ranking Algorithm (BM25)

### BM25 Implementation

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

### Multi-Signal Ranking

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
    
    private double calculateFieldBoost(int docId, ParsedQuery query) {
        double boost = 1.0;
        
        // Title match boost
        if (index.matchesTitle(docId, query.getTerms())) {
            boost *= 2.0;
        }
        
        // URL match boost
        if (index.matchesUrl(docId, query.getTerms())) {
            boost *= 1.5;
        }
        
        // Anchor text match boost
        if (index.matchesAnchorText(docId, query.getTerms())) {
            boost *= 1.3;
        }
        
        return Math.min(boost, 5.0);  // Cap at 5x
    }
    
    private double sigmoid(double x) {
        return 1.0 / (1.0 + Math.exp(-x / 10));
    }
}
```

### Top-K Selection with Heap

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
    
    private boolean isReadyToCrawl(String domain) {
        Long lastCrawl = lastCrawlTime.get(domain);
        if (lastCrawl == null) return true;
        
        long delay = getCrawlDelay(domain);
        return System.currentTimeMillis() - lastCrawl >= delay;
    }
    
    public void add(String url, int priority) {
        // Check if already in frontier
        if (seenUrls.contains(url)) return;
        
        seenUrls.add(url);
        priorityQueues.get(priority).add(url);
        
        String domain = extractDomain(url);
        domainQueues.computeIfAbsent(domain, k -> new LinkedList<>()).add(url);
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
    
    private long computeSimHash(String content) {
        int[] vector = new int[HASH_BITS];
        
        // Tokenize and hash each shingle
        for (String shingle : getShingles(content, 3)) {
            long hash = MurmurHash.hash64(shingle);
            
            for (int i = 0; i < HASH_BITS; i++) {
                if ((hash & (1L << i)) != 0) {
                    vector[i]++;
                } else {
                    vector[i]--;
                }
            }
        }
        
        // Convert to fingerprint
        long fingerprint = 0;
        for (int i = 0; i < HASH_BITS; i++) {
            if (vector[i] > 0) {
                fingerprint |= (1L << i);
            }
        }
        
        return fingerprint;
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
    
    public void add(String word) {
        if (root == null) {
            root = new BKNode(word);
            return;
        }
        
        BKNode current = root;
        int distance = editDistance(word, current.word);
        
        while (current.children.containsKey(distance)) {
            current = current.children.get(distance);
            distance = editDistance(word, current.word);
        }
        
        current.children.put(distance, new BKNode(word));
    }
    
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
    
    private int editDistance(String a, String b) {
        int[][] dp = new int[a.length() + 1][b.length() + 1];
        
        for (int i = 0; i <= a.length(); i++) dp[i][0] = i;
        for (int j = 0; j <= b.length(); j++) dp[0][j] = j;
        
        for (int i = 1; i <= a.length(); i++) {
            for (int j = 1; j <= b.length(); j++) {
                if (a.charAt(i-1) == b.charAt(j-1)) {
                    dp[i][j] = dp[i-1][j-1];
                } else {
                    dp[i][j] = 1 + Math.min(dp[i-1][j-1], 
                                   Math.min(dp[i-1][j], dp[i][j-1]));
                }
            }
        }
        
        return dp[a.length()][b.length()];
    }
}
```

---

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class SearchMetrics {
    
    private final MeterRegistry registry;
    
    // Query metrics
    private final Timer queryLatency;
    private final Counter queriesTotal;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    // Index metrics
    private final Gauge indexSize;
    private final Gauge documentsIndexed;
    
    // Crawler metrics
    private final Counter pagesCrawled;
    private final Counter crawlErrors;
    
    public SearchMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.queryLatency = Timer.builder("search.query.latency")
            .description("Query latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.queriesTotal = Counter.builder("search.queries.total")
            .description("Total queries")
            .register(registry);
    }
    
    public void recordQuery(long durationMs, boolean cacheHit, int resultsCount) {
        queryLatency.record(durationMs, TimeUnit.MILLISECONDS);
        queriesTotal.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
    }
}
```

### Alerting Thresholds

| Metric               | Warning    | Critical   | Action                |
| -------------------- | ---------- | ---------- | --------------------- |
| Query P99 latency    | > 400ms    | > 800ms    | Scale query servers   |
| Cache hit rate       | < 50%      | < 30%      | Increase cache size   |
| Index shard health   | < 95%      | < 90%      | Investigate failures  |
| Crawl rate           | < 3000/s   | < 2000/s   | Check crawler health  |
| Zero-result rate     | > 8%       | > 15%      | Improve index coverage|

---

## Summary

| Component       | Technology/Algorithm          | Key Configuration                |
| --------------- | ----------------------------- | -------------------------------- |
| Inverted Index  | Custom binary + compression   | Delta + VByte, skip lists        |
| Query Processing| Pipeline with expansion       | Tokenize → Stem → Expand         |
| Ranking         | BM25 + PageRank + signals     | k1=1.2, b=0.75, weighted combine |
| Crawler         | Distributed with politeness   | 1s delay, robots.txt respect     |
| Duplicate Detection | SimHash                   | 64-bit, hamming distance ≤ 3     |
| Spell Check     | BK-Tree + edit distance       | Max distance 2                   |

