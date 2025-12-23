# Search Engine - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a search engine design.

---

## Trade-off Questions

### Q1: Why use an inverted index instead of a forward index?

**Answer:**

A forward index maps documents to terms:
```
Doc1 → [term1, term2, term3]
Doc2 → [term2, term4, term5]
```

An inverted index maps terms to documents:
```
term1 → [Doc1]
term2 → [Doc1, Doc2]
```

**Why inverted is better for search:**

| Aspect          | Forward Index              | Inverted Index            |
| --------------- | -------------------------- | ------------------------- |
| Query time      | O(N × M) scan all docs     | O(k) look up k terms      |
| Storage         | Smaller                    | Larger (with positions)   |
| Updates         | Easy (append to doc)       | Complex (update postings) |
| Use case        | Document retrieval         | Keyword search            |

For a query like "pizza NYC", inverted index requires 2 lookups. Forward index would scan all 10 billion documents.

**Trade-off:** We accept higher storage and complex updates for dramatically faster queries.

---

### Q2: Why shard by document ID instead of by term?

**Answer:**

**Option A: Shard by Document ID (Chosen)**
```
Shard 1: Docs 0-99M (all terms for these docs)
Shard 2: Docs 100-199M (all terms for these docs)
```

**Option B: Shard by Term**
```
Shard 1: Terms A-M (all docs for these terms)
Shard 2: Terms N-Z (all docs for these terms)
```

**Why document-based sharding wins:**

| Factor           | By Document              | By Term                  |
| ---------------- | ------------------------ | ------------------------ |
| Query pattern    | Hit all shards           | Hit 1-2 shards per term  |
| Load balancing   | Even (random doc dist)   | Uneven (popular terms)   |
| Hot spots        | Rare                     | Common ("the", "a")      |
| Index updates    | Localized to one shard   | Distributed across many  |
| Shard failure    | Lose subset of docs      | Lose entire terms        |

**Key insight:** Term-based sharding creates severe hot spots. The term "the" appears in billions of documents - that shard becomes a bottleneck.

---

### Q3: How do you handle the "thundering herd" for trending queries?

**Answer:**

**Problem:** A celebrity tweets, suddenly millions search for the same query.

**Solutions:**

1. **Query Result Cache (Redis)**
   - Cache popular query results
   - TTL: 5 minutes for trending topics
   - Single cache fill, millions of hits

2. **Request Coalescing**
   ```java
   // Only one request to backend for same query
   CompletableFuture<Results> future = inFlightRequests.computeIfAbsent(
       queryHash,
       key -> executeQuery(query).whenComplete((r, e) -> inFlightRequests.remove(key))
   );
   return future;
   ```

3. **Stale-While-Revalidate**
   - Serve stale cached results immediately
   - Refresh cache in background

4. **Rate Limiting per Query**
   - Limit identical queries per second
   - Queue excess requests

**Trade-off:** Slight staleness (seconds) for massive latency reduction.

---

### Q4: Why not use Elasticsearch instead of building custom?

**Answer:**

**Elasticsearch Pros:**
- Ready to use, well-documented
- Built-in clustering, replication
- Rich query DSL
- Active community

**Elasticsearch Cons for Web-Scale Search:**

| Factor           | Elasticsearch            | Custom Solution          |
| ---------------- | ------------------------ | ------------------------ |
| Scale            | Billions of docs hard    | Designed for 10B+        |
| Latency          | 50-100ms typical         | <50ms achievable         |
| Cost             | High memory usage        | Optimized compression    |
| Ranking          | Limited customization    | Full control             |
| Index format     | Generic                  | Optimized for use case   |

**When to use Elasticsearch:**
- Internal search (millions of docs)
- Product search (tens of millions)
- Log search (time-series optimized)

**When to build custom:**
- Web-scale (billions of docs)
- Sub-50ms latency required
- Complex ranking algorithms
- Cost optimization critical

