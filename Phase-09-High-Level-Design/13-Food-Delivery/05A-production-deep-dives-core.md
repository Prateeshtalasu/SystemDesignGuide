# Food Delivery - Production Deep Dives (Core)

## Overview

This document covers the core production components: asynchronous messaging (Kafka for order events), caching (Redis for menus and real-time data), and search capabilities (Elasticsearch for restaurant discovery). These are the fundamental building blocks that enable the food delivery system to meet its performance and scale requirements.

---

## 1. Async, Messaging & Event Flow (Kafka)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant data pipelines. In a food delivery system, Kafka serves as the backbone for event-driven architecture, enabling services to communicate asynchronously and maintaining a complete audit trail of all order events.

**What problems does Kafka solve for food delivery?**

1. **Decoupling**: Order service doesn't need to wait for notification service
2. **Event sourcing**: Complete order history for auditing and replay
3. **Peak handling**: Buffer orders during dinner rush
4. **Analytics**: Real-time stream processing for business metrics
5. **Reliability**: Events persist even if consumers are temporarily down

**Core Kafka concepts:**

| Concept | Definition | Food Delivery Example |
|---------|------------|----------------------|
| **Topic** | A named stream of messages | `order-events`, `delivery-updates` |
| **Partition** | A topic split for parallelism | Partitioned by `order_id` |
| **Offset** | Unique ID for each message | Position in order stream |
| **Consumer Group** | Consumers sharing work | Notification consumers |
| **Broker** | Kafka server storing messages | 6 brokers for HA |

### B) OUR USAGE: How We Use Kafka Here

**Why async vs sync for order events?**

Synchronous processing would create bottlenecks:
- Order placement: Must notify restaurant, update search, send confirmation
- Status updates: Must notify customer, partner, update analytics
- During peak: 30 orders/second, each triggering 5+ downstream actions

By using Kafka, the order service returns immediately after publishing events, and downstream services process asynchronously.

**Events in our system:**

| Event | Topic | Volume | Purpose |
|-------|-------|--------|---------|
| `OrderPlaced` | order-events | 30/sec peak | Trigger restaurant notification |
| `OrderStatusChanged` | order-events | 100/sec peak | Update all parties |
| `DeliveryAssigned` | delivery-events | 30/sec peak | Notify partner |
| `LocationUpdated` | location-updates | 7,500/sec | Track deliveries |
| `PaymentProcessed` | payment-events | 30/sec peak | Financial audit |
| `MenuItemUpdated` | menu-updates | 100/sec | Update search index |

**Topic/Queue Design:**

```
Topic: order-events
Partitions: 24
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
Partition Key: order_id

Topic: delivery-events
Partitions: 12
Replication Factor: 3
Retention: 3 days
Cleanup Policy: delete
Partition Key: delivery_id

Topic: location-updates
Partitions: 24
Replication Factor: 3
Retention: 24 hours
Cleanup Policy: delete
Partition Key: partner_id

Topic: menu-updates
Partitions: 6
Replication Factor: 3
Retention: 7 days
Cleanup Policy: compact
Partition Key: restaurant_id
```

**Why 24 partitions for order-events?**
- Peak: 30 orders/second, each generating multiple events
- Each partition handles ~5-10 messages/second efficiently
- 24 partitions allow 24 parallel consumers
- Room for growth without repartitioning

**Partition Key Choice:**

| Topic | Partition Key | Reasoning |
|-------|---------------|-----------|
| order-events | order_id | All events for an order are ordered together |
| delivery-events | delivery_id | Delivery lifecycle events ordered |
| location-updates | partner_id | Partner locations stay in order |
| menu-updates | restaurant_id | Menu changes for restaurant are ordered |

**Consumer Group Design:**

```
Consumer Group: notification-service
Consumers: 6 instances
Purpose: Send push notifications, SMS, emails

Consumer Group: search-indexer
Consumers: 3 instances
Purpose: Update Elasticsearch indexes

Consumer Group: analytics-processor
Consumers: 4 instances
Purpose: Real-time metrics, dashboards

Consumer Group: restaurant-notifier
Consumers: 6 instances
Purpose: Send orders to restaurant tablets
```

**Ordering Guarantees:**

- **Per-key ordering**: Guaranteed (all events for same order in order)
- **Global ordering**: Not guaranteed (not needed)
- **Exactly-once**: Achieved with idempotent consumers

**Offset Management:**

We use manual offset commits for reliability:

```java
@KafkaListener(topics = "order-events", groupId = "notification-service")
public void processOrderEvent(
        List<OrderEvent> events, 
        Acknowledgment ack) {
    try {
        for (OrderEvent event : events) {
            // Idempotency check
            if (!processedEventRepository.exists(event.getEventId())) {
                processEvent(event);
                processedEventRepository.markProcessed(event.getEventId());
            }
        }
        ack.acknowledge();
    } catch (Exception e) {
        log.error("Failed to process order events", e);
        // Don't acknowledge, will be reprocessed
    }
}
```

**Deduplication Strategy:**

```java
@Service
public class IdempotentEventProcessor {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean shouldProcess(String eventId) {
        // Distributed deduplication with Redis
        Boolean isNew = redis.opsForValue()
            .setIfAbsent("processed:" + eventId, "1", Duration.ofHours(24));
        
        return Boolean.TRUE.equals(isNew);
    }
}
```

