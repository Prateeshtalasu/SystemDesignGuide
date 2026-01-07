# Caching Strategies for Performance

## 0️⃣ Prerequisites

- Understanding of caching fundamentals (covered in Phase 4)
- Knowledge of HTTP headers (Cache-Control, ETag, Last-Modified)
- Understanding of CDN concepts (covered in Phase 2)
- Basic familiarity with Redis (covered in Phase 4)

**Quick refresher**: Caching stores frequently accessed data in fast storage (memory) to avoid expensive operations (database queries, API calls). HTTP caching uses browser and CDN caches to store responses, reducing server load and improving response times. This topic focuses on performance-specific caching strategies beyond basic caching concepts.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

Without proper caching strategies:

1. **Repeated expensive operations**: Same database query executed thousands of times
2. **Slow response times**: Every request hits database/API, adding latency
3. **High server load**: Database and services overwhelmed with redundant requests
4. **Wasted bandwidth**: Same data transferred repeatedly
5. **Poor mobile experience**: Slow connections make every request painful

### What Systems Look Like Without Performance Caching

Without performance caching:

- **Product page loads in 2 seconds** because it queries database every time
- **Database CPU at 100%** handling repeated identical queries
- **API calls take 500ms** even for static data
- **Mobile users wait 10+ seconds** for pages to load
- **Bandwidth costs are high** transferring same data repeatedly

### Real Examples

**Example 1**: E-commerce product page. Without caching, every page view queries database for product details, inventory, reviews, recommendations. With caching, data cached for 5 minutes. Result: Page loads 10x faster (200ms vs 2 seconds), database load reduced by 95%.

**Example 2**: News website homepage. Without caching, every visitor triggers database queries for articles. With HTTP caching, page cached at CDN. Result: 99% of requests served from CDN (instant), origin server handles 1% of traffic.

**Example 3**: API returning user profile. Without caching, every API call queries database. With application-level caching, profile cached for 1 hour. Result: API responds in 5ms (cache hit) vs 50ms (database query), 20x faster.

## 2️⃣ Intuition and Mental Model

**Think of caching like a well-organized library:**

Without caching, every time someone wants a popular book, the librarian walks to the storage room, finds it, and brings it back. With caching:
- **Popular books on display shelf** (HTTP cache): Most requested books right at front, instant access
- **Librarian's desk cache** (application cache): Recently requested books kept nearby
- **Storage room** (database): All books stored, accessed when cache misses

**The mental model**: Caching creates layers of fast storage. Data moves from slow storage (database) to fast storage (memory) based on access patterns. The most frequently accessed data stays in the fastest cache (closest to users), less frequent data in slower caches, and everything in the database.

**Another analogy**: Think of caching like a restaurant's mise en place. The chef (server) preps frequently used ingredients (caches data) before service. When an order comes in, ingredients are ready (cache hit) instead of having to fetch them from storage (database query). Popular dishes (hot data) are prepped more, less popular (cold data) less.

## 3️⃣ How it works internally

### Caching Layers

#### 1. Browser Cache (Client-Side)

**Location**: User's browser
**What it caches**: HTTP responses (HTML, CSS, JS, images)
**How it works**: Browser stores responses, serves from cache on subsequent requests

**Cache headers**:
```
Cache-Control: max-age=3600  # Cache for 1 hour
ETag: "abc123"               # Content version identifier
Last-Modified: Mon, 01 Jan 2024 12:00:00 GMT
```

**Cache validation**:
- **Strong validation**: ETag or Last-Modified
- **Weak validation**: Expires or max-age
- **Revalidation**: 304 Not Modified if content unchanged

#### 2. CDN Cache (Edge Cache)

**Location**: CDN edge servers (geographically distributed)
**What it caches**: Static assets, API responses
**How it works**: CDN caches responses, serves from nearest edge location

