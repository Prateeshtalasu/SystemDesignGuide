# Web Crawler - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a web crawler design.

---

## Trade-off Questions

### Q1: Why use a two-level queue (front + back) instead of a single priority queue?

**Answer:**

**Single Priority Queue:**
```
[url1_domain_A, url2_domain_B, url3_domain_A, url4_domain_A, ...]
```

**Problem:** If domain A has high-priority URLs, we might crawl them back-to-back, violating politeness.

**Two-Level Queue:**
```
Front Queue (by priority):
  Priority 1: [domain_A, domain_B, domain_C]
  Priority 2: [domain_D, domain_E]

Back Queue (by domain):
  domain_A: [url1, url3, url4]
  domain_B: [url2]
```

**Benefits:**

| Aspect          | Single Queue           | Two-Level Queue        |
| --------------- | ---------------------- | ---------------------- |
| Politeness      | Hard to enforce        | Natural enforcement    |
| Priority        | Easy                   | Still supported        |
| Domain state    | Need extra tracking    | Built into structure   |
| Memory          | Lower                  | Slightly higher        |

**Key insight:** The two-level structure separates "what priority" from "which domain", allowing both to be optimized independently.

---

### Q2: Why use Bloom filter instead of a hash set for URL deduplication?

**Answer:**

**Hash Set:**
- Exact membership test
- Memory: ~100 bytes per URL (URL string + overhead)
- 10B URLs = 1 TB memory ❌

**Bloom Filter:**
- Probabilistic (false positives possible)
- Memory: ~1.8 bytes per URL at 0.1% FP rate
- 10B URLs = 18 GB memory ✓

**Trade-off Analysis:**

| Factor           | Hash Set               | Bloom Filter           |
| ---------------- | ---------------------- | ---------------------- |
| Memory           | 1 TB                   | 18 GB                  |
| False negatives  | 0%                     | 0%                     |
| False positives  | 0%                     | 0.1%                   |
| Lookup speed     | O(1) average           | O(k) hash functions    |
| Distributed      | Complex                | Easy (merge filters)   |

**Why 0.1% false positives are acceptable:**
- False positive = we skip a URL we haven't seen
- At 1B pages/month, 0.1% = 1M pages missed
- Most are duplicates anyway (same content, different URL)
- We'll likely discover them through other paths

**Mitigation:**
- Use exact check for high-priority URLs
- Periodically rebuild filter to reduce FP rate

---

### Q3: How do you handle infinite crawl traps (e.g., calendar pages)?

**Answer:**

**The Problem:**
```
/calendar/2024/01/01
/calendar/2024/01/02
/calendar/2024/01/03
... (infinite)
```

**Detection Strategies:**

1. **URL Pattern Detection**
   ```java
   // Detect repeating numeric patterns
   if (url.matches(".*/(\\d{4}/\\d{2}/\\d{2}).*")) {
       // Likely a date-based trap
       limitPagesPerPattern(url, 100);
   }
   ```

2. **Depth Limiting**
   ```java
   if (urlMetadata.getDepth() > MAX_DEPTH) {
       skip(url);
   }
   ```

3. **Per-Domain Page Limit**
   ```java
   if (domainStats.getPageCount() > MAX_PAGES_PER_DOMAIN) {
       // Only crawl high-priority pages
       if (url.getPriority() > 2) {
           skip(url);
       }
   }
   ```

4. **URL Similarity Detection**
   ```java
   // If too many similar URLs from same domain
   if (isSimilarToRecentUrls(url, recentUrls)) {
       skip(url);
   }
   ```

5. **Content Similarity**
   ```java
   // If content is nearly identical to previously crawled
   if (simhashSimilarity(content, recentContent) > 0.95) {
       markAsTrap(domain);
   }
   ```

**Practical Limits:**
- Max depth: 15 levels from seed
- Max pages per domain: 1 million
- Max pages per URL pattern: 1,000

---

### Q4: Why partition by domain instead of by URL hash?

**Answer:**

**Option A: Partition by URL Hash**
```
Crawler 1: hash(url) % 15 == 0
Crawler 2: hash(url) % 15 == 1
...
```

