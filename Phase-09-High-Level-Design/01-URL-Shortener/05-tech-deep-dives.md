# URL Shortener - Technology Deep Dives

## Overview

This document covers the technical implementation details for key components: caching strategy, message queue design, database optimizations, and security considerations.

---

## 1. Redis Caching Strategy

### Why Redis for URL Shortener?

Redis is ideal because:

1. **Sub-millisecond latency**: Critical for 50ms redirect target
2. **Simple key-value model**: Perfect for short_code → original_url mapping
3. **Built-in TTL**: Automatic expiration handling
4. **Cluster mode**: Horizontal scaling for high throughput

### Cache Data Structure

```
Key:    url:{short_code}
Value:  {original_url}|{redirect_type}|{expires_at}
TTL:    86400 seconds (24 hours)

Example:
Key:    url:abc123
Value:  https://example.com/long/path|301|1735689600
TTL:    86400
```

**Why pipe-delimited string instead of Redis Hash?**

- Single GET operation vs HGETALL
- Less memory overhead
- Simpler parsing

### Cache Operations

**On URL Creation (Write-Through):**

```java
public void cacheUrl(String shortCode, String originalUrl, int redirectType, Long expiresAt) {
    String value = String.format("%s|%d|%d", originalUrl, redirectType,
                                  expiresAt != null ? expiresAt : 0);

    // Calculate TTL: minimum of 24 hours or time until expiration
    long ttlSeconds = 86400; // 24 hours default
    if (expiresAt != null) {
        long secondsUntilExpiry = expiresAt - System.currentTimeMillis() / 1000;
        ttlSeconds = Math.min(ttlSeconds, secondsUntilExpiry);
    }

    redisTemplate.opsForValue().set("url:" + shortCode, value, ttlSeconds, TimeUnit.SECONDS);
}
```

**On Redirect (Cache-Aside with Fallback):**

```java
public UrlMapping getUrl(String shortCode) {
    // 1. Try cache first
    String cached = redisTemplate.opsForValue().get("url:" + shortCode);
    if (cached != null) {
        return parseUrlMapping(shortCode, cached);
    }

    // 2. Cache miss: query database
    UrlMapping mapping = urlRepository.findByShortCode(shortCode);
    if (mapping == null) {
        // Cache negative result to prevent repeated DB queries
        redisTemplate.opsForValue().set("url:" + shortCode, "NOT_FOUND", 300, TimeUnit.SECONDS);
        return null;
    }

    // 3. Populate cache
    cacheUrl(shortCode, mapping.getOriginalUrl(), mapping.getRedirectType(), mapping.getExpiresAt());

    return mapping;
}
```

### Cache Invalidation

**On URL Deletion:**

```java
public void deleteUrl(String shortCode) {
    // 1. Soft delete in database
    urlRepository.softDelete(shortCode);

    // 2. Invalidate cache
    redisTemplate.delete("url:" + shortCode);

    // 3. Purge CDN cache (async)
    cdnService.purgeAsync(shortCode);
}
```

**On URL Update:**

```java
public void updateUrl(String shortCode, String newOriginalUrl) {
    // 1. Update database
    urlRepository.updateOriginalUrl(shortCode, newOriginalUrl);

    // 2. Delete cache (will be repopulated on next read)
    redisTemplate.delete("url:" + shortCode);

    // 3. Purge CDN
    cdnService.purgeAsync(shortCode);
}
```

### Redis Cluster Configuration

```yaml
# Redis Cluster with 6 nodes (3 primary + 3 replica)
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:6379
        - redis-node-2:6379
        - redis-node-3:6379
        - redis-node-4:6379
        - redis-node-5:6379
        - redis-node-6:6379
      max-redirects: 3
    lettuce:
      pool:
        max-active: 100
        max-idle: 50
        min-idle: 10
```

### Eviction Policy

```
maxmemory 16gb
maxmemory-policy allkeys-lru
```