**Benefits**:
- Reduced latency (served from nearby server)
- Reduced origin load (most requests don't reach origin)
- Global distribution

**Cache invalidation**:
- Time-based (TTL)
- Manual purging
- Cache tags (invalidate related content)

#### 3. Application Cache (Server-Side)

**Location**: Application server memory (Redis, Memcached)
**What it caches**: Database queries, API responses, computed data
**How it works**: Application checks cache before expensive operations

**Cache patterns**:
- **Cache-aside**: Application manages cache
- **Write-through**: Write to cache and database simultaneously
- **Write-behind**: Write to cache, async write to database

#### 4. Database Query Cache

**Location**: Database server
**What it caches**: Query results
**How it works**: Database caches frequent queries internally

**Benefits**: Faster repeated queries
**Limitations**: Limited cache size, invalidated on data changes

### HTTP Caching Headers

#### Cache-Control Directives

**max-age**: How long to cache (seconds)
```
Cache-Control: max-age=3600  # Cache for 1 hour
```

**no-cache**: Must revalidate before using cached response
```
Cache-Control: no-cache  # Always check with server
```

**no-store**: Don't cache at all
```
Cache-Control: no-store  # Never cache
```

**private**: Only cache in browser, not in shared caches (CDN)
```
Cache-Control: private  # Browser cache only
```

**public**: Can cache in shared caches (CDN)
```
Cache-Control: public, max-age=3600  # CDN can cache
```

**must-revalidate**: Must revalidate expired cache
```
Cache-Control: must-revalidate  # Revalidate if expired
```

#### ETag and Last-Modified

**ETag**: Content version identifier
```
ETag: "abc123"
If-None-Match: "abc123"  # Client sends in request
```

**Last-Modified**: When content was last changed
```
Last-Modified: Mon, 01 Jan 2024 12:00:00 GMT
If-Modified-Since: Mon, 01 Jan 2024 12:00:00 GMT  # Client sends
```

**304 Not Modified**: Server responds with 304 if content unchanged
- Saves bandwidth (no response body)
- Faster than full response

## 4️⃣ Simulation-first explanation

### Scenario: Product Page with HTTP Caching

**Without caching**:
```
User 1 requests product page:
  Browser → CDN → Origin Server → Database
  Response: 2 seconds

User 2 requests same product page:
  Browser → CDN → Origin Server → Database
  Response: 2 seconds (repeated work)

User 3 requests same product page:
  Browser → CDN → Origin Server → Database
  Response: 2 seconds (repeated work)
```

**With HTTP caching**:
```
User 1 requests product page:
  Browser → CDN → Origin Server → Database
  Response: 2 seconds
  Headers: Cache-Control: public, max-age=300
  CDN caches for 5 minutes

User 2 requests same product page:
  Browser → CDN (cache hit)
  Response: 50ms (served from CDN)

User 3 requests same product page:
  Browser → CDN (cache hit)
  Response: 50ms (served from CDN)
```

**Performance improvement**: 2 seconds → 50ms (40x faster for cached requests)

### Scenario: API Response with Application Cache

**Without caching**:
```
Request 1: GET /api/user/123
  Application → Database (50ms)
  Response: User data

Request 2: GET /api/user/123
  Application → Database (50ms)
  Response: User data (same query again)

Request 3: GET /api/user/123
  Application → Database (50ms)
  Response: User data (same query again)
```

**With application cache**:
```
Request 1: GET /api/user/123
  Application → Check cache (miss)
  Application → Database (50ms)
  Application → Store in cache (key: user:123, TTL: 1 hour)
  Response: User data

Request 2: GET /api/user/123
  Application → Check cache (hit)
  Response: User data (5ms, from cache)

Request 3: GET /api/user/123
  Application → Check cache (hit)
  Response: User data (5ms, from cache)
```

**Performance improvement**: 50ms → 5ms (10x faster for cached requests)

### Scenario: Cache Warming

**Cold cache**: No data in cache, all requests hit database
```
Request 1: Cache miss → Database (50ms)
Request 2: Cache miss → Database (50ms)
Request 3: Cache miss → Database (50ms)
```

**Warm cache**: Popular data pre-loaded into cache
```
Startup: Pre-load popular products into cache
  Cache: product:1, product:2, product:3, ... (pre-loaded)

Request 1: Cache hit (5ms)
Request 2: Cache hit (5ms)
Request 3: Cache hit (5ms)
```

**Result**: Better performance from start, no cold start penalty

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. Multi-Layer Caching

**Netflix's caching strategy**:
- **Browser cache**: Static assets (JS, CSS, images)
- **CDN cache**: Video metadata, thumbnails
- **Application cache**: User preferences, recommendations
- **Database cache**: Frequent queries

#### 2. Cache Warming Strategies

**E-commerce sites**: Pre-warm cache before traffic spikes
- Black Friday: Pre-load popular products
- Product launches: Pre-load new product data
- Scheduled jobs: Pre-load trending content

#### 3. Cache Invalidation Strategies

**Cache Invalidation Methods**:

1. **Time-Based Expiration (TTL)**:
   - Set expiration time when caching
   - Cache automatically expires
   - Simple but may serve stale data temporarily
   - Best for: Data that changes infrequently

2. **Event-Based Invalidation**:
   - Invalidate cache when data changes
   - Immediate freshness guarantee
   - Requires event system
   - Best for: Data that must be fresh

3. **Version-Based Invalidation**:
   - Include version in cache key
   - Update version when data changes
   - Old versions naturally expire
   - Best for: Frequently changing data

4. **Tag-Based Invalidation**:
   - Tag cached items with categories
   - Invalidate all items with a tag
   - Efficient for related content
   - Best for: Content with relationships

5. **Manual Purging**:
   - Admin-triggered cache clearing
   - For emergencies or maintenance
   - Best for: Administrative operations

**News sites**: Invalidate cache on content updates
- Article published: Invalidate homepage cache
- Article updated: Invalidate article cache
- Use cache tags for related content invalidation

#### 4. HTTP Caching for APIs

**REST APIs**: Use HTTP caching headers
- Public data: `Cache-Control: public, max-age=3600`
- User-specific: `Cache-Control: private, max-age=300`
- Dynamic data: `Cache-Control: no-cache`

### Production Workflow

**Step 1: Identify cacheable content**
- Static assets (always cache)
- Dynamic but stable content (cache with TTL)
- User-specific content (private cache)

**Step 2: Configure caching**
- Set appropriate cache headers
- Configure CDN caching
- Set up application-level caching

**Step 3: Implement cache invalidation**
- Time-based expiration
- Event-based invalidation
- Manual purging when needed

**Step 4: Monitor and tune**
- Track cache hit rates
- Monitor response times
- Adjust TTLs based on data freshness requirements

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: HTTP Caching Headers

**Controller with cache headers**:
```java
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        Product product = productService.getProduct(id);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES)
                .cachePublic()
                .mustRevalidate())
            .eTag(product.getVersion().toString())
            .lastModified(product.getLastModified())
            .body(product);
    }
    
    @GetMapping("/api/user/profile")
    public ResponseEntity<UserProfile> getUserProfile(@AuthenticationPrincipal User user) {
        UserProfile profile = userService.getProfile(user.getId());
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(10, TimeUnit.MINUTES)
                .cachePrivate())  // User-specific, browser cache only
            .body(profile);
    }
}
```

**Conditional requests (ETag)**:
```java
@GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(
            @PathVariable Long id,
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {
        
        Product product = productService.getProduct(id);
        String etag = product.getVersion().toString();
        
        if (etag.equals(ifNoneMatch)) {
            // Content unchanged, return 304
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                .eTag(etag)
                .build();
        }
        
        return ResponseEntity.ok()
            .eTag(etag)
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
            .body(product);
    }
```

#### Example 2: Spring Cache Abstraction

**Configuration**:
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withCacheConfiguration("products", 
                config.entryTtl(Duration.ofMinutes(5)))
            .withCacheConfiguration("users", 
                config.entryTtl(Duration.ofHours(1)))
            .build();
    }
}
```

**Service with caching**:
```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        // This method only executes on cache miss
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        Product updated = productRepository.save(product);
        return updated;
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() {
        // Clear all products from cache
    }
}
```

#### Example 3: Cache-Aside Pattern

**Manual cache management**:
```java
@Service
public class ProductService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    private static final String CACHE_KEY_PREFIX = "product:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(5);
    
    public Product getProduct(Long id) {
        String key = CACHE_KEY_PREFIX + id;
        
        // Check cache
        Product cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached; // Cache hit
        }
        
        // Cache miss - load from database
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        // Store in cache
        redisTemplate.opsForValue().set(key, product, CACHE_TTL);
        
        return product;
    }
    
    public void updateProduct(Product product) {
        // Update database
        productRepository.save(product);
        
        // Invalidate cache
        String key = CACHE_KEY_PREFIX + product.getId();
        redisTemplate.delete(key);
    }
}
```

#### Example 4: Cache Warming

**Pre-load cache on startup**:
```java
@Component
public class CacheWarmer implements ApplicationListener<ContextRefreshedEvent> {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        warmCache();
    }
    
    @Async
    public void warmCache() {
        // Pre-load popular products
        List<Long> popularProductIds = getPopularProductIds();
        
        popularProductIds.parallelStream().forEach(productId -> {
            try {
                productService.getProduct(productId); // Triggers cache load
            } catch (Exception e) {
                log.warn("Failed to warm cache for product {}", productId, e);
            }
        });
        
        log.info("Cache warming completed for {} products", popularProductIds.size());
    }
    
    private List<Long> getPopularProductIds() {
        // Get top 1000 products by view count
        return productRepository.findTop1000ByOrderByViewCountDesc()
            .stream()
            .map(Product::getId)
            .collect(Collectors.toList());
    }
}
```

**Scheduled cache warming**:
```java
@Component
public class ScheduledCacheWarmer {
    