**Outbox Pattern for Order Creation:**

For order creation, we need transactional consistency between database and Kafka:

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Create order in database
        Order order = orderRepository.save(buildOrder(request));
        
        // 2. Write to outbox table (same transaction)
        OutboxEvent event = OutboxEvent.builder()
            .aggregateId(order.getId())
            .eventType("ORDER_PLACED")
            .payload(objectMapper.writeValueAsString(order))
            .createdAt(Instant.now())
            .build();
        outboxRepository.save(event);
        
        return order;
    }
}

// Separate process reads outbox and publishes to Kafka
@Scheduled(fixedDelay = 100)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished(100);
    for (OutboxEvent event : events) {
        kafkaTemplate.send("order-events", event.getAggregateId(), event.getPayload());
        event.setPublished(true);
        outboxRepository.save(event);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Event Flow: Order Placement**

```
Customer places order
    ↓
Order Service:
    1. Validate items with Restaurant Service
    2. Calculate pricing
    3. Authorize payment
    4. Insert order (PostgreSQL)
    5. Insert outbox event (same transaction)
    ↓
Outbox Publisher (every 100ms):
    1. Read unpublished events
    2. Publish to Kafka topic "order-events"
    3. Mark as published
    ↓
Kafka receives OrderPlaced event:
    {
      "eventId": "evt_abc123",
      "eventType": "ORDER_PLACED",
      "orderId": "order_xyz789",
      "restaurantId": "rest_456",
      "customerId": "user_123",
      "items": [...],
      "total": 43.91,
      "timestamp": 1705312800000
    }
    ↓
Partition selected by hash(order_xyz789) % 24 = partition 7
    ↓
[ASYNC] Consumers process:
    - Restaurant Notifier: Send to restaurant tablet
    - Notification Service: Send confirmation to customer
    - Search Indexer: Update restaurant order count
    - Analytics: Update real-time metrics
```

**Event Flow: Order Status Update**

```
Restaurant marks order as READY
    ↓
Order Service updates status
    ↓
Publish OrderStatusChanged event:
    {
      "eventId": "evt_def456",
      "eventType": "ORDER_STATUS_CHANGED",
      "orderId": "order_xyz789",
      "previousStatus": "PREPARING",
      "newStatus": "READY",
      "timestamp": 1705313400000
    }
    ↓
[ASYNC] Consumers:
    - Notification: Push to customer "Your order is ready!"
    - Delivery Service: Assign delivery partner
    - Analytics: Update preparation time metrics
```

**Failure Scenarios:**

**Q: What if consumer crashes mid-batch?**

A: Offset is not committed, so the batch will be reprocessed. Our idempotent processing handles duplicates using the eventId.

**Q: What if Kafka is down?**

A: Order creation still works (outbox pattern):
1. Orders saved to PostgreSQL
2. Events accumulate in outbox table
3. When Kafka recovers, outbox publisher catches up
4. Notifications delayed but no data loss

### Hot Partition Mitigation

**Problem:** During peak times (e.g., Super Bowl Sunday), one popular restaurant might receive many orders, causing one Kafka partition to become hot and creating a bottleneck for order processing.

**Example Scenario:**
```
Normal restaurant: 5 orders/hour → Partition 8 (hash(restaurant_id) % partitions)
Popular restaurant: 50 orders/hour → Partition 8 (same partition!)
Result: Partition 8 receives 10x traffic, becomes bottleneck
Order event consumers for partition 8 become overwhelmed
```

**Detection:**
- Monitor partition lag per partition (Kafka metrics)
- Alert when partition lag > 2x average
- Track partition throughput metrics (orders/second per partition)
- Monitor consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Mitigation Strategies:**

1. **Composite Partition Key (Primary Mitigation):**
   - Use composite key: `restaurantId:orderId` instead of just `restaurant_id`
   - This distributes orders from the same restaurant across multiple partitions
   - Maintains ordering per order (all events for one order stay together)
   - Breaks up hot restaurant traffic

2. **Partition Rebalancing:**
   - If hot partition detected, increase partition count
   - Rebalance existing partitions (Kafka supports this)
   - Distribute hot restaurant orders across multiple partitions

3. **Consumer Scaling:**
   - Add more consumers for hot partitions
   - Auto-scale consumers based on partition lag
   - Each consumer handles subset of hot partition

4. **Rate Limiting:**
   - Rate limit per restaurant (prevent single restaurant from overwhelming)
   - Throttle high-volume restaurants during peak
   - Buffer order events locally if rate limit exceeded

**Implementation:**

```java
@Service
public class HotPartitionDetector {
    
    public void detectAndMitigate() {
        Map<Integer, Long> partitionLag = getPartitionLag();
        double avgLag = partitionLag.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);
        
        partitionLag.entrySet().stream()
            .filter(e -> e.getValue() > avgLag * 2)
            .forEach(e -> {
                int partition = e.getKey();
                long lag = e.getValue();
                
                // Alert ops team
                alertOps("Hot partition detected: " + partition + ", lag: " + lag);
                
                // Auto-scale consumers for this partition
                scaleConsumers(partition, lag);
            });
    }
}
```

**Monitoring:**
- Partition lag per partition (Kafka metrics)
- Partition throughput (orders/second per partition)
- Consumer lag per partition
- Alert threshold: Lag > 2x average for 5 minutes

**Q: How does retry work?**

A: Producer retries:
```java
config.put(ProducerConfig.RETRIES_CONFIG, 3);
config.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);
```

Consumer retries: Don't acknowledge, message reprocessed. After 5 failures, send to dead letter queue.

**Q: What breaks first and what degrades gracefully?**

A: Order of failure:
1. **Notification lag**: Customers don't get updates immediately, but orders work
2. **Search index lag**: New restaurants don't appear immediately
3. **Analytics lag**: Dashboards delayed
4. **All Kafka down**: Orders still work, notifications fail

### Event Schemas

```java
public abstract class OrderEvent {
    private String eventId;
    private String eventType;
    private String orderId;
    private Instant timestamp;
}

public class OrderPlacedEvent extends OrderEvent {
    private String customerId;
    private String restaurantId;
    private List<OrderItem> items;
    private BigDecimal subtotal;
    private BigDecimal total;
    private Address deliveryAddress;
}

public class OrderStatusChangedEvent extends OrderEvent {
    private String previousStatus;
    private String newStatus;
    private String changedBy;  // CUSTOMER, RESTAURANT, PARTNER, SYSTEM
    private String reason;
}
```

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Reliability for order events
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // Batching
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

---

## 2. Caching (Redis)

### A) CONCEPT: What is Redis and Caching Patterns?

Redis is an in-memory data structure store providing sub-millisecond latency. For food delivery, Redis serves multiple critical purposes:

1. **Menu caching**: Avoid database hits for frequently viewed menus
2. **Session storage**: User authentication state
3. **Real-time data**: Active orders, partner locations
4. **Rate limiting**: Protect against abuse

**Caching Patterns:**

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Cache-Aside** | App checks cache, then DB on miss | Restaurant data |
| **Write-Through** | Write to cache and DB together | Active orders |
| **Write-Behind** | Write to cache, async to DB | Location updates |

**We use multiple patterns:**
- **Cache-Aside** for restaurant and menu data
- **Write-Through** for active orders (must be consistent)
- **Write-Behind** for partner locations (DB is for history)

### B) OUR USAGE: How We Use Redis Here

**Data Structures and Keys:**

```
1. RESTAURANT DATA (Hash)
   Key: restaurant:{restaurant_id}
   Fields: name, rating, is_open, delivery_fee, min_order, etc.
   TTL: 1 hour
   Invalidation: On restaurant update
   
2. MENU DATA (String - JSON)
   Key: menu:{restaurant_id}
   Value: Full menu JSON (categories, items, customizations)
   TTL: 15 minutes
   Invalidation: On any menu item change
   
3. ACTIVE ORDERS (Hash)
   Key: order:{order_id}
   Fields: status, restaurant_id, partner_id, customer_id, timestamps
   TTL: None (deleted on completion + 1 hour)
   
4. PARTNER LOCATIONS (Geospatial)
   Key: partners:locations
   Type: GEOADD / GEORADIUS
   Purpose: Find nearby delivery partners
   
5. PARTNER STATUS (Hash)
   Key: partner:{partner_id}
   Fields: status, current_order, last_location, last_update
   TTL: 30 seconds (auto-offline if not refreshed)
   
6. USER SESSIONS (Hash)
   Key: session:{session_id}
   Fields: user_id, user_type, device_id, created_at
   TTL: 30 minutes (sliding)
   
7. SEARCH RESULTS CACHE (String)
   Key: search:{hash_of_params}
   Value: Search results JSON
   TTL: 5 minutes
   
8. RESTAURANT AVAILABILITY (String)
   Key: available:{restaurant_id}
   Value: "true" or "false"
   TTL: 5 minutes
   Purpose: Quick check without full restaurant load
```

**Who reads Redis?**
- Search Service: Cached search results
- Order Service: Active order state
- Delivery Service: Partner locations
- API Gateway: Sessions, rate limits

**Who writes Redis?**
- Restaurant Service: Restaurant and menu data
- Order Service: Order state changes
- Delivery Service: Partner locations
- Auth Service: Sessions

**TTL Behavior:**

| Data | TTL | Reasoning |
|------|-----|-----------|
| Restaurant data | 1 hour | Changes infrequently |
| Menu data | 15 minutes | Items can sell out |
| Active orders | None | Must persist until complete |
| Partner location | 30 seconds | Stale = offline |
| Search results | 5 minutes | Balance freshness and load |
| Sessions | 30 min sliding | Security + UX |

**Eviction Policy:**

```
maxmemory 4gb
maxmemory-policy volatile-lru
```

Why volatile-lru?
- Only evict keys with TTL
- Active orders (no TTL) never evicted
- Menu cache can be evicted and reloaded

**Invalidation Strategy:**

On menu item update:
```java
@Service
public class MenuCacheService {
    
    public void invalidateMenu(String restaurantId) {
        // Delete menu cache
        redisTemplate.delete("menu:" + restaurantId);
        
        // Also invalidate search cache for this restaurant
        // (search results include menu item counts)
        redisTemplate.delete("search:*");  // Or use pattern delete
        
        // Publish event for search index update
        kafkaTemplate.send("menu-updates", restaurantId, 
            new MenuUpdatedEvent(restaurantId));
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Cache HIT Path: Get Restaurant Menu**

```
Request: GET /v1/restaurants/rest_456/menu
    ↓
Restaurant Service:
    ↓
Redis: GET menu:rest_456
    ↓
Redis returns: Full menu JSON
    ↓
Parse and return to client
    ↓
Total latency: ~10ms
```

**Cache MISS Path: Get Restaurant Menu**

```
Request: GET /v1/restaurants/rest_456/menu
    ↓
Restaurant Service:
    ↓
Redis: GET menu:rest_456
    ↓
Redis returns: (nil) - MISS
    ↓
PostgreSQL: 
    SELECT * FROM menu_items 
    WHERE restaurant_id = 'rest_456' 
    ORDER BY category_id, display_order
    ↓
Build menu JSON with categories and items
    ↓
Redis: SET menu:rest_456 {menu_json} EX 900
    ↓
Return to client
    ↓
Total latency: ~50ms
```

**Write-Through: Update Order Status**

```
Request: Update order_xyz789 to PREPARING
    ↓
Order Service:
    ↓
1. Update PostgreSQL:
   UPDATE orders SET status = 'PREPARING', 
   updated_at = NOW() WHERE id = 'order_xyz789'
    ↓
2. Update Redis:
   HSET order:order_xyz789 status "PREPARING" updated_at "..."
    ↓
3. Publish event to Kafka
    ↓
4. Return success
    ↓
Total latency: ~30ms
```

**Geospatial: Find Available Partners**

```
Order ready for pickup at restaurant (37.7749, -122.4194)
    ↓
Delivery Service:
    ↓
Redis: GEORADIUS partners:locations -122.4194 37.7749 5 km 
       WITHCOORD WITHDIST COUNT 20 ASC
    ↓
Redis returns: [
    {partner_id: "p1", distance: 0.5km},
    {partner_id: "p2", distance: 1.2km},
    ...
]
    ↓
For each partner, check status:
Redis: HGET partner:p1 status
    ↓
Filter: status == "ONLINE" AND current_order == null
    ↓
Return available partners sorted by distance
    ↓
Total latency: ~5ms
```

**Failure Scenarios:**

**Q: What if Redis is down?**

A: Circuit breaker activates:

```java
@CircuitBreaker(name = "redis", fallbackMethod = "getMenuFromDB")
public Menu getMenu(String restaurantId) {
    String cached = redisTemplate.opsForValue().get("menu:" + restaurantId);
    if (cached != null) {
        return objectMapper.readValue(cached, Menu.class);
    }
    return loadAndCacheMenu(restaurantId);
}

public Menu getMenuFromDB(String restaurantId, Exception e) {
    log.warn("Redis unavailable, loading from DB", e);
    return menuRepository.findByRestaurantId(restaurantId);
}
```

Impact:
- Menu requests hit database directly
- Latency increases from 10ms to 50ms
- Database load increases 5-10x
- System still functional but degraded

**Q: What happens during peak hours?**

A: Cache becomes critical:
- 290 search requests/second at peak
- Without cache: 290 × 50ms = 14.5 seconds of DB time per second
- With cache (90% hit rate): Only 29 requests hit DB

**Q: What is the circuit breaker behavior?**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      redis:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
```

**Q: Why 15-minute TTL for menus?**

A: Balance between:
- Freshness: Items can sell out, prices can change
- Performance: Menus are large (50KB average)
- During peak, 15 minutes is acceptable staleness
- Item availability checked at order time anyway

### Cache Operations Code

```java
@Service
public class RestaurantCacheService {
    
    private final RedisTemplate<String, String> redis;
    private final ObjectMapper mapper;
    
    private static final Duration RESTAURANT_TTL = Duration.ofHours(1);
    private static final Duration MENU_TTL = Duration.ofMinutes(15);
    
    public Optional<Restaurant> getRestaurant(String restaurantId) {
        String key = "restaurant:" + restaurantId;
        String cached = redis.opsForValue().get(key);
        
        if (cached != null) {
            return Optional.of(mapper.readValue(cached, Restaurant.class));
        }
        return Optional.empty();
    }
    
    public void cacheRestaurant(Restaurant restaurant) {
        String key = "restaurant:" + restaurant.getId();
        redis.opsForValue().set(
            key,
            mapper.writeValueAsString(restaurant),
            RESTAURANT_TTL
        );
    }
    
    public void cacheMenu(String restaurantId, Menu menu) {
        String key = "menu:" + restaurantId;
        redis.opsForValue().set(
            key,
            mapper.writeValueAsString(menu),
            MENU_TTL
        );
    }
    
    public void invalidateRestaurant(String restaurantId) {
        redis.delete("restaurant:" + restaurantId);
        redis.delete("menu:" + restaurantId);
    }
}
```

### Cache Stampede Prevention

**What is Cache Stampede?**

Cache stampede (also called "thundering herd") occurs when cached restaurant or menu data expires and many requests simultaneously try to regenerate it, overwhelming the database.

**Scenario:**
```
T+0s:   Popular restaurant's menu cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database simultaneously
T+1s:   Database overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking to ensure only one request regenerates the cache:

1. **Distributed locking**: Only one thread fetches from database on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class MenuCacheService {
    
    private final RedisTemplate<String, Menu> redis;
    private final DistributedLock lockService;
    private final MenuRepository menuRepository;
    
    public Menu getCachedMenu(String restaurantId) {
        String key = "menu:" + restaurantId;
        
        // 1. Try cache first
        Menu cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:menu:" + restaurantId;
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
            
            // 4. Fetch from database (only one thread does this)
            Menu menu = menuRepository.findByRestaurantId(restaurantId);
            
            if (menu != null) {
                // 5. Populate cache with TTL jitter (add ±10% random jitter to 15min TTL)
                Duration baseTtl = Duration.ofMinutes(15);
                long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
                Duration ttl = baseTtl.plusSeconds(jitter);
                
                redis.opsForValue().set(key, menu, ttl);
            }
            
            return menu;
            
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

## 3. Search (Elasticsearch)

### A) CONCEPT: What is Elasticsearch and Inverted Index?

Elasticsearch is a distributed search engine built on Apache Lucene. It provides full-text search, structured search, and analytics capabilities. For food delivery, it powers restaurant discovery, the most critical user-facing feature.

**What is an inverted index?**

An inverted index maps terms to the documents containing them:

```
Traditional index (document → terms):
  doc1: "Mario's Italian Kitchen"
  doc2: "Italian Garden Restaurant"

Inverted index (term → documents):
  "mario's" → [doc1]
  "italian" → [doc1, doc2]
  "kitchen" → [doc1]
  "garden" → [doc2]
  "restaurant" → [doc2]
```

This allows fast full-text search: "italian" returns both restaurants instantly.

**What gets indexed?**

For food delivery, we index:
- Restaurant name, description
- Cuisine types
- Menu item names
- Location (geo_point)
- Rating, price level
- Operating hours

### B) OUR USAGE: How We Use Elasticsearch Here

**Source of Truth:**
- PostgreSQL is the source of truth
- Elasticsearch is a read-optimized view
- Changes flow: PostgreSQL → Kafka → Elasticsearch

**Indexer Service Design:**

```
PostgreSQL (source)
       │
       ▼
   Kafka Topic
  "menu-updates"
       │
       ▼
  Indexer Service
       │
       ▼
  Elasticsearch
```

**Index Mapping:**

```json
{
  "mappings": {
    "properties": {
      "id": {"type": "keyword"},
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {"type": "keyword"},
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete"
          }
        }
      },
      "description": {"type": "text"},
      "cuisine_types": {"type": "keyword"},
      "location": {"type": "geo_point"},
      "rating": {"type": "float"},
      "rating_count": {"type": "integer"},
      "price_level": {"type": "integer"},
      "delivery_time_minutes": {"type": "integer"},
      "is_open": {"type": "boolean"},
      "is_accepting_orders": {"type": "boolean"},
      "menu_items": {
        "type": "nested",
        "properties": {
          "name": {"type": "text"},
          "description": {"type": "text"},
          "price": {"type": "float"}
        }
      }
    }
  }
}
```

**Query Patterns:**

1. **Location-based search**: Find restaurants near customer
2. **Text search**: Search by name or cuisine
3. **Filtered search**: By rating, price, delivery time
4. **Autocomplete**: As-you-type suggestions

**Ranking Approach:**

We use a custom scoring function:

```java
public SearchSourceBuilder buildSearchQuery(SearchRequest request) {
    BoolQueryBuilder query = QueryBuilders.boolQuery();
    
    // Must be open and accepting orders
    query.filter(QueryBuilders.termQuery("is_open", true));
    query.filter(QueryBuilders.termQuery("is_accepting_orders", true));
    
    // Location filter
    query.filter(QueryBuilders.geoDistanceQuery("location")
        .point(request.getLat(), request.getLng())
        .distance(5, DistanceUnit.KILOMETERS));
    
    // Text search with boosting
    if (request.getQuery() != null) {
        query.must(QueryBuilders.multiMatchQuery(request.getQuery())
            .field("name", 3.0f)
            .field("cuisine_types", 2.0f)
            .field("menu_items.name", 1.0f));
    }
    
    // Custom scoring
    FunctionScoreQueryBuilder functionScore = QueryBuilders
        .functionScoreQuery(query)
        .add(ScoreFunctionBuilders.fieldValueFactorFunction("rating")
            .factor(1.5f)
            .modifier(FieldValueFactorFunction.Modifier.LOG1P))
        .add(ScoreFunctionBuilders.linearDecayFunction("location", 
            new GeoPoint(request.getLat(), request.getLng()), "3km", "1km"))
        .scoreMode(ScoreMode.MULTIPLY);
    
    return new SearchSourceBuilder()
        .query(functionScore)
        .from(request.getOffset())
        .size(request.getLimit());
}
```

### C) REAL STEP-BY-STEP SIMULATION

**Data Flow: Restaurant Update**

```
Restaurant updates operating hours
    ↓