**Google, Bing, etc. all use custom solutions because Elasticsearch doesn't scale to their needs.**

---

### Q5: How do you ensure freshness for news queries?

**Answer:**

**Challenge:** News breaks, users search immediately. Index must reflect current events.

**Multi-tier Indexing Strategy:**

```
┌─────────────────────────────────────────────────────────────┐
│  REAL-TIME INDEX (last 1 hour)                              │
│  - In-memory                                                │
│  - Updated every minute                                     │
│  - ~1M documents                                            │
│  - Queried first for news queries                           │
└─────────────────────────────────────────────────────────────┘
                              +
┌─────────────────────────────────────────────────────────────┐
│  RECENT INDEX (last 24 hours)                               │
│  - SSD-based                                                │
│  - Updated hourly                                           │
│  - ~50M documents                                           │
└─────────────────────────────────────────────────────────────┘
                              +
┌─────────────────────────────────────────────────────────────┐
│  MAIN INDEX (all time)                                      │
│  - Distributed                                              │
│  - Updated daily                                            │
│  - 10B documents                                            │
└─────────────────────────────────────────────────────────────┘
```

**Query Flow:**
1. Detect if query is time-sensitive ("news", "latest", current events)
2. Query real-time index first
3. Merge with recent and main index results
4. Boost freshness in ranking

**Trade-off:** Maintain 3 indexes vs single index. Complexity for freshness.

---

## Scaling Questions

### Q6: How would you scale from 100K to 1M QPS?

**Answer:**

**Current State (100K QPS):**
- 20 query coordinators
- 90 index servers (30 shards × 3 replicas)
- 10 cache servers

**Scaling to 1M QPS (10x):**

1. **Add More Replicas**
   ```
   Current: 3 replicas per shard
   Scaled:  10 replicas per shard
   
   Index servers: 90 → 300
   ```

2. **Increase Cache Layer**
   ```
   Cache servers: 10 → 50
   Cache hit rate target: 60% → 80%
   
   Effective backend QPS: 1M × 0.2 = 200K (2x current)
   ```

3. **Add Query Coordinators**
   ```
   Coordinators: 20 → 100
   ```

4. **Geographic Distribution**
   ```
   Deploy in multiple regions
   Route queries to nearest region
   Reduces cross-region latency
   ```

5. **Index Optimization**
   ```
   More aggressive posting list compression
   Larger in-memory term dictionary
   ```

**Cost Impact:**
- Servers: 240 → 800 (~3.3x)
- Monthly cost: $470K → $1.5M (~3.2x)
- Cost per query stays similar (economies of scale)

---

### Q7: What happens if an index shard goes down?

**Answer:**

**Immediate Impact:**
- Queries to that shard fail
- Results are incomplete (missing ~1% of docs per shard)

**Automatic Recovery:**

1. **Replica Failover (seconds)**
   ```
   Query coordinator detects shard failure
   Routes to replica (we have 3 replicas)
   No user-visible impact if replica healthy
   ```

2. **Health Check**
   ```java
   @Scheduled(fixedRate = 5000)
   public void healthCheck() {
       for (Shard shard : shards) {
           if (!shard.isHealthy()) {
               shardRouter.markUnhealthy(shard);
               alerting.sendAlert("Shard unhealthy: " + shard.getId());
           }
       }
   }
   ```

3. **Graceful Degradation**
   ```
   If all replicas of a shard fail:
   - Return partial results (99% of index)
   - Add header: "X-Partial-Results: true"
   - Log for investigation
   ```

4. **Recovery**
   ```
   - Kubernetes restarts failed pod
   - Shard rebuilds from document store
   - Rejoins cluster when caught up
   - Recovery time: 30 min - 2 hours
   ```

**Key Design Principle:** No single point of failure. Always have replicas.

---

## Failure Scenarios

### Scenario 1: Cache Stampede

**Problem:** Cache expires, thousands of requests hit backend simultaneously.