    @Autowired
    private ProductService productService;
    
    @Scheduled(cron = "0 0 * * * *") // Every hour
    public void warmTrendingProducts() {
        List<Long> trendingIds = getTrendingProductIds();
        trendingIds.forEach(productService::getProduct);
    }
    
    @Scheduled(cron = "0 0 0 * * *") // Daily at midnight
    public void warmAllProducts() {
        List<Long> allIds = getAllProductIds();
        allIds.parallelStream().forEach(productService::getProduct);
    }
}
```

#### Example 5: Multi-Level Caching

**L1 (Local) + L2 (Redis) cache**:
```java
@Service
public class MultiLevelCacheService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    // L1: Local cache (Caffeine)
    private final Cache<String, Product> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();
    
    private static final String REDIS_KEY_PREFIX = "product:";
    private static final Duration REDIS_TTL = Duration.ofMinutes(10);
    
    public Product getProduct(Long id) {
        String key = String.valueOf(id);
        
        // L1: Check local cache
        Product product = localCache.getIfPresent(key);
        if (product != null) {
            return product; // L1 hit
        }
        
        // L2: Check Redis
        String redisKey = REDIS_KEY_PREFIX + id;
        product = redisTemplate.opsForValue().get(redisKey);
        if (product != null) {
            // Store in L1 cache
            localCache.put(key, product);
            return product; // L2 hit
        }
        
        // Cache miss: Load from database
        product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        // Store in both caches
        localCache.put(key, product);
        redisTemplate.opsForValue().set(redisKey, product, REDIS_TTL);
        
        return product;
    }
}
```

#### Example 6: Cache Tags for Invalidation

**Tag-based cache invalidation**:
```java
@Service
public class TaggedCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void cacheProduct(Product product) {
        String key = "product:" + product.getId();
        redisTemplate.opsForValue().set(key, product, Duration.ofHours(1));
        