Restaurant Service updates PostgreSQL
    ↓
Publish RestaurantUpdated event to Kafka
    ↓
Indexer Service consumes event:
    1. Fetch full restaurant data from PostgreSQL
    2. Transform to Elasticsearch document
    3. Index document (update or insert)
    ↓
Elasticsearch updates inverted index
    ↓
Search results reflect change (within seconds)
```

**Query Flow: Customer Searches**

```
Customer searches "pizza" near (37.7749, -122.4194)
    ↓
Search Service receives request
    ↓
Check Redis cache: search:{hash(params)}
    ↓
Cache MISS → Query Elasticsearch:
    1. Analyzer tokenizes "pizza"
    2. Look up "pizza" in inverted index
    3. Filter by geo_distance
    4. Apply scoring function
    5. Sort by score
    ↓
Elasticsearch returns ranked results
    ↓
Cache results in Redis (5 min TTL)
    ↓
Return to customer
    ↓
Total latency: ~50ms (cache miss), ~10ms (cache hit)
```

**Failure Scenarios:**

**Q: How do we handle stale index vs DB truth?**

A: Multiple strategies:
1. **Eventual consistency**: Accept seconds of lag
2. **Critical checks**: Verify availability at order time
3. **Reindex on startup**: Full sync on service restart
4. **Monitoring**: Alert if Kafka lag > threshold

**Q: How do we reindex safely?**

A: Blue-green reindexing:
```java
public void reindex() {
    String newIndex = "restaurants_" + Instant.now().toEpochMilli();
    
    // 1. Create new index with mapping
    createIndex(newIndex);
    
    // 2. Bulk index all restaurants
    restaurantRepository.findAll().forEach(r -> 
        indexDocument(newIndex, r));
    
    // 3. Swap alias atomically
    swapAlias("restaurants", newIndex);
    
    // 4. Delete old index
    deleteOldIndex();
}
```

**Q: What happens if indexer lags?**

A: Degraded search results:
1. New restaurants don't appear
2. Closed restaurants still show
3. Old prices displayed
4. Alert on lag > 5 minutes
5. Critical data verified at order time

**Q: Failure modes and recovery?**

A: Elasticsearch cluster failures:
1. **Single node down**: Other nodes handle load
2. **Master down**: New master elected (< 30 seconds)
3. **All nodes down**: Fallback to PostgreSQL (degraded)
4. **Data corruption**: Reindex from PostgreSQL

### Search Service Implementation

```java
@Service
public class RestaurantSearchService {
    
