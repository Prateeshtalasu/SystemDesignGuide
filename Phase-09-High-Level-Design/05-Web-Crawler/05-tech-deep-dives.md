# Web Crawler - Technology Deep Dives

## Overview

This document covers the technical implementation details for key components: URL frontier management, politeness enforcement, duplicate detection, and distributed coordination.

---

## 1. URL Frontier Deep Dive

### Why is the Frontier Complex?

The URL frontier isn't just a simple queue because:
1. **Politeness**: Can't hammer same domain
2. **Priority**: Important pages should be crawled first
3. **Scale**: Must handle billions of URLs
4. **Distribution**: Multiple crawlers need coordination

### Two-Level Queue Architecture

```java
public class URLFrontier {
    
    // Front queues: priority-based
    private Map<Integer, Queue<String>> frontQueues;  // priority -> domains
    
    // Back queues: domain-based (for politeness)
    private Map<String, Queue<URLEntry>> backQueues;  // domain -> URLs
    
    // Domain metadata
    private Map<String, DomainState> domainStates;
    
    public URLEntry getNext() {
        // 1. Select priority level (weighted random)
        int priority = selectPriority();
        Queue<String> frontQueue = frontQueues.get(priority);
        
        // 2. Find a domain that's ready to crawl
        for (String domain : frontQueue) {
            DomainState state = domainStates.get(domain);
            
            if (state.isReadyToCrawl()) {
                // 3. Get URL from domain's back queue
                Queue<URLEntry> backQueue = backQueues.get(domain);
                URLEntry url = backQueue.poll();
                
                // 4. Update domain state
                state.markCrawled();
                
                // 5. If domain queue empty, remove from front queue
                if (backQueue.isEmpty()) {
                    frontQueue.remove(domain);
                }
                
                return url;
            }
        }
        
        return null;  // No URLs ready
    }
    
    public void add(String url, int priority) {
        String domain = extractDomain(url);
        
        // Add to back queue
        backQueues.computeIfAbsent(domain, k -> new LinkedList<>())
                  .add(new URLEntry(url, priority));
        
        // Ensure domain is in front queue
        frontQueues.computeIfAbsent(priority, k -> new LinkedList<>())
                   .add(domain);
    }
    
    private int selectPriority() {
        // Weighted selection: higher priority = more likely
        // Priority 1: 40%, Priority 2: 30%, Priority 3: 20%, Priority 4: 10%
        double rand = Math.random();
        if (rand < 0.4) return 1;
        if (rand < 0.7) return 2;
        if (rand < 0.9) return 3;
        return 4;
    }
}

public class DomainState {
    private String domain;
    private long lastCrawlTime;
    private int crawlDelayMs;
    private int consecutiveErrors;
    
    public boolean isReadyToCrawl() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastCrawlTime;
        
        // Apply backoff if errors
        int effectiveDelay = crawlDelayMs * (1 + consecutiveErrors);
        
        return elapsed >= effectiveDelay;
    }
    
    public void markCrawled() {
        lastCrawlTime = System.currentTimeMillis();
    }
}
```

### Priority Calculation

```java
public class PriorityCalculator {
    
    public int calculatePriority(String url, URLMetadata metadata) {
        double score = 0.0;
        
        // 1. Domain importance (PageRank-based)
        String domain = extractDomain(url);
        double domainScore = domainRanks.getOrDefault(domain, 0.1);
        score += domainScore * 0.4;
        
        // 2. URL depth (closer to root = higher priority)
        int depth = metadata.getDepth();
        double depthScore = 1.0 / (1 + depth);
        score += depthScore * 0.2;
        
        // 3. URL pattern (news, article = higher)
        if (isNewsUrl(url)) {
            score += 0.2;
        } else if (isArticleUrl(url)) {
            score += 0.1;
        }
        
        // 4. Freshness need
        if (metadata.getLastCrawled() != null) {
            long ageHours = getAgeHours(metadata.getLastCrawled());
            if (ageHours > 168) {  // > 1 week
                score += 0.1;
            }
        } else {
            // Never crawled = high priority
            score += 0.2;
        }
        
        // Convert to priority level (1-10, lower = higher priority)
        if (score > 0.8) return 1;
        if (score > 0.6) return 2;
        if (score > 0.4) return 3;
        if (score > 0.2) return 4;
        return 5;
    }
}
```