**Why LRU (Least Recently Used)?**

- Automatically evicts cold URLs
- Keeps hot URLs in cache
- No manual management needed

---

## 2. Kafka for Analytics Events

### Why Kafka?

1. **Decouples redirect from analytics**: Redirect returns immediately
2. **Handles traffic spikes**: Buffers events during peak load
3. **Durability**: Events persist even if analytics service is down
4. **Replay capability**: Reprocess events if needed

### Topic Design

```
Topic: click-events
Partitions: 12
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
```

**Why 12 partitions?**

- Allows up to 12 parallel consumers
- Matches expected consumer count for throughput
- Can increase later if needed (but not decrease)

### Event Schema

```java
public class ClickEvent {
    private String shortCode;       // Partition key
    private long timestamp;         // Event time
    private String ipHash;          // SHA-256 of IP (privacy)
    private String userAgent;       // Raw user agent string
    private String referrer;        // HTTP Referer header
    private String acceptLanguage;  // For geo inference

    // Populated by consumer
    private String countryCode;
    private String city;
    private String deviceType;
    private String browser;
    private String os;
}
```

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, ClickEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Reliability settings
        config.put(ProducerConfig.ACKS_CONFIG, "1");  // Leader ack only (speed over durability)
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);

        // Batching for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);  // Wait 5ms to batch

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Why acks=1 instead of acks=all?**

- Analytics events are not critical (can lose some)
- Prioritize redirect latency over analytics durability
- 3x replication still provides reasonable durability

### Publishing Events (Non-Blocking)

```java
@Service
public class ClickEventPublisher {

    private final KafkaTemplate<String, ClickEvent> kafkaTemplate;

    public void publishClick(String shortCode, HttpServletRequest request) {
        ClickEvent event = ClickEvent.builder()
            .shortCode(shortCode)
            .timestamp(System.currentTimeMillis())
            .ipHash(hashIp(request.getRemoteAddr()))
            .userAgent(request.getHeader("User-Agent"))
            .referrer(request.getHeader("Referer"))
            .acceptLanguage(request.getHeader("Accept-Language"))
            .build();

        // Fire and forget (non-blocking)
        kafkaTemplate.send("click-events", shortCode, event)
            .addCallback(
                result -> log.debug("Click event sent for {}", shortCode),
                ex -> log.warn("Failed to send click event for {}: {}", shortCode, ex.getMessage())
            );
    }

    private String hashIp(String ip) {
        return DigestUtils.sha256Hex(ip + SALT);
    }
}
```

### Consumer Configuration

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, ClickEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "analytics-consumers");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        // Processing settings
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // Manual commit
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);      // Batch size
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 min max processing

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

### Consumer Implementation

```java
@Service
public class ClickEventConsumer {

    private final ClickRepository clickRepository;
    private final GeoIpService geoIpService;
    private final UserAgentParser userAgentParser;

    @KafkaListener(topics = "click-events", groupId = "analytics-consumers")
    public void consume(List<ClickEvent> events, Acknowledgment ack) {
        try {
            // Enrich events
            List<Click> clicks = events.stream()
                .map(this::enrichEvent)
                .collect(Collectors.toList());

            // Batch insert
            clickRepository.batchInsert(clicks);

            // Update real-time counters
            updateClickCounts(clicks);

            // Commit offset
            ack.acknowledge();

        } catch (Exception e) {
            log.error("Failed to process click events", e);
            // Don't acknowledge - will be reprocessed
        }
    }

    private Click enrichEvent(ClickEvent event) {
        // GeoIP lookup
        GeoLocation geo = geoIpService.lookup(event.getIpHash());

        // User agent parsing
        UserAgent ua = userAgentParser.parse(event.getUserAgent());

        return Click.builder()
            .shortCode(event.getShortCode())
            .clickedAt(Instant.ofEpochMilli(event.getTimestamp()))
            .ipHash(event.getIpHash())
            .referrer(event.getReferrer())
            .countryCode(geo.getCountryCode())
            .city(geo.getCity())
            .deviceType(ua.getDeviceType())
            .browser(ua.getBrowser())
            .os(ua.getOs())
            .build();
    }

    private void updateClickCounts(List<Click> clicks) {
        // Group by short_code and increment counters
        Map<String, Long> counts = clicks.stream()
            .collect(Collectors.groupingBy(Click::getShortCode, Collectors.counting()));

        counts.forEach((shortCode, count) -> {
            // Increment in Redis (real-time)
            redisTemplate.opsForValue().increment("clicks:" + shortCode, count);

            // Batch update in PostgreSQL (periodic)
            // This is done by a separate scheduled job
        });
    }
}
```