        // Add to tag sets
        redisTemplate.opsForSet().add("tag:category:" + product.getCategoryId(), key);
        redisTemplate.opsForSet().add("tag:brand:" + product.getBrandId(), key);
        redisTemplate.opsForSet().add("tag:all", key);
    }
    
    public void invalidateByCategory(Long categoryId) {
        String tagKey = "tag:category:" + categoryId;
        Set<Object> keys = redisTemplate.opsForSet().members(tagKey);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            redisTemplate.delete(tagKey);
        }
    }
    
    public void invalidateByBrand(Long brandId) {
        String tagKey = "tag:brand:" + brandId;
        Set<Object> keys = redisTemplate.opsForSet().members(tagKey);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            redisTemplate.delete(tagKey);
        }
    }
    
    public void invalidateAll() {
        String tagKey = "tag:all";
        Set<Object> keys = redisTemplate.opsForSet().members(tagKey);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            redisTemplate.delete(tagKey);
        }
    }
}
```

#### Example 7: Event-Based Cache Invalidation

**Using message queue for distributed invalidation**:
```java
@Service
public class EventBasedCacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private MessageQueue messageQueue;
    
    /**
     * Invalidate cache when product is updated.
     * Publishes invalidation event for distributed systems.
     */
    @EventListener
    public void onProductUpdated(ProductUpdatedEvent event) {
        Product product = event.getProduct();
        
        // Invalidate local cache
        invalidateProductCache(product.getId());
        
        // Publish invalidation event for other services
        CacheInvalidationEvent invalidationEvent = CacheInvalidationEvent.builder()
            .cacheKey("product:" + product.getId())
            .cacheTags(Arrays.asList(
                "category:" + product.getCategoryId(),
                "brand:" + product.getBrandId()
            ))
            .timestamp(Instant.now())
            .build();
        
        messageQueue.publish("cache.invalidate", invalidationEvent);
    }
    
    /**
     * Listen for cache invalidation events from other services.
     */
    @RabbitListener(queues = "cache.invalidate")
    public void handleCacheInvalidation(CacheInvalidationEvent event) {
        // Invalidate by key
        if (event.getCacheKey() != null) {
            redisTemplate.delete(event.getCacheKey());
        }
        
        // Invalidate by tags
        if (event.getCacheTags() != null) {
            for (String tag : event.getCacheTags()) {
                invalidateByTag(tag);
            }
        }
    }
    
    private void invalidateProductCache(Long productId) {
        String key = "product:" + productId;
        redisTemplate.delete(key);
        
        // Also invalidate related caches
        invalidateRelatedCaches(productId);
    }
    
    private void invalidateRelatedCaches(Long productId) {
        // Invalidate product list caches
        redisTemplate.delete("products:list:*");
        
        // Invalidate category caches
        // (Would need to look up product's category)
    }
}
```

#### Example 8: Version-Based Cache Invalidation

**Using version numbers in cache keys**:
```java
@Service
public class VersionBasedCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private VersionService versionService;
    
    /**
     * Cache with version in key.
     * When data changes, version increments, old cache naturally expires.
     */
    public Product getProduct(Long id) {
        // Get current version
        Long version = versionService.getProductVersion(id);
        
        // Cache key includes version
        String cacheKey = "product:" + id + ":v" + version;
        
        // Try cache
        Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            return product;
        }
        
        // Cache miss: Load from database
        product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        // Cache with TTL
        redisTemplate.opsForValue().set(cacheKey, product, Duration.ofHours(24));
        
        return product;
    }
    
    /**
     * Update product and increment version.
     * Old cache keys become invalid (different version).
     */
    public Product updateProduct(Product product) {
        // Update in database
        Product updated = productRepository.save(product);
        
        // Increment version
        Long newVersion = versionService.incrementProductVersion(product.getId());
        
        // New cache key will be used on next request
        // Old cache key (with old version) will naturally expire
        
        return updated;
    }
}
```

#### Example 9: Write-Through Cache Invalidation

**Update cache and database simultaneously**:
```java
@Service
public class WriteThroughCacheService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    /**
     * Write-through: Update cache and database together.
     * Ensures cache is always fresh.
     */
    public Product updateProduct(Product product) {
        // Update database first
        Product updated = productRepository.save(product);
        
        // Update cache immediately
        String cacheKey = "product:" + updated.getId();
        redisTemplate.opsForValue().set(cacheKey, updated, Duration.ofHours(1));
        
        // Invalidate related caches
        invalidateRelatedCaches(updated);
        
        return updated;
    }
    
    /**
     * Write-behind: Update cache first, async update database.
     * Better performance but risk of data loss.
     */
    @Async
    public CompletableFuture<Product> updateProductAsync(Product product) {
        // Update cache immediately
        String cacheKey = "product:" + product.getId();
        redisTemplate.opsForValue().set(cacheKey, product, Duration.ofHours(1));
        
        // Queue database update
        return CompletableFuture.supplyAsync(() -> {
            return productRepository.save(product);
        });
    }
}
```

#### Example 10: Hierarchical Cache Invalidation

**Invalidate parent and child caches**:
```java
@Service
public class HierarchicalCacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    /**
     * Invalidate cache hierarchy.
     * Example: Category -> Products -> Product Details
     */
    public void invalidateCategory(Long categoryId) {
        // Invalidate category cache
        redisTemplate.delete("category:" + categoryId);
        
        // Invalidate all products in category
        List<Long> productIds = productRepository.findByCategoryId(categoryId);
        for (Long productId : productIds) {
            invalidateProduct(productId);
        }
        
        // Invalidate category list
        redisTemplate.delete("categories:list");
    }
    
    public void invalidateProduct(Long productId) {
        // Invalidate product cache
        redisTemplate.delete("product:" + productId);
        
        // Invalidate product details
        redisTemplate.delete("product:" + productId + ":details");
        
        // Invalidate product recommendations
        redisTemplate.delete("product:" + productId + ":recommendations");
        
        // Invalidate product list caches
        redisTemplate.delete("products:list:*");
    }
}
```

#### Example 11: Probabilistic Cache Invalidation

**Invalidate cache with probability to reduce stampede**:
```java
@Service
public class ProbabilisticCacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    private final Random random = new Random();
    
    /**
     * Probabilistic early expiration.
     * Reduces cache stampede by spreading expiration times.
     */
    public Product getProduct(Long id) {
        String cacheKey = "product:" + id;
        
        // Try cache
        Product product = redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            // Check if should expire early (probabilistic)
            Long ttl = redisTemplate.getExpire(cacheKey);
            if (ttl != null && ttl > 0 && shouldExpireEarly(ttl)) {
                // Expire early with probability
                redisTemplate.expire(cacheKey, Duration.ofSeconds(ttl / 2));
            }
            return product;
        }
        
        // Cache miss: Load from database
        product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        // Cache with random TTL (spread expiration)
        long baseTtl = 3600; // 1 hour
        long randomOffset = random.nextInt(600); // 0-10 minutes
        redisTemplate.opsForValue().set(
            cacheKey, 
            product, 
            Duration.ofSeconds(baseTtl + randomOffset)
        );
        
        return product;
    }
    
    /**
     * Probability of early expiration increases as TTL decreases.
     */
    private boolean shouldExpireEarly(long ttl) {
        // Higher probability when TTL is low
        double probability = 1.0 - (ttl / 3600.0); // 0% at start, 100% at end
        return random.nextDouble() < probability;
    }
}
```

#### Example 12: Cache Invalidation with Database Triggers

**Invalidate cache when database changes**:
```java
@Service
public class DatabaseTriggerCacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * Set up database trigger to publish cache invalidation events.
     * Database -> Trigger -> Message Queue -> Cache Invalidation
     */
    @PostConstruct
    public void setupDatabaseTriggers() {
        // Create database trigger (PostgreSQL example)
        String triggerSql = """
            CREATE OR REPLACE FUNCTION notify_cache_invalidation()
            RETURNS TRIGGER AS $$
            BEGIN
                PERFORM pg_notify('cache_invalidation', 
                    json_build_object(
                        'table', TG_TABLE_NAME,
                        'operation', TG_OP,
                        'id', NEW.id
                    )::text
                );
                RETURN NEW;
            END;
            $$ LANGUAGE plpgsql;
            
            CREATE TRIGGER product_cache_invalidation
            AFTER INSERT OR UPDATE OR DELETE ON products
            FOR EACH ROW EXECUTE FUNCTION notify_cache_invalidation();
            """;
        
        jdbcTemplate.execute(triggerSql);
        
        // Listen for notifications
        listenForDatabaseNotifications();
    }
    
    private void listenForDatabaseNotifications() {
        // Use PostgreSQL LISTEN/NOTIFY or similar mechanism
        // When database changes, trigger cache invalidation
    }
    
    /**
     * Handle database notification.
     */
    public void handleDatabaseNotification(String payload) {
        // Parse notification
        JsonNode notification = parseJson(payload);
        String table = notification.get("table").asText();
        String operation = notification.get("operation").asText();
        Long id = notification.get("id").asLong();
        
        // Invalidate cache based on table and operation
        switch (table) {
            case "products":
                if ("DELETE".equals(operation)) {
                    redisTemplate.delete("product:" + id);
                } else {
                    // Update: Invalidate and let next request refresh
                    redisTemplate.delete("product:" + id);
                    redisTemplate.delete("products:list:*");
                }
                break;
            case "categories":
                invalidateCategory(id);
                break;
        }
    }
}
```

#### Cache Invalidation Best Practices

**1. Choose the Right Strategy**:
- **Time-based (TTL)**: For data that changes predictably
- **Event-based**: For data that must be fresh immediately
- **Version-based**: For frequently changing data
- **Tag-based**: For related content

**2. Invalidate Related Caches**:
- When product updates, invalidate product, category, and list caches
- Use cache tags to track relationships
- Invalidate hierarchically (parent → children)

**3. Handle Distributed Systems**:
- Use message queues for cross-service invalidation
- Use Redis pub/sub for distributed cache invalidation
- Consider eventual consistency vs immediate invalidation

**4. Prevent Cache Stampede**:
- Use probabilistic early expiration
- Add jitter to TTLs
- Use distributed locks for cache warming

**5. Monitor Invalidation**:
- Track invalidation frequency
- Monitor cache hit rates after invalidation
- Alert on excessive invalidations (may indicate issues)

**6. Batch Invalidations**:
- Group related invalidations
- Use pipelines for bulk operations
- Avoid invalidating one-by-one

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Cache Freshness vs Performance
- **Longer TTL**: Better performance, but potentially stale data
- **Shorter TTL**: Fresher data, but more cache misses

#### 2. Memory Usage vs Cache Hit Rate
- **Larger cache**: Higher hit rate, but more memory
- **Smaller cache**: Less memory, but lower hit rate

#### 3. Complexity vs Performance
- **Simple caching**: Easy to implement, limited performance
- **Complex caching**: Better performance, but harder to maintain

### Common Pitfalls

#### Pitfall 1: Stale Data
**Problem**: Cached data becomes outdated
```java
// Bad: Long TTL, data changes frequently
@Cacheable(value = "products", ttl = 3600) // 1 hour
public Product getProduct(Long id) { ... }
// Product price changes, cache still has old price
```

**Solution**: Appropriate TTLs, cache invalidation on updates
```java
// Good: Shorter TTL or invalidate on update
@Cacheable(value = "products", ttl = 300) // 5 minutes
@CacheEvict(value = "products", key = "#product.id")
public Product updateProduct(Product product) { ... }
```

#### Pitfall 2: Cache Stampede
**Problem**: Many requests miss cache simultaneously, all hit database
```java
// Bad: No protection against stampede
public Product getProduct(Long id) {
    Product cached = cache.get(id);
    if (cached == null) {
        // 1000 requests all hit database simultaneously
        return loadFromDatabase(id);
    }
    return cached;
}
```

**Solution**: Locking, pre-warming, or probabilistic early expiration
```java
// Good: Use distributed lock
public Product getProduct(Long id) {
    Product cached = cache.get(id);
    if (cached == null) {
        Lock lock = distributedLock.acquire("product:" + id);
        try {
            // Double-check after acquiring lock
            cached = cache.get(id);
            if (cached == null) {
                cached = loadFromDatabase(id);
                cache.put(id, cached);
            }
        } finally {
            lock.release();
        }
    }
    return cached;
}
```

#### Pitfall 3: Not Setting Cache Headers
**Problem**: Responses not cached, performance not optimized
```java
// Bad: No cache headers
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.getProduct(id);
}
```

**Solution**: Set appropriate cache headers
```java
// Good: Cache headers set
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id) {
    Product product = productService.getProduct(id);
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
        .body(product);
}
```

#### Pitfall 4: Caching User-Specific Data in Shared Cache
**Problem**: One user's data served to another user
```java
// Bad: User-specific data in shared cache
@Cacheable(value = "profiles", key = "#id")
public UserProfile getProfile(Long id) {
    // If cached, any user could get this profile
}
```

**Solution**: Use private cache or include user ID in key
```java
// Good: User-specific cache key
@Cacheable(value = "profiles", key = "#userId + ':' + #id")
public UserProfile getProfile(Long userId, Long id) {
    // Cache key includes user ID
}
```

#### Pitfall 5: Cache Key Collisions
**Problem**: Different data types use same cache key pattern
```java
// Bad: Potential collision
cache.put("user:123", user);      // User object
cache.put("user:123", userProfile); // Different object, same key
```

**Solution**: Use unique key prefixes
```java
// Good: Unique prefixes
cache.put("user:123", user);
cache.put("userProfile:123", userProfile);
```

### Performance Gotchas

#### Gotcha 1: Cache Warming Overhead
**Problem**: Warming cache on startup delays application start
**Solution**: Async cache warming, warm in background

#### Gotcha 2: Cache Memory Pressure
**Problem**: Cache grows too large, causes memory issues
**Solution**: Size limits, eviction policies, monitor memory usage

## 8️⃣ When NOT to use this

### When Caching Isn't Needed

1. **Real-time data**: Data that changes constantly, caching adds complexity without benefit

2. **Unique requests**: Every request is different, no cache hits

3. **Very fast operations**: Operations that are already fast (< 1ms), caching overhead > benefit

4. **Write-heavy workloads**: Mostly writes, few reads, caching doesn't help much

### Anti-Patterns

1. **Over-caching**: Caching everything, including non-beneficial data

2. **Under-caching**: Not caching data that would benefit

3. **No cache invalidation**: Stale data served to users

## 9️⃣ Comparison with Alternatives

### HTTP Caching vs Application Caching

**HTTP Caching**: Browser and CDN cache responses
- Pros: Reduces server load, works across requests, transparent
- Cons: Less control, cache invalidation harder

**Application Caching**: Application manages cache
- Pros: Full control, precise invalidation, application-specific logic
- Cons: More complex, requires cache management code

**Best practice**: Use both - HTTP caching for static/dynamic content, application caching for computed data

### Cache-Aside vs Write-Through

**Cache-Aside**: Application manages cache, writes to database, then cache
- Pros: Simple, flexible, handles cache failures
- Cons: Potential for stale data, more database writes

**Write-Through**: Write to cache and database simultaneously
- Pros: Cache always consistent, no stale data
- Cons: Slower writes, more complex

**Best practice**: Cache-aside for reads, write-through for critical data that must be consistent

## 🔟 Interview follow-up questions WITH answers

### Question 1: How do you handle cache invalidation in a distributed system?

**Answer**:
**Challenges**:
- Multiple servers have their own caches
- Need to invalidate across all servers
- Race conditions (data updated while cache invalidated)

**Strategies**:

1. **Cache invalidation messages**: Publish invalidation events
```java
// Update database
productRepository.save(product);