---

## 2. Politeness Enforcement

### Why Politeness Matters

1. **Legal**: Violating robots.txt can lead to lawsuits
2. **Ethical**: Aggressive crawling can take down websites
3. **Practical**: Getting blocked defeats the purpose
4. **Reputation**: Being a "good citizen" of the web

### Robots.txt Implementation

```java
public class RobotsManager {
    
    private Map<String, RobotsRules> cache = new ConcurrentHashMap<>();
    private static final long CACHE_TTL_MS = 24 * 60 * 60 * 1000;  // 24 hours
    
    public RobotsRules getRules(String domain) {
        RobotsRules cached = cache.get(domain);
        
        if (cached != null && !cached.isExpired()) {
            return cached;
        }
        
        // Fetch robots.txt
        try {
            String robotsUrl = "https://" + domain + "/robots.txt";
            HttpResponse response = httpClient.fetch(robotsUrl, 10000);
            
            if (response.getStatusCode() == 200) {
                RobotsRules rules = parseRobotsTxt(response.getBody());
                rules.setFetchedAt(System.currentTimeMillis());
                cache.put(domain, rules);
                return rules;
            } else if (response.getStatusCode() == 404) {
                // No robots.txt = allow all
                RobotsRules allowAll = RobotsRules.allowAll();
                cache.put(domain, allowAll);
                return allowAll;
            }
        } catch (Exception e) {
            log.warn("Failed to fetch robots.txt for {}: {}", domain, e.getMessage());
        }
        
        // On error, be conservative
        return RobotsRules.defaultRules();
    }
    
    private RobotsRules parseRobotsTxt(String content) {
        RobotsRules rules = new RobotsRules();
        String currentUserAgent = null;
        
        for (String line : content.split("\n")) {
            line = line.trim();
            if (line.isEmpty() || line.startsWith("#")) continue;
            
            int colonIndex = line.indexOf(':');
            if (colonIndex == -1) continue;
            
            String directive = line.substring(0, colonIndex).trim().toLowerCase();
            String value = line.substring(colonIndex + 1).trim();
            
            switch (directive) {
                case "user-agent":
                    currentUserAgent = value;
                    break;
                case "disallow":
                    if (matchesUserAgent(currentUserAgent)) {
                        rules.addDisallow(value);
                    }
                    break;
                case "allow":
                    if (matchesUserAgent(currentUserAgent)) {
                        rules.addAllow(value);
                    }
                    break;
                case "crawl-delay":
                    if (matchesUserAgent(currentUserAgent)) {
                        rules.setCrawlDelay(Integer.parseInt(value));
                    }
                    break;
                case "sitemap":
                    rules.addSitemap(value);
                    break;
            }
        }
        
        return rules;
    }
    
    private boolean matchesUserAgent(String userAgent) {
        if (userAgent == null) return false;
        return userAgent.equals("*") || 
               OUR_USER_AGENT.toLowerCase().contains(userAgent.toLowerCase());
    }
}
```

### Crawl Delay Enforcement

```java
public class PolitenessEnforcer {
    
    private Map<String, Long> lastCrawlTimes = new ConcurrentHashMap<>();
    private Map<String, Integer> crawlDelays = new ConcurrentHashMap<>();
    
    private static final int DEFAULT_DELAY_MS = 1000;
    private static final int MIN_DELAY_MS = 500;
    private static final int MAX_DELAY_MS = 30000;
    
    public void waitForPoliteness(String domain) throws InterruptedException {
        Long lastCrawl = lastCrawlTimes.get(domain);
        int delay = getCrawlDelay(domain);
        
        if (lastCrawl != null) {
            long elapsed = System.currentTimeMillis() - lastCrawl;
            long waitTime = delay - elapsed;
            
            if (waitTime > 0) {
                Thread.sleep(waitTime);
            }
        }
    }
    
    public void recordCrawl(String domain) {
        lastCrawlTimes.put(domain, System.currentTimeMillis());
    }
    
    public int getCrawlDelay(String domain) {
        Integer delay = crawlDelays.get(domain);
        if (delay != null) {
            return delay;
        }
        
        // Check robots.txt
        RobotsRules rules = robotsManager.getRules(domain);
        if (rules.getCrawlDelay() > 0) {
            delay = Math.min(rules.getCrawlDelay() * 1000, MAX_DELAY_MS);
        } else {
            delay = DEFAULT_DELAY_MS;
        }
        
        crawlDelays.put(domain, delay);
        return delay;
    }
    
    public void increaseDelay(String domain, double factor) {
        int currentDelay = getCrawlDelay(domain);
        int newDelay = Math.min((int)(currentDelay * factor), MAX_DELAY_MS);
        crawlDelays.put(domain, newDelay);
        log.info("Increased crawl delay for {} to {}ms", domain, newDelay);
    }
}
```