---

## 3. Database Optimizations

### Connection Pooling

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: 10
      maximum-pool-size: 50
      idle-timeout: 300000 # 5 minutes
      max-lifetime: 1800000 # 30 minutes
      connection-timeout: 20000 # 20 seconds
```

**Why these values?**

- 50 max connections × 10 servers = 500 total connections (within PostgreSQL limits)
- Idle timeout prevents stale connections
- Max lifetime prevents connection issues from long-lived connections

### Query Optimization

**Redirect Query (Most Critical):**

```sql
-- Optimized query with covering index
SELECT original_url, redirect_type
FROM urls
WHERE short_code = $1
  AND deleted_at IS NULL
  AND (expires_at IS NULL OR expires_at > NOW());

-- Supporting index
CREATE INDEX idx_urls_redirect ON urls(short_code)
  INCLUDE (original_url, redirect_type)
  WHERE deleted_at IS NULL;
```

**Why covering index?**

- All needed columns in index
- No table lookup required
- Index-only scan

### Batch Operations

**Batch Click Insert:**

```java
@Repository
public class ClickRepository {

    private static final String INSERT_SQL =
        "INSERT INTO clicks (short_code, clicked_at, ip_hash, referrer, country_code, city, device_type) " +
        "VALUES (?, ?, ?, ?, ?, ?, ?)";

    public void batchInsert(List<Click> clicks) {
        jdbcTemplate.batchUpdate(INSERT_SQL, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Click click = clicks.get(i);
                ps.setString(1, click.getShortCode());
                ps.setTimestamp(2, Timestamp.from(click.getClickedAt()));
                ps.setString(3, click.getIpHash());
                ps.setString(4, click.getReferrer());
                ps.setString(5, click.getCountryCode());
                ps.setString(6, click.getCity());
                ps.setString(7, click.getDeviceType());
            }

            @Override
            public int getBatchSize() {
                return clicks.size();
            }
        });
    }
}
```

### Partition Maintenance

```sql
-- Scheduled job to create future partitions
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    -- Create partition for next month
    partition_date := DATE_TRUNC('month', NOW() + INTERVAL '1 month');
    partition_name := 'clicks_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := partition_date;
    end_date := partition_date + INTERVAL '1 month';

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF clicks
         FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;

-- Drop old partitions (retention: 90 days)
CREATE OR REPLACE FUNCTION drop_old_partitions()
RETURNS void AS $$
DECLARE
    partition_name TEXT;
BEGIN
    FOR partition_name IN
        SELECT tablename FROM pg_tables
        WHERE tablename LIKE 'clicks_%'
        AND tablename < 'clicks_' || TO_CHAR(NOW() - INTERVAL '90 days', 'YYYY_MM')
    LOOP
        EXECUTE format('DROP TABLE IF EXISTS %I', partition_name);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Rate Limiting

### Algorithm: Sliding Window with Redis