**Solution:**

```java
public class StampedeProtectedCache {
    
    public Results get(String query) {
        CacheEntry entry = cache.get(query);
        
        if (entry == null) {
            // Cache miss - use lock to prevent stampede
            return computeWithLock(query);
        }
        
        if (entry.isExpiringSoon()) {
            // Proactive refresh in background
            refreshAsync(query);
        }
        
        return entry.getValue();
    }
    
    private Results computeWithLock(String query) {
        Lock lock = locks.get(query);
        
        if (lock.tryLock()) {
            try {
                // Double-check cache
                CacheEntry entry = cache.get(query);
                if (entry != null) return entry.getValue();
                
                // Compute and cache
                Results results = backend.search(query);
                cache.put(query, results);
                return results;
            } finally {
                lock.unlock();
            }
        } else {
            // Another thread is computing, wait for result
            return waitForResult(query);
        }
    }
}
```

---

### Scenario 2: Crawler Overloads a Website

**Problem:** Aggressive crawling brings down a small website.

**Solution:**

```java
public class PoliteCrawler {
    
    // Default: 1 request per second per domain
    private static final long DEFAULT_DELAY_MS = 1000;
    
    // Respect robots.txt Crawl-delay directive
    private long getCrawlDelay(String domain) {
        RobotsRules rules = robotsCache.get(domain);
        if (rules != null && rules.getCrawlDelay() > 0) {
            return rules.getCrawlDelay() * 1000;
        }
        return DEFAULT_DELAY_MS;
    }
    
    // Adaptive delay based on response time
    private void adaptDelay(String domain, long responseTimeMs) {
        if (responseTimeMs > 5000) {
            // Server is slow, back off
            increaseDelay(domain, 2.0);
        } else if (responseTimeMs > 2000) {
            increaseDelay(domain, 1.5);
        }
    }
    
    // Handle 429 (Too Many Requests)
    private void handleRateLimit(String domain, HttpResponse response) {
        String retryAfter = response.getHeader("Retry-After");
        long backoffMs = retryAfter != null 
            ? Long.parseLong(retryAfter) * 1000 
            : 60000;  // Default 1 minute
        
        pauseDomain(domain, backoffMs);
    }
}
```

---

### Scenario 3: Spam Sites Gaming Rankings

**Problem:** SEO spam sites artificially inflate rankings.

**Solution:**

**Multi-layer Defense:**

1. **Content Quality Signals**
   - Thin content detection (low word count)
   - Keyword stuffing detection
   - Duplicate content across domains

2. **Link Analysis**
   - Link farm detection (unnatural link patterns)
   - Anchor text diversity
   - Domain authority

3. **User Behavior Signals**
   - Click-through rate
   - Bounce rate (pogo-sticking)
   - Dwell time

4. **Manual Review**
   - Spam reports from users
   - Quality rater feedback