    private final RestHighLevelClient esClient;
    private final RedisTemplate<String, String> redis;
    
    public SearchResponse search(SearchRequest request) {
        // Check cache first
        String cacheKey = "search:" + hashRequest(request);
        String cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return parseResponse(cached);
        }
        
        // Build Elasticsearch query
        SearchSourceBuilder sourceBuilder = buildSearchQuery(request);
        
        org.elasticsearch.action.search.SearchRequest esRequest = 
            new org.elasticsearch.action.search.SearchRequest("restaurants")
                .source(sourceBuilder);
        
        // Execute search
        org.elasticsearch.action.search.SearchResponse esResponse = 
            esClient.search(esRequest, RequestOptions.DEFAULT);
        
        // Transform results
        SearchResponse response = transformResults(esResponse);
        
        // Cache results
        redis.opsForValue().set(cacheKey, 
            serializeResponse(response), 
            Duration.ofMinutes(5));
        
        return response;
    }
    
    private String hashRequest(SearchRequest request) {
        return DigestUtils.md5Hex(
            request.getLat() + ":" + 
            request.getLng() + ":" + 
            request.getQuery() + ":" +
            request.getCuisine() + ":" +
            request.getPage()
        );
    }
}
```

---

## 4. Database Transactions Deep Dive

### A) CONCEPT: What are Database Transactions in Food Delivery Context?

Database transactions ensure ACID properties for critical operations like order placement, payment processing, and delivery assignment. In a food delivery system, transactions are essential for maintaining data consistency when multiple operations must succeed or fail together.

**What problems do transactions solve here?**

1. **Atomicity**: Order placement must create order, reserve inventory, and authorize payment atomically
2. **Consistency**: Menu items can't be oversold, orders can't be double-charged
3. **Isolation**: Concurrent order placements don't interfere with each other
4. **Durability**: Once committed, order data persists even if system crashes

**Transaction Isolation Levels:**

| Level | Description | Use Case |
|-------|-------------|----------|
| READ COMMITTED | Default, prevents dirty reads | Menu browsing, order history |
| REPEATABLE READ | Prevents non-repeatable reads | Order placement, inventory checks |
| SERIALIZABLE | Highest isolation, prevents phantom reads | Payment processing, critical updates |

### B) OUR USAGE: How We Use Transactions Here

**Critical Transactional Operations:**

1. **Order Placement Transaction**:
   - Create order record
   - Reserve menu item inventory
   - Authorize payment (not charge yet)
   - Create order items
   - All must succeed or all rollback

2. **Payment Processing Transaction**:
   - Charge payment
   - Update order status (CONFIRMED → PAID)
   - Update restaurant earnings
   - Create payment record
   - Update inventory (decrement)

3. **Order Cancellation Transaction**:
   - Update order status (CONFIRMED → CANCELLED)
   - Release inventory
   - Refund payment (if charged)
   - Update restaurant stats

**Transaction Configuration:**

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ,
    propagation = Propagation.REQUIRED,
    timeout = 30,
    rollbackFor = {Exception.class}
)
public Order createOrder(CreateOrderRequest request) {
    // Transactional operations
}
```