```java
@Service
public class RateLimiter {

    private final RedisTemplate<String, String> redisTemplate;

    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String redisKey = "ratelimit:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000L);

        // Use Redis sorted set for sliding window
        redisTemplate.execute(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                operations.multi();

                // Remove old entries
                operations.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);

                // Add current request
                operations.opsForZSet().add(redisKey, String.valueOf(now), now);

                // Count requests in window
                operations.opsForZSet().count(redisKey, windowStart, now);

                // Set expiry
                operations.expire(redisKey, windowSeconds, TimeUnit.SECONDS);

                return operations.exec();
            }
        });

        Long count = redisTemplate.opsForZSet().count(redisKey, windowStart, now);
        return count != null && count <= limit;
    }
}
```

### Rate Limit Tiers

| Tier       | URL Creation | Analytics API | Redirects |
| ---------- | ------------ | ------------- | --------- |
| Anonymous  | 10/hour      | N/A           | Unlimited |
| Free       | 100/hour     | 1000/hour     | Unlimited |
| Pro        | 10,000/hour  | 100,000/hour  | Unlimited |
| Enterprise | Unlimited    | Unlimited     | Unlimited |

### Response Headers

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String apiKey = request.getHeader("Authorization");
        RateLimitResult result = rateLimiter.check(apiKey);

        // Always set headers
        response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(result.getRemaining()));
        response.setHeader("X-RateLimit-Reset", String.valueOf(result.getResetTime()));

        if (!result.isAllowed()) {
            response.setStatus(429);
            response.setHeader("Retry-After", String.valueOf(result.getRetryAfter()));
            return false;
        }

        return true;
    }
}
```

---

## 5. Security Considerations

### Input Validation

```java
@Service
public class UrlValidator {

    private static final Pattern URL_PATTERN = Pattern.compile(
        "^https?://[\\w.-]+(?:\\.[\\w.-]+)+[\\w\\-._~:/?#\\[\\]@!$&'()*+,;=]*$"
    );

    private static final Set<String> BLOCKED_DOMAINS = Set.of(
        "malware.com", "phishing.net"  // Loaded from database
    );

    public ValidationResult validate(String url) {
        // 1. Format validation
        if (!URL_PATTERN.matcher(url).matches()) {
            return ValidationResult.invalid("Invalid URL format");
        }

        // 2. Length check
        if (url.length() > 2048) {
            return ValidationResult.invalid("URL too long (max 2048 characters)");
        }

        // 3. Scheme check
        if (!url.startsWith("http://") && !url.startsWith("https://")) {
            return ValidationResult.invalid("Only HTTP and HTTPS URLs allowed");
        }

        // 4. Domain blocklist check
        String domain = extractDomain(url);
        if (BLOCKED_DOMAINS.contains(domain)) {
            return ValidationResult.invalid("Domain is blocked");
        }

        // 5. Safe browsing API check (async, don't block)
        safeBrowsingService.checkAsync(url);

        return ValidationResult.valid();
    }
}
```

### Custom Alias Validation

```java
public ValidationResult validateAlias(String alias) {
    // 1. Length check
    if (alias.length() < 3 || alias.length() > 30) {
        return ValidationResult.invalid("Alias must be 3-30 characters");
    }

    // 2. Character check
    if (!alias.matches("^[a-zA-Z0-9-]+$")) {
        return ValidationResult.invalid("Only alphanumeric and hyphens allowed");
    }

    // 3. Reserved words
    if (RESERVED_WORDS.contains(alias.toLowerCase())) {
        return ValidationResult.invalid("This alias is reserved");
    }

    // 4. Profanity check
    if (profanityFilter.containsProfanity(alias)) {
        return ValidationResult.invalid("Alias contains prohibited words");
    }

    return ValidationResult.valid();
}
```

### API Key Security

```java
@Service
public class ApiKeyService {

    public String generateApiKey() {
        // Generate 32 random bytes
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);