---

## 3. Duplicate Detection

### URL-Level Deduplication (Bloom Filter)

```java
public class URLDeduplicator {
    
    private BloomFilter<String> bloomFilter;
    private Set<String> recentUrls;  // For false positive verification
    
    public URLDeduplicator(long expectedUrls, double falsePositiveRate) {
        // 10B URLs, 0.1% FP rate = ~18 GB
        this.bloomFilter = BloomFilter.create(
            Funnels.stringFunnel(StandardCharsets.UTF_8),
            expectedUrls,
            falsePositiveRate
        );
        this.recentUrls = ConcurrentHashMap.newKeySet();
    }
    
    public boolean isDuplicate(String url) {
        String normalizedUrl = normalizeUrl(url);
        String urlHash = hash(normalizedUrl);
        
        // Check bloom filter first (fast)
        if (!bloomFilter.mightContain(urlHash)) {
            return false;
        }
        
        // Bloom filter says maybe - verify with exact check
        // Only for recent URLs (can't store all 10B)
        return recentUrls.contains(urlHash);
    }
    
    public void markSeen(String url) {
        String normalizedUrl = normalizeUrl(url);
        String urlHash = hash(normalizedUrl);
        
        bloomFilter.put(urlHash);
        
        // Keep recent URLs for false positive verification
        recentUrls.add(urlHash);
        
        // Evict old entries (LRU)
        if (recentUrls.size() > 10_000_000) {
            evictOldest();
        }
    }
    
    private String hash(String url) {
        return Hashing.sha256()
                      .hashString(url, StandardCharsets.UTF_8)
                      .toString();
    }
}
```

### Content-Level Deduplication (SimHash)

```java
public class ContentDeduplicator {
    
    private static final int HASH_BITS = 64;
    private static final int SHINGLE_SIZE = 3;  // 3-word shingles
    private static final int MAX_HAMMING_DISTANCE = 3;
    
    // LSH buckets for near-duplicate detection
    private Map<Long, Set<String>> lshBuckets = new ConcurrentHashMap<>();
    
    public long computeSimHash(String content) {
        // Tokenize into words
        String[] words = content.toLowerCase()
                                .replaceAll("[^a-z0-9\\s]", "")
                                .split("\\s+");
        
        // Create shingles
        Set<String> shingles = new HashSet<>();
        for (int i = 0; i <= words.length - SHINGLE_SIZE; i++) {
            String shingle = words[i] + " " + words[i+1] + " " + words[i+2];
            shingles.add(shingle);
        }
        
        // Compute SimHash
        int[] vector = new int[HASH_BITS];
        
        for (String shingle : shingles) {
            long hash = MurmurHash3.hash64(shingle.getBytes());
            
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
    
    public boolean isNearDuplicate(long simhash) {
        // Check LSH buckets
        for (int band = 0; band < 4; band++) {
            long bucketKey = (simhash >>> (band * 16)) & 0xFFFF;
            
            Set<String> bucket = lshBuckets.get(bucketKey);
            if (bucket != null) {
                for (String candidateHash : bucket) {
                    long candidate = Long.parseLong(candidateHash);
                    if (hammingDistance(simhash, candidate) <= MAX_HAMMING_DISTANCE) {
                        return true;
                    }
                }
            }
        }
        
        return false;
    }
    
    public void addToIndex(long simhash, String urlHash) {
        // Add to all 4 LSH buckets
        for (int band = 0; band < 4; band++) {
            long bucketKey = (simhash >>> (band * 16)) & 0xFFFF;
            lshBuckets.computeIfAbsent(bucketKey, k -> ConcurrentHashMap.newKeySet())
                      .add(String.valueOf(simhash));
        }
    }
    
    private int hammingDistance(long a, long b) {
        return Long.bitCount(a ^ b);
    }
}
```