**Option B: Partition by Domain Hash (Chosen)**
```
Crawler 1: hash(domain) % 15 == 0  → All example.com URLs
Crawler 2: hash(domain) % 15 == 1  → All another.com URLs
...
```

**Why Domain Partitioning Wins:**

| Factor           | URL Hash               | Domain Hash            |
| ---------------- | ---------------------- | ---------------------- |
| Politeness       | Need coordination      | Local enforcement      |
| robots.txt       | Fetch multiple times   | Fetch once per crawler |
| Connection reuse | Poor                   | Excellent              |
| DNS caching      | Distributed            | Local per crawler      |
| Hot spots        | Even distribution      | Some domains are hot   |

**Key insight:** Crawling is inherently domain-centric due to politeness. Keeping all URLs for a domain on one crawler simplifies everything.

**Handling Hot Spots:**
- Large domains (wikipedia.org) can be sub-partitioned
- Use consistent hashing for rebalancing
- Monitor domain distribution

---

### Q5: Should we use breadth-first or depth-first crawling?

**Answer:**

**Breadth-First (BFS):**
```
Level 0: homepage.com
Level 1: homepage.com/about, homepage.com/products, homepage.com/blog
Level 2: homepage.com/products/item1, homepage.com/blog/post1, ...
```

**Depth-First (DFS):**
```
homepage.com → homepage.com/about → homepage.com/about/team → ...
```

**Best-First (Chosen):**
```
Prioritize by: importance score, freshness need, depth
```

**Comparison:**

| Factor           | BFS                    | DFS                    | Best-First             |
| ---------------- | ---------------------- | ---------------------- | ---------------------- |
| Coverage         | Broad, shallow         | Deep, narrow           | Balanced               |
| Important pages  | Found early            | May miss               | Prioritized            |
| Traps            | Limited impact         | Can get stuck          | Avoided                |
| Memory           | High (wide frontier)   | Low                    | Medium                 |
| Freshness        | Good for homepages     | Poor                   | Configurable           |

**Why Best-First:**
- Homepage and key pages are usually linked from many places → high priority
- Deep pages in traps have low priority → naturally avoided
- Can tune for different use cases (news vs archive)

---

## Scaling Questions

### Q6: How would you scale from 1B to 10B pages/month?

**Answer:**

**Current State (1B pages/month):**
- 15 crawler pods
- 1,000 pages/second
- 3 Kafka brokers

**Scaling to 10B pages/month:**

1. **Linear Scaling of Crawlers**
   ```
   Crawlers: 15 → 150
   Throughput: 1,000/s → 10,000/s
   ```

2. **Kafka Scaling**
   ```
   Brokers: 3 → 10
   Partitions: 15 → 150
   ```

3. **Storage Scaling**
   ```
   Monthly storage: 21 TB → 210 TB
   Consider tiered storage (hot/cold)
   ```

4. **Frontier Scaling**
   ```
   Bloom filter: 18 GB → 180 GB
   Shard across multiple Redis instances
   ```

5. **DNS Scaling**
   ```
   More DNS resolvers
   Longer cache TTL
   Local DNS cache per crawler
   ```

**Bottlenecks to Address:**
- **Politeness limits**: Can't crawl same domain faster
  - Solution: Crawl more domains, not faster per domain
  
- **Coordinator overhead**: More crawlers to manage
  - Solution: Hierarchical coordination, domain sharding

- **Network bandwidth**: 10x more data
  - Solution: Better compression, regional deployment

---

### Q7: What happens if a crawler pod crashes mid-crawl?

**Answer:**

**Immediate Impact:**
- URLs assigned to that crawler stop being crawled
- In-flight requests are lost
- Partial results may be lost

**Recovery Mechanisms:**

1. **Kafka Consumer Rebalancing**
   ```
   - Crashed crawler's partitions reassigned to others
   - URLs re-consumed from last committed offset
   - Some URLs may be re-crawled (idempotent)
   ```