### C) REAL STEP-BY-STEP SIMULATION: Database Transaction Flow

**Normal Flow: Order Placement Transaction**

```
Step 1: Begin Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: BEGIN TRANSACTION                                │
│ Isolation Level: REPEATABLE READ                             │
│ Transaction ID: tx_order_789                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Restaurant Availability
┌─────────────────────────────────────────────────────────────┐
│ SELECT status, accepting_orders FROM restaurants             │
│ WHERE id = 'restaurant_123'                                  │
│ FOR UPDATE  -- Row-level lock                                │
│                                                              │
│ Result: status = 'OPEN', accepting_orders = true             │
│ Lock acquired on restaurant row                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Check Menu Item Availability
┌─────────────────────────────────────────────────────────────┐
│ SELECT id, name, price, available_quantity                   │
│ FROM menu_items                                              │
│ WHERE restaurant_id = 'restaurant_123'                      │
│   AND id IN ('item_1', 'item_2')                             │
│   AND available_quantity > 0                                │
│ FOR UPDATE  -- Lock menu items                               │
│                                                              │
│ Result: Both items available                                 │
│ Locks acquired on menu item rows                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Create Order Record
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO orders (id, restaurant_id, customer_id,         │
│                     status, total_amount, ...)               │
│ VALUES ('order_456', 'restaurant_123', 'customer_789',       │
│         'CONFIRMED', 45.50, ...)                             │
│                                                              │
│ Result: 1 row inserted                                       │
│ Order ID: order_456                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Create Order Items
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO order_items (order_id, menu_item_id, quantity,   │
│                          price, ...)                         │
│ VALUES                                                       │
│   ('order_456', 'item_1', 2, 15.00, ...),                    │
│   ('order_456', 'item_2', 1, 15.50, ...)                     │
│                                                              │
│ Result: 2 rows inserted                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Reserve Inventory
┌─────────────────────────────────────────────────────────────┐
│ UPDATE menu_items                                            │
│ SET available_quantity = available_quantity - 2             │
│ WHERE id = 'item_1'                                          │
│                                                              │
│ UPDATE menu_items                                            │
│ SET available_quantity = available_quantity - 1             │
│ WHERE id = 'item_2'                                          │
│                                                              │
│ Result: 2 rows updated                                       │
│ Inventory reserved (not decremented yet)                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 7: Authorize Payment
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO payments (id, order_id, amount, status, ...)     │
│ VALUES ('pay_789', 'order_456', 45.50, 'AUTHORIZED', ...)   │
│                                                              │
│ Result: 1 row inserted                                       │
│ Payment authorized (not charged yet)                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 8: Commit Transaction
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: COMMIT TRANSACTION                              │
│                                                              │
│ All changes made permanent:                                 │
│ - Order record created                                      │
│ - Order items created                                       │
│ - Inventory reserved                                        │
│ - Payment authorized                                        │
│ - Locks released                                            │
│                                                              │
│ Total time: ~80ms                                            │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Inventory Oversell Prevention**

```
Step 1: Concurrent Orders for Same Item
┌─────────────────────────────────────────────────────────────┐
│ Order A: Customer 1 orders 5 units of item_1                │
│ Order B: Customer 2 orders 3 units of item_1                │
│ Available quantity: 6 units                                 │
│ Both transactions start simultaneously                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Both Check Availability
┌─────────────────────────────────────────────────────────────┐
│ Order A: SELECT available_quantity FROM menu_items          │
│          WHERE id = 'item_1' FOR UPDATE                     │
│          Result: 6 (lock acquired)                         │
│                                                              │
│ Order B: SELECT available_quantity FROM menu_items          │
│          WHERE id = 'item_1' FOR UPDATE                     │
│          Waiting for lock (blocked)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Order A Updates Inventory
┌─────────────────────────────────────────────────────────────┐
│ Order A: UPDATE menu_items                                   │
│          SET available_quantity = 6 - 5 = 1                 │
│          WHERE id = 'item_1'                                 │
│                                                              │
│ Result: 1 row updated                                        │
│ Available quantity now: 1                                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Order A Commits
┌─────────────────────────────────────────────────────────────┐
│ Order A: COMMIT TRANSACTION                                  │
│ Lock released                                                │
│                                                              │
│ Order B: Lock acquired, sees available_quantity = 1         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Order B Checks Availability Again
┌─────────────────────────────────────────────────────────────┐
│ Order B: SELECT available_quantity FROM menu_items          │
│          WHERE id = 'item_1' FOR UPDATE                      │
│          Result: 1 (after Order A committed)                │
│                                                              │
│ Order B: Checks if 1 >= 3 (required quantity)               │
│          Result: FALSE                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 6: Order B Fails
┌─────────────────────────────────────────────────────────────┐
│ Order B: ROLLBACK TRANSACTION                                │
│ Error: "Insufficient inventory"                             │
│                                                              │
│ Application:                                                 │
│ - Return 409 Conflict to customer                           │
│ - Suggest alternative items                                 │
│ - Log inventory constraint violation                        │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Payment Authorization Failure**