---

## 4. HTTP Fetching

### Async HTTP Client

```java
public class AsyncFetcher {
    
    private final HttpClient httpClient;
    private final ExecutorService executor;
    private final Semaphore concurrencyLimit;
    
    public AsyncFetcher(int maxConcurrency) {
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .build();
        
        this.executor = Executors.newFixedThreadPool(maxConcurrency);
        this.concurrencyLimit = new Semaphore(maxConcurrency);
    }
    
    public CompletableFuture<CrawlResult> fetch(String url) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                concurrencyLimit.acquire();
                return doFetch(url);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return CrawlResult.failed(url, "Interrupted");
            } finally {
                concurrencyLimit.release();
            }
        }, executor);
    }
    
    private CrawlResult doFetch(String url) {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("User-Agent", USER_AGENT)
                .header("Accept", "text/html,application/xhtml+xml")
                .header("Accept-Encoding", "gzip, deflate")
                .header("Accept-Language", "en-US,en;q=0.9")
                .timeout(Duration.ofSeconds(30))
                .GET()
                .build();
            
            long startTime = System.currentTimeMillis();
            HttpResponse<byte[]> response = httpClient.send(request, 
                HttpResponse.BodyHandlers.ofByteArray());
            long duration = System.currentTimeMillis() - startTime;
            
            // Decompress if needed
            byte[] body = response.body();
            String encoding = response.headers()
                .firstValue("Content-Encoding")
                .orElse("");
            
            if (encoding.contains("gzip")) {
                body = decompress(body);
            }
            
            return CrawlResult.success(url, response.statusCode(), body, 
                                       response.headers(), duration);
            
        } catch (HttpTimeoutException e) {
            return CrawlResult.failed(url, "Timeout");
        } catch (IOException e) {
            return CrawlResult.failed(url, "IO Error: " + e.getMessage());
        } catch (Exception e) {
            return CrawlResult.failed(url, "Error: " + e.getMessage());
        }
    }
    
    private byte[] decompress(byte[] compressed) throws IOException {
        try (GZIPInputStream gis = new GZIPInputStream(
                new ByteArrayInputStream(compressed));
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            
            byte[] buffer = new byte[8192];
            int len;
            while ((len = gis.read(buffer)) != -1) {
                bos.write(buffer, 0, len);
            }
            return bos.toByteArray();
        }
    }
}
```

### Retry Logic

```java
public class RetryingFetcher {
    
    private final AsyncFetcher fetcher;
    private static final int MAX_RETRIES = 3;
    private static final int[] BACKOFF_MS = {1000, 5000, 15000};
    
    public CompletableFuture<CrawlResult> fetchWithRetry(String url) {
        return fetchWithRetry(url, 0);
    }
    
    private CompletableFuture<CrawlResult> fetchWithRetry(String url, int attempt) {
        return fetcher.fetch(url)
            .thenCompose(result -> {
                if (result.isSuccess()) {
                    return CompletableFuture.completedFuture(result);
                }
                
                if (shouldRetry(result, attempt)) {
                    return delay(BACKOFF_MS[attempt])
                        .thenCompose(v -> fetchWithRetry(url, attempt + 1));
                }
                
                return CompletableFuture.completedFuture(result);
            });
    }
    
    private boolean shouldRetry(CrawlResult result, int attempt) {
        if (attempt >= MAX_RETRIES) return false;
        
        // Retry on transient errors
        int status = result.getStatusCode();
        if (status == 429 || status == 503 || status == 504) {
            return true;
        }
        
        // Retry on network errors
        String error = result.getError();
        if (error != null && (error.contains("Timeout") || 
                             error.contains("Connection"))) {
            return true;
        }
        
        return false;
    }
    
    private CompletableFuture<Void> delay(long ms) {
        return CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(ms);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

---

## 5. Link Extraction

### HTML Parser

```java
public class LinkExtractor {
    