2. **Kubernetes Restart**
   ```
   - Pod automatically restarted
   - Rejoins consumer group
   - Gets partition assignment
   ```

3. **Checkpoint Recovery**
   ```java
   // Periodic checkpoint of crawler state
   @Scheduled(fixedRate = 60000)
   public void checkpoint() {
       CrawlerState state = CrawlerState.builder()
           .domainStates(domainStates)
           .inFlightUrls(inFlightUrls)
           .build();
       
       checkpointStore.save(crawlerId, state);
   }
   
   // On restart
   public void recover() {
       CrawlerState state = checkpointStore.load(crawlerId);
       if (state != null) {
           restoreState(state);
       }
   }
   ```

4. **Idempotent Storage**
   ```java
   // Document storage is idempotent
   // Re-crawling same URL just overwrites
   documentStore.put(urlHash, document);  // Safe to repeat
   ```

**Data Loss Analysis:**
- URLs in Kafka: Not lost (replicated)
- URLs in local queue: Lost, but will be re-discovered
- Downloaded but not stored: Lost, will re-crawl
- Stored documents: Safe

---

## Failure Scenarios

### Scenario 1: DNS Server Overload

**Problem:** DNS queries timing out, crawling stops.

**Solution:**

```java
public class ResilientDNSResolver {
    
    private List<String> dnsServers = Arrays.asList(
        "8.8.8.8",      // Google
        "8.8.4.4",      // Google backup
        "1.1.1.1",      // Cloudflare
        "208.67.222.222" // OpenDNS
    );
    
    private Map<String, CachedDNS> cache = new ConcurrentHashMap<>();
    
    public InetAddress resolve(String domain) {
        // 1. Check cache first
        CachedDNS cached = cache.get(domain);
        if (cached != null && !cached.isExpired()) {
            return cached.getAddress();
        }
        
        // 2. Try each DNS server
        for (String server : dnsServers) {
            try {
                InetAddress address = resolveFrom(domain, server);
                cache.put(domain, new CachedDNS(address, TTL_HOURS));
                return address;
            } catch (Exception e) {
                log.warn("DNS {} failed for {}", server, domain);
            }
        }
        
        // 3. Use stale cache if available
        if (cached != null) {
            log.warn("Using stale DNS for {}", domain);
            return cached.getAddress();
        }
        
        throw new DNSResolutionException(domain);
    }
}
```

---

### Scenario 2: Target Site Blocking Our Crawler

**Problem:** Site returns 403 or blocks our IP.

**Solution:**

```java
public class BlockDetector {
    
    public void handleResponse(String domain, int statusCode) {
        if (statusCode == 403 || statusCode == 429) {
            DomainState state = domainStates.get(domain);
            state.incrementBlockCount();
            
            if (state.getBlockCount() > 3) {
                // Likely blocked
                log.warn("Appears blocked by {}", domain);
                
                // Back off significantly
                state.setCrawlDelay(state.getCrawlDelay() * 10);
                
                // Alert for manual review
                alerting.send("Crawler blocked by " + domain);
                
                // Consider:
                // 1. Reduce crawl rate further
                // 2. Use different User-Agent
                // 3. Skip domain temporarily
            }
        }
    }
}
```

**Prevention:**
- Respect robots.txt strictly
- Use reasonable crawl delays
- Identify as a legitimate crawler
- Provide contact info in User-Agent

---

### Scenario 3: Storage System Failure

**Problem:** S3 is unavailable, can't store crawled pages.

**Solution:**