```java
public class SpamDetector {
    
    public double getSpamScore(Document doc) {
        double score = 0.0;
        
        // Keyword density (stuffing detection)
        double keywordDensity = calculateKeywordDensity(doc);
        if (keywordDensity > 0.05) {  // More than 5% is suspicious
            score += 0.3;
        }
        
        // Thin content
        if (doc.getWordCount() < 100) {
            score += 0.2;
        }
        
        // Link pattern analysis
        if (hasUnnaturalLinkPattern(doc)) {
            score += 0.4;
        }
        
        // Known spam domain
        if (spamDomainList.contains(doc.getDomain())) {
            score += 0.5;
        }
        
        return Math.min(score, 1.0);
    }
}
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand basic components (crawler, index, query processor)
- Know what an inverted index is
- Basic understanding of ranking (TF-IDF)
- Simple capacity estimation

**Sample Answer Quality:**

> "For the index, I'd use an inverted index that maps words to documents. When a user searches, we look up each word and find documents containing all of them. We'd need to shard the index across multiple machines because it's too big for one server."

**Red Flags:**
- Doesn't know what an inverted index is
- Proposes scanning all documents for each query
- No mention of caching

---

### L5 (Mid-Level)

**What's Expected:**
- Deep understanding of inverted index (posting lists, compression)
- Multiple ranking signals (BM25, PageRank, freshness)
- Sharding strategy with justification
- Caching layers and invalidation
- Failure handling

**Sample Answer Quality:**

> "I'd shard by document ID rather than by term because term-based sharding creates hot spots - 'the' appears in billions of documents. Each shard holds a portion of documents with their complete posting lists. For ranking, I'd combine BM25 for text relevance with PageRank for authority, plus freshness signals for time-sensitive queries. The cache has multiple layers: CDN for popular queries, Redis for query results, and in-memory for hot posting lists."

**Red Flags:**
- Can't explain why document-based sharding
- Only mentions one ranking signal
- No discussion of failure modes

---

### L6 (Senior)

**What's Expected:**
- System evolution from MVP to scale
- Complex trade-off analysis
- Real-world examples (how Google/Bing solve this)
- Cost optimization strategies
- Novel solutions to hard problems

**Sample Answer Quality:**

> "Let me walk through how this system would evolve. For MVP, we'd use Elasticsearch with 3 shards - gets us to market fast with millions of documents. As we scale to billions, we'd migrate to a custom index format with SIMD-optimized compression - Google's research shows 4-8x compression with PForDelta. 

> For ranking, we'd start with BM25 but evolve to learned ranking with a two-stage approach: fast first-stage retrieval with BM25, then neural re-ranking on top 1000 candidates. This is how Bing does it.

> The hardest problem is freshness vs completeness. I'd propose a tiered index architecture: real-time index in memory for the last hour, recent index on SSD for the last day, and main index for everything else. News queries hit all three and merge results with freshness boosting.

> For cost optimization at scale, the biggest lever is compression. Moving from VByte to SIMD-BP128 could save 40% on storage and memory, which at our scale is millions per year."

**Red Flags:**
- Can't discuss evolution over time
- No awareness of industry solutions
- Focuses only on technical, not business trade-offs

---

## Common Interviewer Pushbacks

### "Your latency is too high"

**Response:**
"Let me optimize the critical path. First, I'd add a query result cache - 60% of queries repeat. Second, I'd keep the top 100K terms' posting lists in memory - that's 95% of lookups. Third, I'd use early termination in ranking - once we have 1000 high-confidence results, stop scanning. These combined should get us under 100ms p99."

### "How do you handle a new language?"

**Response:**
"Adding a new language requires: (1) language-specific tokenizer - Chinese needs word segmentation, (2) stemmer/lemmatizer for that language, (3) stop word list, (4) spell correction dictionary. We'd create a separate index partition for the language and route queries based on detected language. The ranking algorithm stays the same, but we'd need language-specific training data for any ML components."

### "What if you had unlimited budget?"

**Response:**
"With unlimited budget, I'd: (1) Keep the entire index in memory - eliminates disk I/O latency, (2) Deploy in 20+ regions for <50ms global latency, (3) Use neural ranking for all queries instead of just top candidates, (4) Real-time indexing with sub-minute freshness for all content, (5) Personalized ranking per user. Estimated cost: $50M+/month vs current $500K."

### "What would you cut with 2 weeks to build?"

**Response:**
"For a 2-week MVP: (1) Use Elasticsearch instead of custom index, (2) Simple TF-IDF ranking only, (3) No spell correction, (4) No personalization, (5) Single region deployment, (6) Basic crawler without politeness optimization. This gets us a working search with millions of documents. We'd iterate from there."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | Always explain both options, justify choice        |
| Scaling             | Horizontal scaling, caching, replication           |
| Failures            | Redundancy, graceful degradation, recovery         |
| Optimization        | Caching, compression, early termination            |
| Evolution           | MVP → Production → Scale phases                    |