    public List<ExtractedLink> extractLinks(String html, String baseUrl) {
        List<ExtractedLink> links = new ArrayList<>();
        
        try {
            Document doc = Jsoup.parse(html, baseUrl);
            
            // Extract <a> tags
            for (Element link : doc.select("a[href]")) {
                String href = link.attr("abs:href");
                String anchor = link.text();
                String rel = link.attr("rel");
                
                if (isValidUrl(href) && !isNoFollow(rel)) {
                    links.add(new ExtractedLink(href, anchor, "hyperlink"));
                }
            }
            
            // Extract <link> tags (canonical, alternate)
            for (Element link : doc.select("link[href]")) {
                String href = link.attr("abs:href");
                String rel = link.attr("rel");
                
                if (rel.equals("canonical")) {
                    links.add(new ExtractedLink(href, null, "canonical"));
                } else if (rel.equals("alternate")) {
                    links.add(new ExtractedLink(href, null, "alternate"));
                }
            }
            
            // Extract from meta refresh
            for (Element meta : doc.select("meta[http-equiv=refresh]")) {
                String content = meta.attr("content");
                String url = extractUrlFromRefresh(content);
                if (url != null) {
                    links.add(new ExtractedLink(url, null, "redirect"));
                }
            }
            
        } catch (Exception e) {
            log.warn("Failed to parse HTML from {}: {}", baseUrl, e.getMessage());
        }
        
        return links;
    }
    
    private boolean isValidUrl(String url) {
        if (url == null || url.isEmpty()) return false;
        if (url.startsWith("javascript:")) return false;
        if (url.startsWith("mailto:")) return false;
        if (url.startsWith("#")) return false;
        if (url.startsWith("tel:")) return false;
        
        try {
            URI uri = new URI(url);
            String scheme = uri.getScheme();
            return scheme != null && (scheme.equals("http") || scheme.equals("https"));
        } catch (URISyntaxException e) {
            return false;
        }
    }
    
    private boolean isNoFollow(String rel) {
        return rel != null && rel.toLowerCase().contains("nofollow");
    }
}
```

---

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class CrawlerMetrics {
    
    private final MeterRegistry registry;
    
    // Counters
    private final Counter pagesCrawled;
    private final Counter pagesSuccessful;
    private final Counter pagesFailed;
    private final Counter bytesDownloaded;
    private final Counter urlsDiscovered;
    private final Counter duplicatesFiltered;
    
    // Gauges
    private final AtomicLong frontierSize;
    private final AtomicLong activeCrawls;
    
    // Timers
    private final Timer fetchLatency;
    private final Timer parseLatency;
    
    public CrawlerMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.pagesCrawled = Counter.builder("crawler.pages.total")
            .description("Total pages crawled")
            .register(registry);
        
        this.fetchLatency = Timer.builder("crawler.fetch.latency")
            .description("Page fetch latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.frontierSize = new AtomicLong(0);
        Gauge.builder("crawler.frontier.size", frontierSize, AtomicLong::get)
            .description("URLs in frontier")
            .register(registry);
    }
    
    public void recordCrawl(CrawlResult result) {
        pagesCrawled.increment();
        
        if (result.isSuccess()) {
            pagesSuccessful.increment();
            bytesDownloaded.increment(result.getContentLength());
        } else {
            pagesFailed.increment();
        }
        
        fetchLatency.record(result.getDuration(), TimeUnit.MILLISECONDS);
    }
}
```

### Alerting Thresholds

| Metric               | Warning    | Critical   | Action                |
| -------------------- | ---------- | ---------- | --------------------- |
| Crawl rate           | < 800/s    | < 500/s    | Check crawler health  |
| Success rate         | < 90%      | < 80%      | Investigate failures  |
| Frontier size        | > 500M     | > 1B       | Scale crawlers        |
| P99 fetch latency    | > 20s      | > 30s      | Check network         |
| Duplicate rate       | > 10%      | > 20%      | Check dedup logic     |

---

## Summary

| Component         | Technology/Algorithm          | Key Configuration                |
| ----------------- | ----------------------------- | -------------------------------- |
| URL Frontier      | Two-level queues              | Priority + domain-based          |
| Politeness        | Per-domain delays             | Default 1s, respect robots.txt   |
| URL Dedup         | Bloom filter                  | 18 GB for 10B URLs, 0.1% FP      |
| Content Dedup     | SimHash + LSH                 | 64-bit, hamming ≤ 3              |
| HTTP Fetching     | Async NIO client              | 30s timeout, 3 retries           |
| Link Extraction   | JSoup parser                  | <a>, <link>, meta refresh        |