```java
public class ResilientDocumentStore {
    
    private S3Client s3Client;
    private LocalBuffer localBuffer;
    private CircuitBreaker circuitBreaker;
    
    public void store(String urlHash, byte[] content) {
        if (circuitBreaker.isOpen()) {
            // S3 is down, buffer locally
            localBuffer.add(urlHash, content);
            return;
        }
        
        try {
            s3Client.putObject(bucket, urlHash, content);
            circuitBreaker.recordSuccess();
        } catch (Exception e) {
            circuitBreaker.recordFailure();
            
            if (circuitBreaker.isOpen()) {
                log.error("S3 circuit breaker opened");
                alerting.send("S3 storage failure");
            }
            
            // Buffer locally
            localBuffer.add(urlHash, content);
        }
    }
    
    // Background job to flush buffer when S3 recovers
    @Scheduled(fixedRate = 60000)
    public void flushBuffer() {
        if (circuitBreaker.isClosed() && !localBuffer.isEmpty()) {
            localBuffer.drainTo(this::storeToS3);
        }
    }
}
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand basic crawler components
- Know about robots.txt
- Simple queue-based design
- Basic politeness concept

**Sample Answer Quality:**

> "The crawler would have a queue of URLs to visit. For each URL, we fetch the page, parse it to find more links, and add those to the queue. We need to check robots.txt before crawling and not hit the same site too fast."

**Red Flags:**
- No mention of robots.txt
- Single-threaded design for billions of pages
- No duplicate detection

---

### L5 (Mid-Level)

**What's Expected:**
- Two-level queue architecture
- Bloom filter for deduplication
- Distributed crawler design
- Politeness enforcement details
- Failure handling

**Sample Answer Quality:**

> "I'd use a two-level queue: front queues for priority and back queues for per-domain politeness. URLs are partitioned by domain hash so each crawler handles specific domains - this keeps politeness local. For deduplication, a Bloom filter with 10B capacity at 0.1% false positive rate uses about 18GB. Kafka distributes URLs to crawlers, and if a crawler crashes, Kafka rebalances partitions to surviving crawlers."

**Red Flags:**
- Can't explain why domain-based partitioning
- No understanding of Bloom filter trade-offs
- Missing failure recovery discussion

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- Handling edge cases (traps, blocks)
- Cost optimization
- Real-world examples

**Sample Answer Quality:**

> "Let me walk through how this evolves. For MVP, a single crawler with a Redis queue works for millions of pages. As we scale to billions, we need distributed crawlers partitioned by domain - this is how Googlebot works.

> The hardest problems are: (1) Infinite traps - we detect these through URL pattern analysis and content similarity, limiting to 1000 pages per pattern. (2) Getting blocked - we monitor 403/429 rates per domain and back off aggressively. Google actually publishes their crawler IPs so sites can whitelist them.

> For cost optimization, the biggest lever is not re-crawling unchanged pages. We use If-Modified-Since headers and track change frequencies per URL. A page that hasn't changed in 6 months doesn't need weekly re-crawls - this can reduce crawl volume by 50%+."

**Red Flags:**
- No discussion of edge cases
- Can't explain evolution from simple to complex
- Missing cost/efficiency considerations

---

## Common Interviewer Pushbacks

### "Your crawler is too aggressive"

**Response:**
"You're right to call that out. Politeness is critical. I'd implement: (1) Default 1-second delay between requests to same domain, (2) Strict robots.txt compliance with Crawl-delay directive, (3) Adaptive backoff when we see 429s or slow responses, (4) Per-domain rate limits based on server response times. We should be a good citizen - getting blocked defeats the purpose."

### "How do you handle JavaScript-rendered pages?"

**Response:**
"For MVP, I'd skip JS rendering - it's 10x more expensive. If we need it later, we'd add a headless browser pool (Puppeteer/Playwright) for specific domains. We'd identify JS-heavy sites by detecting empty/minimal content after HTML parsing, then route those to the rendering pool. This is how Google does it - they have a separate 'second wave' indexing for JS content."

### "What if you had unlimited budget?"

**Response:**
"With unlimited budget: (1) Deploy crawlers in every major region to reduce latency, (2) Use headless browsers for all pages to get JS content, (3) Crawl everything daily instead of weekly, (4) Store full page history for change tracking, (5) Real-time crawling triggered by sitemap pings. Estimated cost: $500K+/month vs current $15K."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | Two-level queue, Bloom filter, domain partitioning |
| Scaling             | Linear crawler scaling, Kafka partitions           |
| Failures            | Kafka rebalancing, checkpointing, circuit breakers |
| Edge cases          | Traps, blocks, JS rendering                        |
| Evolution           | MVP → Production → Scale phases                    |