```
Step 1: Order Created, Payment Authorization Attempted
┌─────────────────────────────────────────────────────────────┐
│ Transaction: Order placement in progress                     │
│ - Order record created ✅                                    │
│ - Inventory reserved ✅                                    │
│ - Payment authorization attempted ❌                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Payment Gateway Returns Error
┌─────────────────────────────────────────────────────────────┐
│ Payment Gateway API:                                          │
│ - Status: 402 Payment Required                               │
│ - Error: "Insufficient funds"                                │
│                                                              │
│ Exception thrown in application                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Automatic Rollback
┌─────────────────────────────────────────────────────────────┐
│ @Transactional detects exception:                           │
│ - Rolls back entire transaction                              │
│ - Order record deleted                                       │
│ - Inventory released                                         │
│ - All locks released                                         │
│                                                              │
│ PostgreSQL: ROLLBACK TRANSACTION                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Application Handling
┌─────────────────────────────────────────────────────────────┐
│ Application catches PaymentException:                        │
│                                                              │
│ 1. Return 402 Payment Required to customer                  │
│ 2. Log payment failure (for analytics)                      │
│ 3. No order created (atomicity maintained)                  │
│ 4. Customer can retry with different payment method         │
└─────────────────────────────────────────────────────────────┘
```

**Isolation Level Behavior: Concurrent Menu Updates**