// Publish invalidation event
messageQueue.publish("cache.invalidate", "product:" + product.getId());

// All servers listen and invalidate their caches
@EventListener
public void handleInvalidation(CacheInvalidationEvent event) {
    cache.evict(event.getKey());
}
```

2. **TTL-based expiration**: Let cache expire naturally
- Simpler, but allows stale data temporarily
- Good for data that can tolerate slight staleness

3. **Version-based invalidation**: Include version in cache key
```java
String cacheKey = "product:" + product.getId() + ":" + product.getVersion();
// When version changes, old cache entries become invalid
```

4. **Redis pub/sub**: Use Redis for distributed cache invalidation
```java
// Server 1: Updates and invalidates
redis.delete("product:123");
redis.publish("cache.invalidate", "product:123");

// All servers: Subscribe and invalidate local cache
redis.subscribe("cache.invalidate", (channel, key) -> {
    localCache.evict(key);
});
```

### Question 2: How do you prevent cache stampede?

**Answer**:
**Cache stampede**: Many requests miss cache simultaneously, all hit database

**Solutions**:

1. **Distributed locking**: Only one request loads, others wait
```java
Lock lock = distributedLock.acquire("product:" + id);
try {
    Product cached = cache.get(id);
    if (cached == null) {
        cached = loadFromDatabase(id);
        cache.put(id, cached);
    }
    return cached;
} finally {
    lock.release();
}
```

2. **Probabilistic early expiration**: Refresh cache before it expires
```java
long ttl = cache.getTTL(key);
if (ttl < random(0.1 * originalTTL, 0.2 * originalTTL)) {
    // Refresh cache asynchronously
    asyncRefresh(key);
}
```

3. **Background refresh**: Always refresh in background before expiration
```java
@Scheduled(fixedRate = 300000) // Every 5 minutes
public void refreshCache() {
    // Refresh items expiring soon
    cache.getKeysExpiringSoon().forEach(this::refresh);
}
```

4. **Cache warming**: Pre-load cache before expiration
```java
// Refresh cache 1 minute before expiration
if (timeToExpiry < 60) {
    asyncRefresh(key);
}
```

### Question 3: How do you choose cache TTL values?

**Answer**:
**Factors**:

1. **Data change frequency**:
   - Frequently changing: Short TTL (minutes)
   - Rarely changing: Long TTL (hours/days)

2. **Staleness tolerance**:
   - Must be fresh: Short TTL
   - Can tolerate staleness: Long TTL

3. **Cache hit rate target**:
   - Need high hit rate: Longer TTL
   - Accept lower hit rate: Shorter TTL

4. **Memory constraints**:
   - Limited memory: Shorter TTL (evicts more)
   - Plenty of memory: Longer TTL

**Examples**:
- **Product details**: 5 minutes (price changes occasionally)
- **User profile**: 1 hour (changes infrequently)
- **Static content**: 24 hours (rarely changes)
- **News articles**: 1 hour (changes after publishing)

**Best practice**: Start with conservative TTL, monitor hit rates and staleness, adjust based on metrics

### Question 4: How do you implement cache warming for a high-traffic system?

**Answer**:
**Strategies**:

1. **Startup warming**: Pre-load cache when application starts
```java
@PostConstruct
public void warmCacheOnStartup() {
    // Pre-load popular items
    popularItems.forEach(this::loadIntoCache);
}
```

2. **Scheduled warming**: Periodically refresh cache
```java
@Scheduled(cron = "0 0 * * * *") // Hourly
public void warmTrendingItems() {
    trendingItems.forEach(this::refreshCache);
}
```

3. **Predictive warming**: Warm cache based on patterns
```java
// Warm products likely to be viewed next
List<Long> predictedProducts = mlModel.predictNextViews(currentProduct);
predictedProducts.forEach(this::warmCache);
```

4. **Event-driven warming**: Warm on related events
```java
@EventListener
public void onProductView(ProductViewedEvent event) {
    // Warm related products
    relatedProducts(event.getProductId()).forEach(this::warmCache);
}
```

**Considerations**:
- Async warming (don't block startup)
- Gradual warming (don't overwhelm system)
- Monitor warming effectiveness
- Prioritize high-value items

### Question 5: How do you measure cache effectiveness?

**Answer**:
**Key metrics**:

1. **Cache hit rate**: Percentage of requests served from cache
   ```
   Hit rate = Cache hits / (Cache hits + Cache misses)
   Target: > 80% for good performance
   ```

2. **Average response time**: Compare cached vs non-cached
   ```
   Cached: 5ms
   Non-cached: 50ms
   Improvement: 10x faster
   ```

3. **Cache size**: Memory usage
   ```
   Monitor: Cache size, memory pressure
   Alert: If cache size approaches limits
   ```

4. **Eviction rate**: How often items evicted
   ```
   High eviction rate: Cache too small or TTL too short
   ```

5. **Staleness**: How often stale data served
   ```
   Track: Data age when served
   Alert: If stale data served too frequently
   ```

**Implementation**:
```java
@Aspect
@Component
public class CacheMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordCacheHit(String cacheName) {
        meterRegistry.counter("cache.hits", "cache", cacheName).increment();
    }
    
    public void recordCacheMiss(String cacheName) {
        meterRegistry.counter("cache.misses", "cache", cacheName).increment();
    }
    
    public void recordCacheAccessTime(String cacheName, long duration) {
        meterRegistry.timer("cache.access.time", "cache", cacheName)
            .record(duration, TimeUnit.MILLISECONDS);
    }
}
```

### Question 6 (L5/L6): How would you design a caching system that handles millions of requests per second?

**Answer**:
**Design considerations**:

1. **Multi-level caching**:
   - L1: Local cache (Caffeine) - fastest, limited size
   - L2: Distributed cache (Redis Cluster) - larger, shared
   - L3: Database - source of truth

2. **Sharding**: Distribute cache across multiple nodes
   - Consistent hashing for key distribution
   - Reduces single node load

3. **Cache topology**:
   ```
   Request → L1 Cache (Local) → L2 Cache (Redis) → Database
              ↓ hit              ↓ hit              ↓ hit
            Response          Response          Response
   ```

4. **Optimizations**:
   - **Batch loading**: Load multiple items in one operation
   - **Pipelining**: Pipeline Redis operations
   - **Async refresh**: Refresh cache asynchronously
   - **Compression**: Compress cached values to reduce memory

5. **Monitoring**:
   - Cache hit rates per level
   - Latency per level
   - Memory usage
   - Eviction rates

**Architecture**:
```
Load Balancer → API Servers (L1: Caffeine)
                    ↓ (miss)
                Redis Cluster (L2, sharded)
                    ↓ (miss)
                Database (L3)
```

**Key principles**:
- Hot data in L1 (fastest access)
- Warm data in L2 (shared, larger)
- Cold data in database
- Monitor and tune based on access patterns

## 1️⃣1️⃣ One clean mental summary

Caching strategies for performance store frequently accessed data in fast storage (memory) to avoid expensive operations, dramatically improving response times and reducing server load. Multi-layer caching includes browser cache (client-side HTTP responses), CDN cache (edge servers for global distribution), application cache (server-side for database queries and computed data), and database query cache (database-level caching). HTTP caching uses headers like `Cache-Control`, `ETag`, and `Last-Modified` to control how responses are cached. Critical considerations include cache invalidation (time-based expiration, event-based invalidation), cache warming (pre-loading popular data), preventing cache stampede (distributed locking, probabilistic early expiration), and measuring effectiveness (hit rates, response time improvements). Always set appropriate TTLs based on data change frequency, use cache headers for HTTP responses, invalidate cache on updates, and monitor cache metrics to tune performance. Use multi-level caching (L1 local, L2 distributed) for optimal performance at scale.

