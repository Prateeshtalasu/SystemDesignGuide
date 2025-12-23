# Pastebin - Technology Deep Dives

## 1. Object Storage (S3) Strategy

### Why S3 for Content?

1. **Scalability**: Automatically scales to petabytes
2. **Cost**: $0.023/GB/month (much cheaper than database storage)
3. **Durability**: 99.999999999% (11 nines)
4. **CDN Integration**: Direct CloudFront integration

### Content Compression

All content is gzip compressed before storage:

```java
@Service
public class ContentStorageService {
    
    private final S3Client s3Client;
    
    public String store(String pasteId, String content) {
        byte[] compressed = compress(content);
        String key = generateKey(pasteId);
        
        s3Client.putObject(PutObjectRequest.builder()
            .bucket("pastebin-content")
            .key(key)
            .contentType("text/plain")
            .contentEncoding("gzip")
            .metadata(Map.of(
                "original-size", String.valueOf(content.length()),
                "paste-id", pasteId
            ))
            .build(),
            RequestBody.fromBytes(compressed));
        
        return key;
    }
    
    private byte[] compress(String content) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             GZIPOutputStream gzip = new GZIPOutputStream(baos)) {
            gzip.write(content.getBytes(StandardCharsets.UTF_8));
            gzip.finish();
            return baos.toByteArray();
        } catch (IOException e) {
            throw new StorageException("Compression failed", e);
        }
    }
    
    private String generateKey(String pasteId) {
        LocalDate now = LocalDate.now();
        return String.format("pastes/%d/%02d/%02d/%s.txt.gz",
            now.getYear(), now.getMonthValue(), now.getDayOfMonth(), pasteId);
    }
}
```

### Compression Ratios

| Content Type | Typical Ratio | Example |
|--------------|---------------|---------|
| Source code | 70-80% | 10KB → 2-3KB |
| Log files | 80-90% | 100KB → 10-20KB |
| JSON/XML | 75-85% | 50KB → 7-12KB |
| Plain text | 60-70% | 20KB → 6-8KB |

### Deduplication Implementation

```java
public class DeduplicationService {
    
    public StorageResult storeWithDedup(String content) {
        String hash = DigestUtils.sha256Hex(content);
        String dedupKey = "deduplicated/" + hash + ".txt.gz";
        
        // Check if content already exists
        if (s3Client.doesObjectExist("pastebin-content", dedupKey)) {
            return StorageResult.deduplicated(dedupKey);
        }
        
        // Store new content
        byte[] compressed = compress(content);
        s3Client.putObject(/* ... */);
        
        return StorageResult.stored(dedupKey, compressed.length);
    }
}
```

---

## 2. Redis Caching Strategy

### Cache Keys and TTLs

| Key Pattern | Value | TTL | Purpose |
|-------------|-------|-----|---------|
| `meta:{id}` | JSON metadata | 1 hour | Fast existence check |
| `content:{id}` | Compressed text | 15 min | Hot content cache |
| `rate:{ip}:{action}` | Counter | 1 hour | Rate limiting |
| `user:{id}:pastes` | List of IDs | 5 min | User's paste list |

### Cache-Aside Pattern

```java
@Service
public class PasteCacheService {
    
    private final RedisTemplate<String, byte[]> redisTemplate;
    private final ContentStorageService storageService;
    
    public byte[] getContent(String pasteId, String storageKey) {
        String cacheKey = "content:" + pasteId;
        
        // Try cache first
        byte[] cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss: fetch from S3
        byte[] content = storageService.fetch(storageKey);
        
        // Only cache if small enough (< 1MB)
        if (content.length < 1024 * 1024) {
            redisTemplate.opsForValue().set(cacheKey, content, 
                Duration.ofMinutes(15));
        }
        
        return content;
    }
}
```

### Rate Limiting with Redis