```
Scenario: Restaurant updates menu while customer places order

Transaction A (Customer):              Transaction B (Restaurant):
─────────────────────────────────────────────────────────────────
BEGIN TRANSACTION                      BEGIN TRANSACTION
                                       
SELECT * FROM menu_items               UPDATE menu_items
WHERE restaurant_id = 'r123'           SET price = price * 1.1
FOR UPDATE                             WHERE restaurant_id = 'r123'
                                       
Result: item_1 = $10.00                Result: item_1 = $11.00
        item_2 = $15.00                        item_2 = $16.50
                                       
-- Customer sees old prices            COMMIT
-- (REPEATABLE READ isolation)         
                                       
INSERT INTO orders (...)               -- Prices updated
VALUES (..., 25.00, ...)               
                                       
COMMIT                                 
                                       
✅ Order created with old prices       ✅ Menu updated
                                       
                                       
With REPEATABLE READ:                   With READ COMMITTED:
- Customer sees consistent snapshot    - Customer sees latest prices
- Order uses prices from transaction   - Order uses new prices
  start (even if menu updated)          (may cause confusion)
- Prevents price changes mid-order     - May cause pricing issues
```

**Lock Contention: Peak Hour Order Placement**

```
Step 1: Dinner Rush - 50 Orders/Second
┌─────────────────────────────────────────────────────────────┐
│ 50 concurrent transactions trying to place orders            │
│ Many targeting same popular restaurants                     │
│                                                              │
│ Lock queue forms on restaurant and menu item rows          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Lock Acquisition Strategy
┌─────────────────────────────────────────────────────────────┐
│ Application: Acquire locks in consistent order              │
│ 1. Lock restaurant row (by restaurant_id)                  │
│ 2. Lock menu items (by item_id, sorted)                     │
│                                                              │
│ Prevents deadlocks by consistent lock ordering             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Lock Wait Timeout
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: lock_timeout = 5 seconds                        │
│                                                              │
│ If transaction waits > 5 seconds for lock:                 │
│ - Transaction aborted                                        │
│ - Error: "canceling statement due to lock timeout"         │
│ - Application retries with exponential backoff             │
└─────────────────────────────────────────────────────────────┘
```