        // Encode as base64
        String key = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);

        // Prefix for identification
        return "turl_" + key;  // e.g., turl_xK9mN2pQ...
    }

    public void storeApiKey(Long userId, String apiKey) {
        // Store only the hash
        String hash = DigestUtils.sha256Hex(apiKey);
        String prefix = apiKey.substring(0, 12);  // For identification

        ApiKeyEntity entity = ApiKeyEntity.builder()
            .userId(userId)
            .keyHash(hash)
            .keyPrefix(prefix)
            .createdAt(Instant.now())
            .build();

        apiKeyRepository.save(entity);
    }

    public Optional<ApiKeyEntity> validateApiKey(String apiKey) {
        String hash = DigestUtils.sha256Hex(apiKey);
        return apiKeyRepository.findByKeyHash(hash);
    }
}
```

### HTTPS and Headers

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy("default-src 'self'")
                .xssProtection(xss -> xss.block(true))
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)
                )
            )
            .requiresChannel(channel -> channel
                .anyRequest().requiresSecure()  // Force HTTPS
            );

        return http.build();
    }
}
```

---

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class MetricsCollector {

    private final MeterRegistry registry;

    // Counters
    private final Counter urlsCreated;
    private final Counter redirectsTotal;
    private final Counter cacheHits;
    private final Counter cacheMisses;

    // Timers
    private final Timer redirectLatency;
    private final Timer createLatency;

    // Gauges
    private final AtomicLong activeConnections;

    public MetricsCollector(MeterRegistry registry) {
        this.registry = registry;

        this.urlsCreated = Counter.builder("urls.created.total")
            .description("Total URLs created")
            .register(registry);

        this.redirectsTotal = Counter.builder("redirects.total")
            .description("Total redirects served")
            .tag("status", "success")
            .register(registry);

        this.redirectLatency = Timer.builder("redirect.latency")
            .description("Redirect latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordRedirect(String shortCode, long durationMs, boolean cacheHit) {
        redirectsTotal.increment();
        redirectLatency.record(durationMs, TimeUnit.MILLISECONDS);

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
public class SystemHealthIndicator implements HealthIndicator {

    private final RedisTemplate<String, String> redisTemplate;
    private final JdbcTemplate jdbcTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean healthy = true;

        // Check Redis
        try {
            redisTemplate.opsForValue().get("health-check");
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DOWN: " + e.getMessage());
            healthy = false;
        }

        // Check PostgreSQL
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            details.put("postgresql", "UP");
        } catch (Exception e) {
            details.put("postgresql", "DOWN: " + e.getMessage());
            healthy = false;
        }

        // Check Kafka
        try {
            kafkaTemplate.send("health-check", "ping").get(5, TimeUnit.SECONDS);
            details.put("kafka", "UP");
        } catch (Exception e) {
            details.put("kafka", "DOWN: " + e.getMessage());
            // Kafka down is not critical for redirects
        }

        return healthy ? Health.up().withDetails(details).build()
                       : Health.down().withDetails(details).build();
    }
}
```

### Alerting Thresholds

| Metric               | Warning  | Critical  | Action                |
| -------------------- | -------- | --------- | --------------------- |
| Redirect P99 latency | > 80ms   | > 150ms   | Scale app servers     |
| Cache hit rate       | < 85%    | < 70%     | Increase cache size   |
| Error rate           | > 0.1%   | > 1%      | Investigate logs      |
| Database connections | > 80%    | > 95%     | Scale connection pool |
| Kafka consumer lag   | > 10,000 | > 100,000 | Scale consumers       |

---

## Summary

| Component     | Technology              | Key Configuration                    |
| ------------- | ----------------------- | ------------------------------------ |
| Cache         | Redis Cluster           | 16GB, LRU eviction, 24h TTL          |
| Message Queue | Kafka                   | 12 partitions, RF=3, 7-day retention |
| Database      | PostgreSQL              | Connection pooling, covering indexes |
| Rate Limiting | Redis Sorted Sets       | Sliding window algorithm             |
| Security      | Spring Security         | HTTPS, CSP headers, input validation |
| Monitoring    | Micrometer + Prometheus | P50/P95/P99 latencies, counters      |