```java
@Service
public class RateLimiter {
    
    private final StringRedisTemplate redis;
    
    public RateLimitResult checkLimit(String identifier, String action, int limit) {
        String key = String.format("rate:%s:%s", identifier, action);
        String windowKey = key + ":" + (System.currentTimeMillis() / 3600000); // Hour window
        
        Long count = redis.opsForValue().increment(windowKey);
        if (count == 1) {
            redis.expire(windowKey, Duration.ofHours(1));
        }
        
        return RateLimitResult.builder()
            .allowed(count <= limit)
            .current(count.intValue())
            .limit(limit)
            .resetAt(getNextHour())
            .build();
    }
}
```

---

## 3. Content Delivery (CDN)

### CloudFront Configuration

```yaml
# CloudFront Distribution Config
Origins:
  - DomainName: pastebin-content.s3.amazonaws.com
    Id: S3Origin
    S3OriginConfig:
      OriginAccessIdentity: origin-access-identity/cloudfront/XXXXX

  - DomainName: api.pastebin.com
    Id: APIOrigin
    CustomOriginConfig:
      HTTPSPort: 443
      OriginProtocolPolicy: https-only

CacheBehaviors:
  - PathPattern: /raw/*
    TargetOriginId: S3Origin
    ViewerProtocolPolicy: redirect-to-https
    CachePolicyId: CachingOptimized
    TTL:
      DefaultTTL: 3600
      MaxTTL: 86400
    Compress: true

  - PathPattern: /v1/*
    TargetOriginId: APIOrigin
    ViewerProtocolPolicy: redirect-to-https
    CachePolicyId: CachingDisabled
```

### Cache Invalidation

```java
@Service
public class CdnInvalidationService {
    
    private final CloudFrontClient cloudFront;
    
    public void invalidatePaste(String pasteId) {
        cloudFront.createInvalidation(CreateInvalidationRequest.builder()
            .distributionId("XXXXX")
            .invalidationBatch(InvalidationBatch.builder()
                .paths(Paths.builder()
                    .items("/raw/" + pasteId)
                    .quantity(1)
                    .build())
                .callerReference(UUID.randomUUID().toString())
                .build())
            .build());
    }
}
```

---

## 4. Security Implementation

### XSS Prevention

```java
@Component
public class ContentSanitizer {
    
    public String sanitizeForDisplay(String content) {
        // HTML encode for safe display
        return HtmlUtils.htmlEscape(content);
    }
    
    public boolean containsMaliciousPatterns(String content) {
        // Check for common attack patterns
        List<Pattern> patterns = List.of(
            Pattern.compile("<script[^>]*>", Pattern.CASE_INSENSITIVE),
            Pattern.compile("javascript:", Pattern.CASE_INSENSITIVE),
            Pattern.compile("on\\w+\\s*=", Pattern.CASE_INSENSITIVE)
        );
        
        return patterns.stream()
            .anyMatch(p -> p.matcher(content).find());
    }
}
```

### Password Protection

```java
@Service
public class PastePasswordService {
    
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
    
    public String hashPassword(String password) {
        return encoder.encode(password);
    }
    
    public boolean verifyPassword(String password, String hash) {
        return encoder.matches(password, hash);
    }
}
```

### Content Security Headers

```java
@Configuration
public class SecurityHeadersConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new HandlerInterceptor() {
            @Override
            public void postHandle(HttpServletRequest request, 
                                   HttpServletResponse response,
                                   Object handler, ModelAndView modelAndView) {
                response.setHeader("Content-Security-Policy", 
                    "default-src 'self'; script-src 'self' 'unsafe-inline'");
                response.setHeader("X-Content-Type-Options", "nosniff");
                response.setHeader("X-Frame-Options", "DENY");
                response.setHeader("X-XSS-Protection", "1; mode=block");
            }
        });
    }
}
```

---

## 5. Syntax Highlighting

### Server-Side Highlighting (Optional)

```java
@Service
public class SyntaxHighlightService {
    
    public String highlight(String content, String language) {
        // Use Pygments or similar library
        ProcessBuilder pb = new ProcessBuilder(
            "pygmentize",
            "-l", language,
            "-f", "html",
            "-O", "nowrap=true"
        );
        
        Process process = pb.start();
        process.getOutputStream().write(content.getBytes());
        process.getOutputStream().close();
        
        return new String(process.getInputStream().readAllBytes());
    }
}
```