**Retry Logic for Failed Transactions**

```java
@Retryable(
    value = {DeadlockLoserDataAccessException.class, 
             PessimisticLockingFailureException.class,
             TransactionTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 200, multiplier = 2)
)
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Order createOrder(CreateOrderRequest request) {
    // Transaction logic
    // Retries automatically on deadlock/lock timeout
}
```

**Recovery Behavior:**

```
Transaction Failure → Automatic Retry:
1. Deadlock detected → Retry with 200ms delay
2. Lock timeout → Retry with 400ms delay
3. Payment failure → No retry (business logic error)
4. Inventory insufficient → No retry (business logic error)
5. Max retries exceeded → Return error to client

Monitoring:
- Track retry rate (alert if > 3%)
- Track deadlock rate (alert if > 0.5%)
- Track lock timeout rate (alert if > 1%)
- Track payment failure rate (alert if > 5%)
```

---

## Summary

| Component | Technology | Key Configuration |
|-----------|------------|-------------------|
| Event Streaming | Kafka | 24 partitions for orders, RF=3 |
| Caching | Redis Cluster | 4GB, volatile-lru |
| Search | Elasticsearch | 6 nodes, geo + text |
| Sync Pattern | Outbox + CDC | Transactional consistency |

### Key Design Decisions

1. **Outbox pattern**: Ensures order events are never lost
2. **Cache-aside for menus**: Balance freshness and performance
3. **Write-through for orders**: Real-time state consistency
4. **Elasticsearch for search**: Full-text + geo queries
5. **Event-driven indexing**: Eventual consistency acceptable