### Client-Side Highlighting (Preferred)

Use libraries like Prism.js or highlight.js in the frontend:

```html
<pre><code class="language-python" id="paste-content"></code></pre>

<script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.29.0/prism.min.js"></script>
<script>
  fetch('/raw/' + pasteId)
    .then(r => r.text())
    .then(content => {
      document.getElementById('paste-content').textContent = content;
      Prism.highlightAll();
    });
</script>
```

---

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class PasteMetrics {
    
    private final MeterRegistry registry;
    
    // Counters
    private final Counter pastesCreated;
    private final Counter pastesViewed;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    // Histograms
    private final Timer createLatency;
    private final Timer viewLatency;
    private final DistributionSummary pasteSize;
    
    public PasteMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.pastesCreated = Counter.builder("pastes.created")
            .tag("visibility", "all")
            .register(registry);
        
        this.pasteSize = DistributionSummary.builder("paste.size.bytes")
            .publishPercentiles(0.5, 0.9, 0.99)
            .register(registry);
    }
    
    public void recordCreate(Paste paste) {
        pastesCreated.increment();
        pasteSize.record(paste.getSizeBytes());
    }
}
```

### Health Checks

```java
@Component
public class PastebinHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check PostgreSQL
        details.put("database", checkDatabase());
        
        // Check Redis
        details.put("cache", checkRedis());
        
        // Check S3
        details.put("storage", checkS3());
        
        boolean allHealthy = details.values().stream()
            .allMatch(v -> "UP".equals(v));
        
        return allHealthy ? Health.up().withDetails(details).build()
                          : Health.down().withDetails(details).build();
    }
}
```

---

## 7. Cleanup Worker Implementation

```java
@Component
public class ExpirationCleanupJob {
    
    private final PasteRepository pasteRepository;
    private final S3Client s3Client;
    private final RedisTemplate<String, Object> redis;
    
    @Scheduled(cron = "0 0 * * * *") // Every hour
    public void cleanupExpiredPastes() {
        log.info("Starting expiration cleanup");
        
        int totalDeleted = 0;
        List<Paste> expired;
        
        do {
            // Fetch batch of expired pastes
            expired = pasteRepository.findExpired(
                Instant.now(), 
                PageRequest.of(0, 1000)
            );
            
            if (expired.isEmpty()) break;
            
            // Collect storage keys for batch S3 delete
            List<String> storageKeys = expired.stream()
                .map(Paste::getStorageKey)
                .distinct()
                .collect(Collectors.toList());
            
            // Soft delete in database
            List<String> ids = expired.stream()
                .map(Paste::getId)
                .collect(Collectors.toList());
            pasteRepository.softDeleteBatch(ids);
            
            // Invalidate cache
            ids.forEach(id -> {
                redis.delete("meta:" + id);
                redis.delete("content:" + id);
            });
            
            // Delete from S3 (batch)
            deleteFromS3Batch(storageKeys);
            
            totalDeleted += expired.size();
            
        } while (expired.size() == 1000);
        
        log.info("Cleanup complete. Deleted {} pastes", totalDeleted);
    }
    
    private void deleteFromS3Batch(List<String> keys) {
        // S3 supports up to 1000 keys per delete request
        List<ObjectIdentifier> objects = keys.stream()
            .map(k -> ObjectIdentifier.builder().key(k).build())
            .collect(Collectors.toList());
        
        s3Client.deleteObjects(DeleteObjectsRequest.builder()
            .bucket("pastebin-content")
            .delete(Delete.builder().objects(objects).build())
            .build());
    }
}
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Content Storage | S3 | Gzip compression, lifecycle policies |
| Cache | Redis Cluster | 15-min TTL for content, 1-hour for metadata |
| CDN | CloudFront | 1-hour TTL, gzip enabled |
| Syntax Highlighting | Prism.js (client) | 100+ languages |
| Rate Limiting | Redis | Sliding window, per-IP and per-user |
| Cleanup | Spring Scheduler | Hourly batch processing |
| Monitoring | Micrometer | Latency percentiles, cache hit rates |

